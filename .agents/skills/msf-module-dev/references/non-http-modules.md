# Scanner, protocol, local, and post module patterns

Inspect the current protocol/post mixin and its specs before writing connection, reporting, or file logic. Mature mixins often own options, connection lifecycle, and service reporting.

## Scanner modules

Use `Msf::Auxiliary::Scanner` and implement `run_host(ip)` rather than `run`:

```ruby
# frozen_string_literal: true

class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::Tcp
  include Msf::Auxiliary::Scanner
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Acme Protocol Version Scanner',
        'Description' => %q{
          This module identifies Acme Protocol services and reports their version.
        },
        'Author' => ['Contributor'],
        'License' => MSF_LICENSE,
        'References' => [],
        'DefaultOptions' => {
          'RPORT' => 31337
        },
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => [IOC_IN_LOGS]
        }
      )
    )
  end

  def run_host(ip)
    connect
    sock.put("VERSION\r\n")
    banner = sock.get_once(1024, 5).to_s
    return unless banner.start_with?('ACME/')

    version = banner.split('/', 2).last.to_s.strip
    print_good("#{Rex::Socket.to_authority(ip, rport)} - Acme Protocol #{version}")
    report_service(
      host: ip,
      port: rport,
      proto: 'tcp',
      name: 'acme',
      info: "Acme Protocol #{version}"
    )
  rescue Rex::ConnectionError => e
    vprint_error("#{Rex::Socket.to_authority(ip, rport)} - Connection failed: #{e.message}")
  ensure
    disconnect
  end
end
```

- `RHOSTS` and `THREADS` come from Scanner; do not re-register them.
- Keep per-host state local to `run_host`; scanner instances may be used concurrently.
- A nil/empty banner is not a product match.
- Use `Rex::Socket.to_authority` for explicit host/port output.
- Verify whether the protocol mixin already reports host/service before adding a duplicate call.

## Raw TCP modules

Use `connect`, `sock.put`, `sock.get_once`, and `disconnect`. Distinguish transport failure from an unexpected protocol response:

```ruby
def check
  connect
  sock.put(probe)
  response = sock.get_once(4096, 5)
  return Exploit::CheckCode::Unknown('The service closed the connection without a response') if response.blank?
  return Exploit::CheckCode::Safe('The response did not match the Acme Protocol fingerprint') unless acme_response?(response)

  version = extract_version(response)
  return Exploit::CheckCode::Detected('Acme Protocol was detected without a version') unless version
  return Exploit::CheckCode::Safe("Acme Protocol #{version} is not affected") unless vulnerable_version?(version)

  Exploit::CheckCode::Appears("Acme Protocol #{version} is affected")
rescue Rex::ConnectionError => e
  Exploit::CheckCode::Unknown("Connection failed: #{e.message}")
ensure
  disconnect
end
```

Use `fail_with(Failure::Unreachable, ...)` in `run`/`exploit`, not in `check`. Avoid broad rescues that turn programmer/parser errors into a misleading connection failure.

If a payload format stores an IPv4 address in a raw AF_INET field, explicitly reject IPv6 addresses and hostnames and spec those failures. Do not silently truncate or resolve differently from framework expectations.

## LoginScanner contracts

When adding or changing `Metasploit::Framework::LoginScanner` code, treat it as library code:

- add focused/shared RSpec coverage;
- document public APIs with YARD;
- reuse one connection for banner negotiation and authentication when the protocol permits it;
- rescue specific network/protocol exceptions, not `StandardError`;
- return `UNABLE_TO_CONNECT` for nil replies, timeout, reset, or transport failure;
- return the incorrect-credential status only after an explicit authentication rejection;
- return success only after protocol-specific proof of authentication;
- keep proof/status metadata consistent with existing LoginScanner result objects.

Test at least: success, explicit rejection, nil read, timeout, reset, malformed banner, and connection count. Do not report every successful login as a vulnerability. Let the scanner/protocol layer own host, service, and credential records where its API already does so; avoid duplicate module-level notes and reports.

Authentication brute force normally implies both `IOC_IN_LOGS` and `ACCOUNT_LOCKOUTS` in module Notes.

## Protocol mixins

- FTP: use the current FTP mixin (`connect_login`, `send_cmd`, and its reporting behavior).
- SMB: use the current RubySMB-backed client mixins; do not implement SMB framing manually.
- SSH: use framework SSH/login-scanner support and its credential result contract.
- LDAP, DCERPC, SMB, and modern HTTP mixins may already create layered services/resources. Inspect their `report_*_service` helpers before adding another.

## Session ownership and module type

If the operation fundamentally requires an existing session, it normally belongs under `modules/post` (or `Msf::Exploit::Local` for privilege escalation), not an auxiliary module with a hand-registered `SESSION` option and custom session plumbing.

Use:

- `Msf::Exploit::Local` when the outcome is local exploitation/privilege escalation and a payload/session is produced;
- `Msf::Post` for gathering, administration, persistence, and other work through an existing session.

Declare the correct `Platform` and `SessionTypes`. Use architecture constants and session APIs, not regexes over human-readable architecture strings.

## Local exploit checks

Missing observability is not proof of safety:

```ruby
def check
  return Exploit::CheckCode::Unknown('The session cannot determine the current privilege level') unless current_user
  return Exploit::CheckCode::Safe("The required file #{target_path} does not exist") unless file_exist?(target_path)
  return Exploit::CheckCode::Unknown('The session lacks permission to inspect the vulnerable file') unless readable?(target_path)

  version = vulnerable_component_version
  return Exploit::CheckCode::Unknown('The component version could not be determined') unless version
  return Exploit::CheckCode::Safe("Component version #{version} is not affected") unless vulnerable_version?(version)

  Exploit::CheckCode::Appears("Component version #{version} is affected")
end
```

Use `Unknown`, not `Safe`, when missing privileges, permissions, a session capability, or response data prevents assessment. `Safe` needs affirmative evidence that the exploit path is not applicable.

Do not add a precondition that accidentally rejects the low-privileged user the exploit is designed to elevate.

## Remote file operations

Include `Msf::Post::File` and check every mutation result:

```ruby
fail_with(Failure::NotFound, "Target file #{target_path} does not exist") unless file_exist?(target_path)
fail_with(Failure::NoAccess, "Target file #{target_path} is not writable") unless writable?(target_path)

original = read_file(target_path)
fail_with(Failure::UnexpectedReply, "Could not read #{target_path}") if original.nil?

# A failed write may still be partial. Arm restoration before attempting it.
@file_to_restore = { path: target_path, data: original }

unless write_file(target_path, replacement)
  fail_with(Failure::UnexpectedReply, "Failed to write #{target_path}")
end

written = read_file(target_path)
fail_with(Failure::UnexpectedReply, "Could not verify #{target_path}") unless written == replacement
```

Rules:

- Check `write_file`, `append_file`, `upload_file`, and equivalent return values.
- Verify the resulting file/state where practical; command exit status alone may lie.
- Preserve original contents/permissions/configuration and restore that captured state, not an assumed default.
- Register only artifacts created by the module. Use `register_file_for_cleanup` for files and `register_dir_for_cleanup` for directories.
- If editing a pre-existing file, FileDropper deletion is usually wrong; capture the original state and implement explicit restoration.
- Arm restoration before calling a write API because a false/exceptional return may follow a partial write; cleanup must consume that state on both failure and success.
- Persistence mixins may invoke installation hooks automatically. Inspect their lifecycle before calling an install method yourself.

Test write failure, partial write, verification failure, cleanup failure, and repeated execution.

## Process execution

Prefer the structured API:

```ruby
output = create_process(
  '/usr/bin/id',
  args: ['-u'],
  time_out: 15,
  opts: { 'Subshell' => false }
)
```

Use `create_process(executable, args: [], time_out: 15, opts: {})` instead of deprecated `cmd_exec` forms that pass command and arguments separately. A single-string `cmd_exec(command)` may still be appropriate where a shell command is genuinely required; avoid unsafe concatenation of untrusted values.

## Post module pattern

```ruby
# frozen_string_literal: true

class MetasploitModule < Msf::Post
  include Msf::Post::File

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Acme App Configuration Gather',
        'Description' => %q{
          This module gathers the Acme App configuration from an existing session.
        },
        'License' => MSF_LICENSE,
        'Author' => ['Contributor'],
        'Platform' => %w[linux unix],
        'SessionTypes' => %w[meterpreter shell],
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )

    register_options(
      [
        OptString.new('CONFIG_PATH', [true, 'Path to the Acme App configuration', '/etc/acme/config.yml'])
      ]
    )
  end

  def run
    path = datastore['CONFIG_PATH']
    fail_with(Failure::NotFound, "Configuration file #{path} does not exist") unless file_exist?(path)
    fail_with(Failure::NoAccess, "Configuration file #{path} is not readable") unless readable?(path)

    data = read_file(path)
    fail_with(Failure::UnexpectedReply, "Could not read #{path}") if data.nil?

    loot_path = store_loot(
      'acme.config',
      'text/plain',
      session,
      data,
      'acme-config.yml',
      'Acme App configuration'
    )
    print_good("Configuration saved to #{loot_path}")
  end
end
```

Do not assume a database record exists. Guard `session.db_record` access (`session.db_record&.id`) and test with the database disconnected.

## Generated fetch commands

When framework/library code generates download-and-execute shell:

- do not trust a downloader's exit status alone; verify that the expected artifact exists and is usable;
- gate fallback so a false-success downloader does not suppress the next method;
- test the exact generated command under the shells claimed (for example bash, dash, and BusyBox ash);
- use fake downloaders to cover false success, missing/empty file, invalid executable, network failure, and fallback;
- avoid subshell/wrapper behavior that hides the status needed by the fallback logic.

This is usually library work and therefore requires focused specs, not only a manual module run.

## Non-HTTP QA

- Connection refusal, timeout, reset, nil/empty read, malformed response, and explicit rejection are distinct paths.
- Every `check` path has a reason and missing observability maps to `Unknown`.
- Every target file is checked before opening; every mutation and restoration is verified.
- Protocol-layer reporting is not duplicated by the module.
- Session/database assumptions are tested with no DB record.
- Target platform and architecture use constants and are exercised on every claimed environment.
- Cleanup restores original state and an immediate rerun succeeds.

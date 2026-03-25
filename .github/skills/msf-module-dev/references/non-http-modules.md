# Non-HTTP Module Patterns

Reference for writing MSF modules that don't use the HttpClient mixin: scanners, TCP/UDP services, local exploits, and post-exploitation modules.

---

## Scanner Modules (`Msf::Auxiliary::Scanner`)

The Scanner mixin handles RHOSTS iteration, threading, and progress display. You implement `run_host(ip)` instead of `run`.

### Key rules

- `include Msf::Auxiliary::Scanner` **after** protocol mixins (order matters)
- Implement `def run_host(ip)` — called once per host in RHOSTS
- Do NOT define `def run` — the Scanner mixin provides it
- `RHOSTS` and `THREADS` are auto-registered
- Use `peer` for log prefixing (returns `"host:port"`)

### Scanner template

```ruby
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient  # or Tcp, Ftp, SMB, etc.
  include Msf::Auxiliary::Scanner            # MUST come after protocol mixins
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Product Version Scanner',
        'Description' => %q{
          Scans for Vendor Product instances and fingerprints the version.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )

    register_options([
      Opt::RPORT(8080)
    ])
  end

  def run_host(ip)
    res = send_request_cgi('uri' => normalize_uri(target_uri.path))
    return unless res

    if res.body =~ /Version: (\d+\.\d+\.\d+)/
      version = Regexp.last_match(1)
      print_good("#{peer} - Detected version #{version}")
      report_service(host: ip, port: rport, proto: 'tcp', name: 'http', info: "Product #{version}")
    else
      vprint_status("#{peer} - Service detected but version unknown")
    end
  end
end
```

### Batch scanner (alternative)

For protocols where per-host connections are expensive, implement `run_batch` + `run_batch_size` instead of `run_host`:

```ruby
def run_batch_size
  50
end

def run_batch(hosts)
  # hosts is an array of IP strings
  hosts.each do |ip|
    # process batch
  end
end
```

---

## TCP-Based Modules (`Msf::Exploit::Remote::Tcp`)

For raw TCP services without a dedicated protocol mixin.

### Key rules

- `include Msf::Exploit::Remote::Tcp` — auto-registers `RHOSTS`, `RPORT`, `SSL`, `ConnectTimeout`
- Use `connect` to open a socket, `disconnect` to close
- Use `sock.put(data)` to send, `sock.get_once(length, timeout)` to receive
- Always handle connection failures

### TCP module pattern

```ruby
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::Tcp
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Service Banner Grab',
        'Description' => %q{
          Connects to the target service and grabs the banner.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )

    register_options([
      Opt::RPORT(9090)
    ])
  end

  def run
    connect
    banner = sock.get_once(1024, 10)
    disconnect

    if banner && !banner.empty?
      print_good("Banner: #{banner.strip}")
      report_service(host: rhost, port: rport, proto: 'tcp', name: 'vendor_svc', info: banner.strip)
    else
      print_error('No banner received')
    end
  rescue Rex::ConnectionError => e
    fail_with(Failure::Unreachable, e.message)
  end
end
```

---

## FTP Modules (`Msf::Exploit::Remote::Ftp`)

### Key mixins and methods

```ruby
include Msf::Exploit::Remote::Ftp
```

Auto-registers: `RPORT` (21), `FTPUSER`, `FTPPASS`, `FTPTimeout`, `FTPDEBUG`.

| Method                                  | Description                                 |
| --------------------------------------- | ------------------------------------------- |
| `connect_login`                         | Connect + authenticate (returns true/false) |
| `connect`                               | Open FTP control connection                 |
| `send_cmd(args, recv = true)`           | Send an FTP command                         |
| `send_cmd_data(args, data, mode = 'a')` | Send data via a data channel                |
| `data_connect(mode = 'a')`              | Open a data channel                         |
| `disconnect`                            | Close connections                           |

---

## SMB Modules (`Msf::Exploit::Remote::SMB`)

```ruby
include Msf::Exploit::Remote::DCERPC
include Msf::Exploit::Remote::SMB::Client
```

Auto-registers: `RPORT` (445), `SMBUser`, `SMBPass`, `SMBDomain`, `SMB::ProtocolVersion`.

Common methods: `connect`, `smb_login`, `smb_create(filename)`, `smb_write`.

---

## SSH Modules

SSH modules commonly use `Net::SSH` via the framework's SSH mixin:

```ruby
include Msf::Exploit::Remote::SSH
```

For scanner-style SSH modules, look at existing modules in `modules/auxiliary/scanner/ssh/`.

---

## Local Exploit Modules (`Msf::Exploit::Local`)

For privilege escalation and post-exploitation exploits that require an existing session.

### Key rules

- Inherit from `Msf::Exploit::Local` (not `Msf::Exploit::Remote`)
- **MUST** specify `'SessionTypes'` in metadata (e.g., `['shell', 'meterpreter']`)
- Use `prepend Msf::Exploit::Remote::AutoCheck` — always prepend, never include
- Common mixins: `Msf::Post::File`, `Msf::Post::Linux::Priv`, `Msf::Post::Linux::Kernel`, `Msf::Exploit::EXE`, `Msf::Exploit::FileDropper`
- Use `session` object for all target interaction
- Use `cmd_exec(command)` to run commands on target
- Use `write_file(path, data)` / `read_file(path)` from `Msf::Post::File`

### Local exploit template

```ruby
class MetasploitModule < Msf::Exploit::Local
  Rank = ExcellentRanking

  include Msf::Post::File
  include Msf::Post::Linux::Priv
  include Msf::Exploit::EXE
  include Msf::Exploit::FileDropper

  prepend Msf::Exploit::Remote::AutoCheck

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Product Local Privilege Escalation',
        'Description' => %q{
          Description of the local vulnerability and how it achieves
          privilege escalation.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'References' => [
          ['CVE', 'YYYY-NNNNN']
        ],
        'DisclosureDate' => '2026-01-15',
        'Platform' => ['unix', 'linux'],
        'SessionTypes' => ['shell', 'meterpreter'],
        'Arch' => [ARCH_X64, ARCH_CMD],
        'Targets' => [
          ['Automatic', {}]
        ],
        'DefaultTarget' => 0,
        'DefaultOptions' => {
          'PAYLOAD' => 'linux/x64/meterpreter/reverse_tcp'
        },
        'Privileged' => true,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [REPEATABLE_SESSION],
          'SideEffects' => [ARTIFACTS_ON_DISK]
        }
      )
    )
  end

  def check
    output = cmd_exec('cat /etc/product_version')
    if output =~ /(\d+\.\d+)/
      version = Rex::Version.new(Regexp.last_match(1))
      if version < Rex::Version.new('2.0')
        return CheckCode::Appears("Version #{version} is vulnerable")
      end
    end
    CheckCode::Safe
  end

  def exploit
    unless is_root?
      fail_with(Failure::NoAccess, 'Must run as root or with sudo')
    end

    payload_path = '/tmp/.payload'
    write_file(payload_path, generate_payload_exe)
    register_file_for_cleanup(payload_path)
    cmd_exec("chmod +x #{payload_path}")
    cmd_exec(payload_path)
  end
end
```

### AutoCheck (`prepend Msf::Exploit::Remote::AutoCheck`)

- **Always use `prepend`**, never `include` — it wraps `exploit`/`run` to call `check` first
- Auto-registers `AutoCheck` (default: true) and `ForceExploit` (default: false) advanced options
- If `check` returns `Safe`, the exploit aborts unless `ForceExploit` is set
- If `check` returns `Appears` or `Vulnerable`, exploitation proceeds

### FileDropper (`Msf::Exploit::FileDropper`)

Tracks files/dirs created on target and cleans them up after exploitation:

```ruby
include Msf::Exploit::FileDropper

# In exploit method:
register_file_for_cleanup('/tmp/payload.bin')
register_dir_for_cleanup('/tmp/exploit_dir')
```

---

## Post-Exploitation Modules (`Msf::Post`)

For modules that run after initial access via an existing session.

### Key rules

- Inherit from `Msf::Post`
- **MUST** specify `'Platform'` and `'SessionTypes'`
- `'SessionTypes'` can be: `'shell'`, `'meterpreter'`, `'powershell'`
- Use `session` to interact: `session.type`, `session.platform`, `session.session_host`
- Common mixins: `Msf::Post::File`, `Msf::Post::Windows::Registry`, `Msf::Post::Linux::Priv`
- Use `sysinfo` for system information (meterpreter sessions)
- Use `cmd_exec(command)` to run system commands

### Post module template

```ruby
class MetasploitModule < Msf::Post
  include Msf::Post::File
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Multi Gather Application Credentials',
        'Description' => %q{
          Extracts saved credentials from Application X config files.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Platform' => %w[linux win],
        'SessionTypes' => %w[shell meterpreter],
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )
  end

  def run
    hostname = sysinfo.nil? ? cmd_exec('hostname') : sysinfo['Computer']
    print_status("Running on #{hostname} (#{session.session_host})")

    config = read_file('/etc/app/config.yml')
    if config.nil? || config.empty?
      print_error('Config file not found')
      return
    end

    print_good('Found config file, extracting credentials...')
    path = store_loot(
      'app.config',
      'text/plain',
      session,
      config,
      'config.yml',
      'Application config file'
    )
    print_good("Config saved to: #{path}")
  end
end
```

### Post module mixins reference

| Mixin                          | Purpose                                                             |
| ------------------------------ | ------------------------------------------------------------------- |
| `Msf::Post::File`              | `read_file`, `write_file`, `file_exist?`, `directory?`, `readable?` |
| `Msf::Post::Linux::Priv`       | `is_root?`, `id`, `whoami`                                          |
| `Msf::Post::Linux::Kernel`     | `uname`, `kernel_release`, `kernel_modules`                         |
| `Msf::Post::Windows::Registry` | `registry_getvaldata`, `registry_enumkeys`, etc.                    |
| `Msf::Post::Windows::Priv`     | `is_admin?`, `is_system?`, `getsystem`                              |
| `Msf::Post::Common`            | `cmd_exec(command)`, `get_env(name)`                                |
| `Msf::Post::Unix`              | `get_users`, `get_groups`, `enum_user_directories`                  |

---

## CheckCode Constants

Used in `def check` for exploit and scanner modules:

| Constant                 | Code            | When to return                                          |
| ------------------------ | --------------- | ------------------------------------------------------- |
| `CheckCode::Unknown`     | `'unknown'`     | Cannot determine vulnerability status (timeout, error)  |
| `CheckCode::Safe`        | `'safe'`        | Target is NOT vulnerable (patched, not running service) |
| `CheckCode::Detected`    | `'detected'`    | Service is running but vuln status unknown              |
| `CheckCode::Appears`     | `'appears'`     | Likely vulnerable (version/banner-based)                |
| `CheckCode::Vulnerable`  | `'vulnerable'`  | Confirmed vulnerable (active verification)              |
| `CheckCode::Unsupported` | `'unsupported'` | Module does not support check                           |

CheckCode accepts an optional reason string:

```ruby
CheckCode::Appears("Version #{version} is in the vulnerable range")
CheckCode::Safe('Target is running patched version 2.1.0')
```

---

## Multipart File Upload (`vars_form_data`)

For HTTP modules that upload files via multipart/form-data:

```ruby
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'upload'),
  'vars_form_data' => [
    {
      'name' => 'file',
      'filename' => 'shell.php',
      'content_type' => 'application/octet-stream',
      'data' => payload_content,
      'encoding' => 'binary'
    },
    {
      'name' => 'description',
      'data' => 'Uploaded file'
    }
  ]
)
```

### `vars_form_data` field keys

| Key              | Required | Description                               |
| ---------------- | -------- | ----------------------------------------- |
| `'name'`         | Yes      | Form field name                           |
| `'data'`         | Yes      | Field value or file content               |
| `'filename'`     | No       | Filename (presence makes it a file field) |
| `'content_type'` | No       | MIME type for file fields                 |
| `'encoding'`     | No       | `'binary'` for binary content             |

---

## Verbose Output

Use `vprint_*` methods for output only shown when `VERBOSE` is true:

```ruby
vprint_status('Attempting connection...')    # Informational
vprint_good('Token obtained')               # Success
vprint_warning('Retrying request...')       # Warning
vprint_error('Unexpected response code')    # Error (non-fatal)
```

These are identical to `print_*` but suppressed unless `datastore['VERBOSE']` is true. Use them for detailed debugging info that would be noisy in normal operation.

---

## Common Protocol Mixins Quick Reference

| Protocol   | Mixin                               | Default Port | Key Methods                                          |
| ---------- | ----------------------------------- | ------------ | ---------------------------------------------------- |
| HTTP       | `Msf::Exploit::Remote::HttpClient`  | 80/443       | `send_request_cgi`, `normalize_uri`                  |
| TCP (raw)  | `Msf::Exploit::Remote::Tcp`         | -            | `connect`, `disconnect`, `sock.put`, `sock.get_once` |
| UDP        | `Msf::Exploit::Remote::Udp`         | -            | `connect_udp`, `disconnect_udp`, `udp_sock`          |
| FTP        | `Msf::Exploit::Remote::Ftp`         | 21           | `connect_login`, `send_cmd`, `data_connect`          |
| SMB        | `Msf::Exploit::Remote::SMB::Client` | 445          | `connect`, `smb_login`, `smb_create`                 |
| SSH        | `Msf::Exploit::Remote::SSH`         | 22           | Framework SSH wrapper                                |
| SMTP       | `Msf::Exploit::Remote::Smtp`        | 25           | `connect`, `raw_send_recv`                           |
| MySQL      | `Msf::Exploit::Remote::MYSQL`       | 3306         | `mysql_login`, `mysql_query`                         |
| PostgreSQL | `Msf::Exploit::Remote::Postgres`    | 5432         | `postgres_login`, `postgres_query`                   |
| MSSQL      | `Msf::Exploit::Remote::MSSQL`       | 1433         | `mssql_login`, `mssql_query`                         |
| SNMP       | `Msf::Exploit::Remote::SNMPClient`  | 161          | SNMP get/set/walk                                    |
| LDAP       | `Msf::Exploit::Remote::LDAP`        | 389/636      | LDAP bind/search                                     |
| Telnet     | `Msf::Exploit::Remote::Telnet`      | 23           | `connect`, `negotiate`                               |
| WinRM      | `Msf::Exploit::Remote::WinRM`       | 5985/5986    | `winrm_run_cmd`                                      |

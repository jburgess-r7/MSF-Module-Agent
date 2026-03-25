---
name: msf-module-dev
description: "Write, review, or fix Metasploit Framework modules following Rapid7 standards. Use when: creating MSF module, auxiliary module, exploit module, metasploit development, msftidy, rubocop, MSF contribution, rapid7 PR, module metadata, HttpClient mixin, store_loot, report_cred, SOAP XML module, HTTP exploit."
---

# Metasploit Framework Module Development

Complete reference for writing MSF modules that pass `msftidy`, RuboCop, and Rapid7 code review.

## When to Use

- Writing a new auxiliary or exploit module
- Converting a PoC script into an MSF module
- Reviewing or fixing an existing module for PR submission
- Debugging msftidy or rubocop failures

## Module Type Classification

| If the module...                                                               | Type                   | Path                                             |
| ------------------------------------------------------------------------------ | ---------------------- | ------------------------------------------------ |
| Gathers data via an existing vulnerability (email dump, cred theft, file read) | `Msf::Auxiliary`       | `auxiliary/gather/`                              |
| Performs admin actions on a service (config change, user creation)             | `Msf::Auxiliary`       | `auxiliary/admin/http/`                          |
| Scans for the presence of a vulnerability                                      | `Msf::Auxiliary`       | `auxiliary/scanner/http/`                        |
| Achieves code execution (shell, meterpreter)                                   | `Msf::Exploit::Remote` | `exploits/multi/http/` or `exploits/linux/http/` |
| Runs after initial access (privesc, lateral movement)                          | `Msf::Post`            | `post/multi/gather/`                             |

## Required File Structure

```
modules/<type>/path/module_name.rb           # The module
documentation/modules/<type>/path/module_name.md  # Documentation
```

Module filenames: lowercase, underscores, no hyphens. e.g. `webapp_xss_svg_stored.rb`

## Module Template

Every module MUST follow this exact structure. See [references/module-template.md](./references/module-template.md) for a complete annotated template.

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

class MetasploitModule < Msf::Auxiliary  # or Msf::Exploit::Remote
  include Msf::Exploit::Remote::HttpClient
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => '',
        'Description' => %q{
        },
        'Author' => [],
        'License' => MSF_LICENSE,
        'References' => [],
        'DisclosureDate' => 'YYYY-MM-DD',
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => [IOC_IN_LOGS]
        }
      )
    )
    register_options([])
  end

  def run
  end
end
```

## Critical Rules

### Metadata

1. **Name**: Short, descriptive, title case. No special chars (`&<>=`). Include vendor and vuln type.
2. **Description**: Use `%q{...}` multi-line format. Indent content to align with the `Description` key. Explain what the module does, the vulnerability, affected versions, and prerequisites.
3. **Author**: Array of `'Name'` or `'Name <email>'`. No Twitter handles. Balanced angle brackets.
4. **References**: Array of `['TYPE', 'VALUE']` pairs. See [references/metadata.md](./references/metadata.md).
5. **DisclosureDate**: ISO 8601 format `'YYYY-MM-DD'`. Required for exploits. Recommended for auxiliary.
6. **Notes**: MUST contain `Stability`, `Reliability`, and `SideEffects` keys. See [references/metadata.md](./references/metadata.md).
7. **License**: Always `MSF_LICENSE` unless the code has a specific BSD/MIT license.

### Code Style

1. **2-space indentation**, no tabs, no trailing whitespace
2. **Single quotes** unless string interpolation is needed
3. **No `require`** for MSF libs — they autoload
4. **No `print`/`puts`** — use `print_status`, `print_good`, `print_error`, `print_warning`
5. **No `rescue Exception`** — rescue specific errors or `StandardError`
6. Hash values in `update_info` must start on the **same line** as their key
7. The `update_info(` call must start on its **own line** after `super(`
8. Multi-line `OptEnum`/`register_options` arrays: first element on a **new line** after `[`
    ```ruby
    # GOOD
    OptEnum.new('MODE', [
      true, 'Operation mode', 'check', ['check', 'exploit']
    ])
    # BAD (triggers Layout/FirstArrayElementLineBreak)
    OptEnum.new('MODE', [true, 'Operation mode', 'check',
                         ['check', 'exploit']])
    ```
9. Run `msftidy` and `rubocop` before submitting (see Validation section below)

### HTTP Modules

1. Always `include Msf::Exploit::Remote::HttpClient`
2. Use `send_request_cgi({...})` — NEVER raw HTTP libraries
3. Always check for `nil` response (timeout): `fail_with(Failure::Unreachable, '...') unless res`
4. Use `normalize_uri(target_uri.path, 'endpoint')` for URL paths
5. Register `TARGETURI` if the app may not be at the web root:
    ```ruby
    OptString.new('TARGETURI', [true, 'Base path', '/'])
    ```
6. Set sensible `DefaultOptions` for `RPORT` and `SSL`
7. Use `res.get_json_document` to parse JSON responses
8. Use `Rex::Text::Table` for formatted output

### Credential and Data Reporting

Use `include Msf::Auxiliary::Report` and call:

- **`store_loot(type, ctype, host, data, filename, description)`** — for exfiltrated files/data
- **`create_credential` + `create_credential_login`** — for discovered credentials
- **`report_service(host:, port:, proto:, name:)`** — to record identified services

See [references/reporting.md](./references/reporting.md) for the full credential reporting pattern.

### Error Handling

Use `fail_with(Failure::REASON, 'message')` — never `raise` or `abort`.

| Constant                   | When to use                               |
| -------------------------- | ----------------------------------------- |
| `Failure::NoAccess`        | Authentication failed                     |
| `Failure::UnexpectedReply` | Server responded but not as expected      |
| `Failure::NotFound`        | Endpoint or resource missing              |
| `Failure::NotVulnerable`   | Target is patched / not vulnerable        |
| `Failure::Unreachable`     | Connection refused, timeout, nil response |
| `Failure::BadConfig`       | Invalid module options                    |
| `Failure::TimeoutExpired`  | Operation timed out                       |

## Validation

### Running msftidy

`msftidy` extracts the module type from the file path — it looks for `/modules/<type>/` in the absolute path. If the file is outside a `modules/` directory tree, it will give a false "Unexpected super class" warning.

```bash
# CORRECT — file is under a modules/ tree:
mkdir -p /tmp/msf_test/modules/auxiliary/admin/http
cp my_module.rb /tmp/msf_test/modules/auxiliary/admin/http/
cd /opt/metasploit-framework/embedded/framework
ruby -e '
  require_relative "tools/dev/msftidy"
  f = MsftidyRunner.new("/tmp/msf_test/modules/auxiliary/admin/http/my_module.rb")
  f.run_checks
  puts "msftidy status: #{f.status}"
'
# Status 0 = clean, 1 = warnings, 2 = errors
```

### Running rubocop

**CRITICAL**: Always use MSF's `.rubocop.yml` config. The default rubocop config has much stricter limits that don't apply to MSF modules (e.g. `Metrics/MethodLength` defaults to 10 lines vs MSF's 300, `FrozenStringLiteralComment` is disabled, `Style/Documentation` is excluded for `modules/**/*`, etc.).

```bash
cd /opt/metasploit-framework/embedded/framework
rubocop --config .rubocop.yml /path/to/my_module.rb
```

If the file is outside the framework `modules/` directory, `Style/Documentation` may still flag — this is a false positive that disappears when the file is in its final location.

### Testing in msfconsole

The easiest way to load custom modules:

```bash
# Option 1: Copy to ~/.msf4/modules/ (auto-loaded on startup)
mkdir -p ~/.msf4/modules/auxiliary/admin/http
cp my_module.rb ~/.msf4/modules/auxiliary/admin/http/
msfconsole -q
# Module is immediately available:
#   use auxiliary/admin/http/my_module

# Option 2: Use loadpath (point to parent dir containing modules/ subfolder)
# The path must contain a modules/ subdirectory with the proper tree
msfconsole -q -x "loadpath /path/to/project"
# Where /path/to/project/modules/auxiliary/admin/http/my_module.rb exists
```

After loading, test `check` first, then `run`:

```
msf6 > use auxiliary/admin/http/my_module
msf6 auxiliary(admin/http/my_module) > set RHOSTS target
msf6 auxiliary(admin/http/my_module) > set VERBOSE true
msf6 auxiliary(admin/http/my_module) > check
msf6 auxiliary(admin/http/my_module) > run
```

### Checklist

Before submitting, verify:

- [ ] `ruby -c module.rb` — no syntax errors
- [ ] `msftidy` — status 0, no warnings (file must be under a `modules/` path tree)
- [ ] `rubocop --config .rubocop.yml` — no offenses (run from MSF framework dir)
- [ ] `Notes` hash has `Stability`, `Reliability`, `SideEffects`
- [ ] `DisclosureDate` is `YYYY-MM-DD` format
- [ ] `References` use correct type identifiers (`CVE`, `EDB`, `URL`, `GHSA`)
- [ ] No hardcoded IPs, domains, or credentials
- [ ] All `send_request_cgi` calls check for `nil` response
- [ ] Module documentation `.md` file exists with Verification Steps and Scenarios
- [ ] Tested `check` and `run` against a real instance via `~/.msf4/modules/` or `loadpath`

## Reference Files

| Topic                                                     | File                                                                           |
| --------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Complete annotated module template                        | [references/module-template.md](./references/module-template.md)               |
| Metadata reference (Notes, References, Rankings, Options) | [references/metadata.md](./references/metadata.md)                             |
| Credential and loot reporting patterns                    | [references/reporting.md](./references/reporting.md)                           |
| Module documentation template                             | [references/documentation-template.md](./references/documentation-template.md) |
| HttpClient mixin API reference                            | [references/http-client.md](./references/http-client.md)                       |

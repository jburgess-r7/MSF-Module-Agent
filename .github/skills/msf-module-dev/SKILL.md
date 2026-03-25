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
8. Run `msftidy` (`ruby tools/dev/msftidy.rb modules/path/module.rb`) before submitting

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

## Validation Checklist

Before submitting, verify:

- [ ] `ruby -c module.rb` — no syntax errors
- [ ] `ruby tools/dev/msftidy.rb module.rb` — no warnings
- [ ] `rubocop module.rb` — no offenses (or only pre-existing style in surrounding code)
- [ ] `Notes` hash has `Stability`, `Reliability`, `SideEffects`
- [ ] `DisclosureDate` is `YYYY-MM-DD` format
- [ ] `References` use correct type identifiers (`CVE`, `EDB`, `URL`, `GHSA`)
- [ ] No hardcoded IPs, domains, or credentials
- [ ] All `send_request_cgi` calls check for `nil` response
- [ ] Module documentation `.md` file exists with Verification Steps and Scenarios
- [ ] Tested against a real vulnerable instance with console output captured

## Reference Files

| Topic                                                     | File                                                                           |
| --------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Complete annotated module template                        | [references/module-template.md](./references/module-template.md)               |
| Metadata reference (Notes, References, Rankings, Options) | [references/metadata.md](./references/metadata.md)                             |
| Credential and loot reporting patterns                    | [references/reporting.md](./references/reporting.md)                           |
| Module documentation template                             | [references/documentation-template.md](./references/documentation-template.md) |
| HttpClient mixin API reference                            | [references/http-client.md](./references/http-client.md)                       |

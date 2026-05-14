---
name: msf-module-dev
description: "Write, review, or fix Metasploit Framework modules following Rapid7 standards. Use when: creating MSF module, auxiliary module, exploit module, local exploit, post module, scanner module, metasploit development, msftidy, rubocop, MSF contribution, rapid7 PR, module metadata, HttpClient mixin, store_loot, report_cred, SOAP XML module, HTTP exploit, TCP module, FTP module, SMB module, privilege escalation."
---

# Metasploit Framework Module Development

Complete reference for writing MSF modules that pass `msftidy`, RuboCop, and Rapid7 code review.

## When to Use

- Writing a new auxiliary or exploit module
- Converting a PoC script into an MSF module
- Reviewing or fixing an existing module for PR submission
- Debugging msftidy or rubocop failures

## Module Type Classification

| If the module...                                                               | Type                   | Path example                                        |
| ------------------------------------------------------------------------------ | ---------------------- | --------------------------------------------------- |
| Gathers data via an existing vulnerability (email dump, cred theft, file read) | `Msf::Auxiliary`       | `auxiliary/gather/`                                 |
| Performs admin actions on a service (config change, user creation)             | `Msf::Auxiliary`       | `auxiliary/admin/http/` or `auxiliary/admin/smb/`   |
| Scans for the presence of a vulnerability or service                           | `Msf::Auxiliary`       | `auxiliary/scanner/http/`, `auxiliary/scanner/ftp/` |
| Achieves remote code execution (shell, meterpreter)                            | `Msf::Exploit::Remote` | `exploits/multi/http/` or `exploits/linux/http/`    |
| Achieves local privilege escalation (requires existing session)                | `Msf::Exploit::Local`  | `exploits/linux/local/`, `exploits/windows/local/`  |
| Runs after initial access (data gathering, lateral movement)                   | `Msf::Post`            | `post/multi/gather/`, `post/linux/gather/`          |

## Required File Structure

```
modules/<type>/path/module_name.rb           # The module
documentation/modules/<type>/path/module_name.md  # Documentation
```

Module filenames: lowercase, underscores, no hyphens. e.g. `webapp_xss_svg_stored.rb`

Each new module should be in its own file under the appropriate `modules/` subdirectory. In some scenarios, adding actions or targets to an existing module may be preferred.

Before writing a new module, verify there is not an existing module or open pull request that already covers the same vulnerability.

## Module Template

Every module MUST follow this exact structure. See [references/module-template.md](./references/module-template.md) for a complete annotated template.

```ruby
# frozen_string_literal: true

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

1. **Name**: Short, descriptive, title case. No special chars (`&<>=`). Include vendor name, product, and vulnerability type for searchability. Good: `'Acme FooApp Unauthenticated SQL Injection'`. Bad: `'Authenticated admin can upload crafted zip file for RCE'`.
2. **Description**: Use `%q{...}` multi-line format. Indent content to align with the `Description` key. Explain what the module does, the vulnerability, affected versions, fixed version (when known), and prerequisites.
3. **Author**: Array of strings. Format: `'Name', # role comment` (comment on same line). Use `#` comments for role attribution, not parentheses in the string. No Twitter handles (enforced by msftidy). Balanced angle brackets for email (enforced by msftidy).
    ```ruby
    'Author' => [
      'Discoverer Name', # Vulnerability discovery
      'Module Author',   # Metasploit module
    ]
    ```
4. **References**: Array of `['TYPE', 'VALUE']` pairs. **Each reference MUST be its own array**. Never combine: `['CVE', '2024-1234', 'URL', 'https://...']` is WRONG.
    ```ruby
    # CORRECT
    'References' => [
      ['CVE', '2024-1234'],
      ['URL', 'https://vendor.com/advisory']
    ]
    # WRONG — multiple refs in one array
    'References' => [
      ['CVE', '2024-1234', 'URL', 'https://vendor.com/advisory']
    ]
    ```
5. **DisclosureDate**: ISO 8601 format `'YYYY-MM-DD'`. Required for exploits. Recommended for auxiliary.
6. **Notes**: MUST contain `Stability`, `Reliability`, and `SideEffects` keys. See [references/metadata.md](./references/metadata.md).
    - `REPEATABLE_SESSION` is **only** for modules that actually create sessions (exploits). Auxiliary modules that don't create sessions must use `'Reliability' => []`.
7. **Rank**: Only for **exploit** modules. Never set `Rank` on auxiliary or post modules.
8. **License**: Always `MSF_LICENSE` unless the code has a specific BSD/MIT license.

### Code Style

1. **2-space indentation**, no tabs, no trailing whitespace
2. **`# frozen_string_literal: true`** as the very first line of every new `.rb` file (before the license header). The RuboCop cop is disabled project-wide for legacy code, but new files must include it.
3. **Single quotes** unless string interpolation is needed
4. **No `require`** for MSF/Rex libs — they autoload. Never `require 'msf/core'` or `require 'nokogiri'` (already bundled). Only `require` for stdlib not loaded by framework (e.g., `require 'fiddle'`, `require 'ipaddr'`).
5. **No `print`/`puts`** — use `print_status`, `print_good`, `print_error`, `print_warning`
6. **All `print_*` messages must start with a capital letter**
7. **No `rescue Exception`** — rescue specific errors or `StandardError`
8. **No inline `rescue`** — `disconnect rescue nil` triggers RuboCop's `Style/RescueModifier`. Use a proper `begin`/`rescue` block or let the framework handle it.
9. Hash values in `update_info` must start on the **same line** as their key
10. The `update_info(` call must start on its **own line** after `super(`
11. Multi-line `OptEnum`/`register_options` arrays: first element on a **new line** after `[` 
    ```ruby
    # GOOD
    OptEnum.new('MODE', [
      true, 'Operation mode', 'check', ['check', 'exploit']
    ])
    # BAD (triggers Layout/FirstArrayElementLineBreak)
    OptEnum.new('MODE', [true, 'Operation mode', 'check',
                         ['check', 'exploit']])
    ```
12. Use `Rex::Version` for version comparisons instead of manual major/minor/patch splitting
    ```ruby
    # GOOD
    Rex::Version.new(version) < Rex::Version.new('4.2.1')
    # BAD
    major, minor, patch = version.split('.').map(&:to_i)
    if major < 4 || (major == 4 && minor < 2)
    ```
13. Prefer hash return values over arrays for methods returning multiple distinct items. Use kwargs for reusable APIs.
14. Don't use `get_`/`set_` prefixes for accessor methods in new code
15. Method parameter names must be at least 2 characters (except well-known crypto abbreviations)
16. Prefer module mixin APIs over reimplementing core functionality
17. Use `Faker` (e.g. `Faker::Internet.password`, `Faker::Internet.username`) for generating fake usernames/accounts — not `Rex::Text.rand_text_alphanumeric`
18. Use `Rex::Socket.to_authority(ip, port)` instead of `"#{ip}:#{port}"` for host:port formatting — handles IPv6 addresses correctly
19. Use `Rex::Stopwatch.elapsed_time` to track elapsed time
20. Use `Rex::MIME::Message` for constructing MIME messages — don't hardcode MIME boundaries
21. Use `Rex::RandomIdentifier::Generator` for generating random variable names, specifying the target language to avoid generating language keywords
22. Don't set a default payload (`DefaultOptions` with `'PAYLOAD'`) in modules unless absolutely necessary — let the framework choose the most appropriate payload automatically
23. Use `Msf::Exploit::SQLi` mixin when exploiting SQL injection vulnerabilities
24. If there is only one action in an exploit, omit `Actions`/`DefaultAction` unless there is a clear reason to keep them
25. When checking for a string in a response, consider whether it will always be in English across different locales/versions
26. Ensure hardcoded strings used in regex matching will be consistent across multiple software versions
27. When opening a file on the target, verify the file exists first
28. Use the TEST-NET-1 range (`192.0.2.0/24`) for example/non-routable IP addresses in unit tests and spec files. Local/private IPs are fine in module documentation scenarios.

### Common Patterns

- Register options with `register_options` and `register_advanced_options`
- Use `SCREAMING_SNAKE_CASE` option names and `CamelCase` advanced option names
- Use `datastore['OPTION_NAME']` to access module options
- Use `print_status`, `print_good`, `print_error`, `print_warning` for console output
- Use `vprint_*` variants for verbose-only output
- Use `send_request_cgi` for HTTP requests in modules
- Use `connect`/`disconnect` for TCP socket operations

### HTTP Modules

1. Always `include Msf::Exploit::Remote::HttpClient`
2. Use `send_request_cgi({...})` — NEVER raw HTTP libraries
3. Always check for `nil` response (timeout): `fail_with(Failure::Unreachable, '...') unless res`
4. Always check body with `.to_s` before string operations: `res.body.to_s.include?('foo')` — `res.body` can be nil
5. Use `normalize_uri(target_uri.path, 'endpoint')` for URL paths. `normalize_uri` does not percent-encode path segments — if you embed user-supplied values in a path (e.g. a username or identifier), encode special characters manually before passing them in (`str.gsub('@', '%40')` etc.).
6. **Do NOT override `target_uri`** — the HttpClient mixin defines it. If you need the base path, use `datastore['TARGETURI']` directly.
7. **Do NOT re-register options that HttpClient already provides** — `RHOSTS`, `RPORT`, `VHOST`, `SSL`, `TARGETURI` are auto-registered by the mixin. Re-registering them causes duplicate options.
8. Set sensible `DefaultOptions` for `RPORT` and `SSL`
    ```ruby
    'DefaultOptions' => {
      'RPORT' => 443,
      'SSL' => true
    }
    ```
    Without this, the module defaults to port 80/plaintext even for HTTPS-only services.
9. Use `res.get_json_document` to parse JSON responses — never manual `JSON.parse(res.body)`
10. Use `res.get_html_document` for HTML parsing (returns a Nokogiri document) — useful for extracting CSRF tokens, form fields, version strings
11. For XML/SOAP responses, use `res.get_xml_document` or `Nokogiri::XML(res.body.to_s)` — then `doc.remove_namespaces!` to simplify XPath queries across SOAP namespaces. See the [HttpClient reference](./references/http-client.md) for patterns.
12. **XML escaping**: When injecting user input into SOAP/XML request bodies, always XML-escape the values to prevent XML injection. Define a private `xml_escape` helper or use `CGI.escapeHTML`:
    ```ruby
    def xml_escape(str)
      str.to_s.gsub('&', '&amp;').gsub('<', '&lt;').gsub('>', '&gt;').gsub('"', '&quot;').gsub("'", '&apos;')
    end
    ```
13. Use `Rex::Text::Table` for formatted output
14. Cookie jar is automatic with `keep_cookies: true` — don't manually track cookies with local variables
15. **Password/credential option defaults**: Use `nil` for passwords, not empty string `''`. Empty string is only acceptable if the product's default password is literally blank.
    ```ruby
    # GOOD — forces user to set a value
    OptString.new('PASSWORD', [true, 'Password', nil])
    # BAD — empty string implies blank is normal
    OptString.new('PASSWORD', [true, 'Password', ''])
    ```
16. For payload-triggering requests that won't return (shell established), set `timeout` to 0 or 1:
    ```ruby
    send_request_cgi({ 'uri' => shell_uri, 'method' => 'POST' }, 1)
    ```
17. For exploit modules, prefer `WfsDelay` (wait-for-session delay) over custom sleep options

### Non-HTTP Modules (TCP, FTP, SMB, etc.)

1. Use the appropriate protocol mixin (`Msf::Exploit::Remote::Tcp`, `Msf::Exploit::Remote::Ftp`, `Msf::Exploit::Remote::SMB::Client`, etc.)
2. For raw TCP: `connect`/`disconnect`, `sock.put(data)`, `sock.get_once(len, timeout)`
3. Always handle `Rex::ConnectionError` (connection refused, timeout)
4. For FTP: use `connect_login` for auth, `send_cmd` for commands
5. Use the `RubySMB` library for SMB modules
6. See [references/non-http-modules.md](./references/non-http-modules.md) for templates and protocol mixin reference

### Payload Delivery (CmdStager vs Fetch)

When an exploit module supports binary payloads (dropper targets), choose the right delivery mechanism:

- **CmdStager** (`include Msf::Exploit::CmdStager`): Splits a binary payload into small chunks delivered via the command injection channel itself. Useful when the target has no outbound HTTP/network access. Flavors: `printf`, `bourne`, `wget`, `curl`, `certutil`, `psh_invokewebrequest`.
- **Fetch payloads** (e.g., `fetch/linux/x64/meterpreter/reverse_tcp`): MSF starts an HTTP server and the target downloads the payload binary via `curl`/`wget`. Simpler and more reliable than CmdStager when the target can reach the attacker's HTTP server.

**Guidance**: If the target environment has `curl`/`wget` available (most Linux servers), fetch payloads are simpler and preferred. CmdStager is useful when outbound HTTP is blocked or the tools aren't available. Consider offering both via separate targets if the exploit channel supports it.

**Additional payload rules**:
- Use `ARCH_CMD` payloads instead of command stagers when only `curl`/`wget` and other download mechanisms are available
- Define bad characters (`'BadChars'`) instead of explicitly base64-encoding payloads
- Don't check the number of sessions at the end of an exploit and report success based on that — not all payloads open sessions
- Don't submit opaque binary blobs — everything must include source code and build instructions

### Scanner Modules

1. `include Msf::Auxiliary::Scanner` — **after** protocol-specific mixins
2. Implement `def run_host(ip)` — do NOT define `def run`
3. `RHOSTS` and `THREADS` are auto-registered by the Scanner mixin
4. Use `peer` for log messages (returns `"host:port"`)

### Local Exploit Modules

1. Inherit from `Msf::Exploit::Local` (not `Msf::Exploit::Remote`)
2. **Must** specify `'SessionTypes'` (e.g., `['shell', 'meterpreter']`)
3. Use `prepend Msf::Exploit::Remote::AutoCheck` (always **prepend**, never include)
4. Use `Msf::Post::File` for `read_file`, `write_file`, `file_exist?`
5. Use `Msf::Exploit::FileDropper` + `register_file_for_cleanup` for artifact cleanup
6. Use `create_process(executable, args: [], time_out: 15, opts: {})` instead of the deprecated `cmd_exec` with separate arguments. `cmd_exec(command)` with a single string is still fine.

### Post-Exploitation Modules

1. Inherit from `Msf::Post`
2. **Must** specify `'Platform'` and `'SessionTypes'`
3. Use `session` to interact with the target (`.type`, `.platform`, `.session_host`)
4. Common mixins: `Msf::Post::File`, `Msf::Post::Linux::Priv`, `Msf::Post::Windows::Registry`
5. Use `store_loot` with `session` (not `rhost`) as the host parameter

### Check Method

Implement a `check` method whenever possible so users can verify vulnerability status before exploitation.

Return `CheckCode` constants (not booleans) from `def check`:

- `CheckCode::Safe` — not vulnerable
- `CheckCode::Detected` — service running, vuln status unknown
- `CheckCode::Appears('reason')` — likely vulnerable (version-based only)
- `CheckCode::Vulnerable('reason')` — confirmed vulnerable (the vulnerability has been exploited/proven)
- `CheckCode::Unknown` — cannot determine

**Important semantics**: `CheckCode::Vulnerable` should only be used when the vulnerability has actually been confirmed by triggering it (e.g. injecting a canary value). `CheckCode::Appears` should only be used when the application's version has been checked and falls within the vulnerable range. Don't use `Vulnerable` for version-only checks or `Appears` for non-version-based heuristics.

**Namespace gotcha for Auxiliary modules**: In `Msf::Exploit::Remote` subclasses, bare `CheckCode::Vulnerable` resolves fine. In `Msf::Auxiliary` subclasses, you **must** use the fully qualified `Msf::Exploit::CheckCode::Vulnerable('reason')` (and likewise for `Safe`, `Detected`, etc.). Using the bare constant causes `NameError: uninitialized constant`.

```ruby
# In Msf::Auxiliary — MUST use full namespace
def check
  return Msf::Exploit::CheckCode::Detected('Service found') if service_present?
  Msf::Exploit::CheckCode::Safe('Not found')
end
```

**Key rules from PR reviews:**

- **Never raise exceptions or call `fail_with` in check** — always return a CheckCode. Do catch exceptions that may be raised and ensure a valid CheckCode is returned.
- **Keep check logic completely separate from run/exploit logic** — don't use boolean flags that change a method's return type between CheckCode and data
- **Verify check doesn't false-positive against unrelated software** — if you check for `/admin/dashboard` existing, ensure the response actually contains the expected application's fingerprint. Use specific regular expressions or `res.get_html_document` with CSS selectors for version extraction — don't use generic selectors like `href .*` to grab the version.
- **Research minimum vulnerable version** — determine a minimum version where the application is vulnerable and mark prior versions as safe
- **Verify check helper methods** — if helper methods are shared between `#check` and `#exploit`/`#run`, ensure there is no condition (exception, return, etc.) where `#check` could return something other than a CheckCode
- Use `prepend Msf::Exploit::Remote::AutoCheck` to auto-run check before exploit — always `prepend`, never `include`. Prefer this over manually calling `check` inside `exploit`
- `get_version` methods should return a `Rex::Version` object

### Credential and Data Reporting

Use `include Msf::Auxiliary::Report` and call:

- **`store_loot(type, ctype, host, data, filename, description)`** — for exfiltrated files/data. **Always use `store_loot` instead of `File.write`** for saving data to disk.
    - **Exception**: `File.write` is acceptable for **local** temp files needed during exploitation (e.g., writing to `Dir.mktmpdir` for git operations, creating local payload files). These must be cleaned up in the `cleanup` method.
- **`create_credential` + `create_credential_login`** — for discovered credentials
- **`report_service(host:, port:, proto:, name:)`** — to record identified services
- **`report_vuln(host:, port:, proto:, name:, info:, refs:)`** — to record a confirmed vulnerability in the database once the module has positively confirmed it. Usually call this from `run`, `run_host`, or `exploit`; calling it from `check` is acceptable when the module intentionally records confirmation there. Pass `refs: references` to link the module's References array. See [references/reporting.md](./references/reporting.md).

**Scanner/gather modules MUST report findings** — if your module discovers data but doesn't call `store_loot`, `report_vuln`, or `report_cred`, reviewers will reject it.

See [references/reporting.md](./references/reporting.md) for the full credential reporting pattern.

### Cleanup

Use the `cleanup` method for artifact removal (shell files, temp data). **Always call `super`** to ensure parent mixins clean up connections/sessions:

```ruby
def cleanup
  super
  return unless @shell_uri

  send_request_cgi(
    'method' => 'POST',
    'uri' => @shell_uri,
    'vars_post' => { @cleanup_param => '1' },
    'timeout' => 5
  )
end
```

Do NOT put cleanup logic at the end of `exploit` — if something fails or the session dies, `cleanup` is guaranteed to run but code after the exploit point may not.

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

**Avoid broad rescue blocks that swallow errors silently** — the framework catches exceptions at the top level. Wrapping the entire body of `run` in `rescue ::StandardError => e; print_error(e.message)` silently hides errors that should propagate to the user. Specific rescues for known exceptions (e.g., `rescue Timeout::Error => e`) are acceptable; general error swallowing is not.

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
# Use the MSF-bundled rubocop binary, not the system one
/opt/metasploit-framework/embedded/bin/rubocop --config .rubocop.yml /path/to/my_module.rb
```

If the framework is installed elsewhere, find the binary with:

```bash
find /opt -name rubocop -path '*/metasploit*' 2>/dev/null | head -1
```

If the file is outside the framework `modules/` directory, `Style/Documentation` may still flag — this is a false positive that disappears when the file is in its final location.

### Running msftidy_docs

Validate documentation markdown files with:

```bash
cd /opt/metasploit-framework/embedded/framework
ruby tools/dev/msftidy_docs.rb /path/to/documentation/modules/type/path/module_name.md
```

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

- [ ] `# frozen_string_literal: true` is the first line of every new `.rb` file
- [ ] `ruby -c module.rb` — no syntax errors
- [ ] `msftidy` — status 0, no warnings (file must be under a `modules/` path tree)
- [ ] `msftidy_docs` — `ruby tools/dev/msftidy_docs.rb <documentation_file>` passes on documentation markdown
- [ ] `rubocop --config .rubocop.yml` — no offenses (run from MSF framework dir)
- [ ] `Notes` hash has `Stability`, `Reliability`, `SideEffects`
- [ ] `Reliability` uses `REPEATABLE_SESSION` only if module creates sessions
- [ ] `Rank` is set only on exploit modules (not auxiliary or post)
- [ ] `DisclosureDate` is `YYYY-MM-DD` format
- [ ] `References` — each ref is its own array; no combined `['CVE', 'x', 'URL', 'y']`
- [ ] `Author` — format is `'Name', # role comment` (not parens-in-string)
- [ ] Module name is descriptive and searchable (vendor + product + vuln type)
- [ ] Description lists vulnerable versions and fixed version (when known)
- [ ] No hardcoded IPs, domains, or credentials; password defaults are `nil` not `''`
- [ ] No `require` for bundled MSF/Rex libs; no `require 'msf/core'`
- [ ] Does not re-register mixin-provided options (RHOSTS, RPORT, VHOST, SSL, TARGETURI)
- [ ] Does not override `target_uri` method
- [ ] All `print_*` messages start with a capital letter
- [ ] All `send_request_cgi` calls check for `nil` response (HTTP modules)
- [ ] All response body access uses `.to_s` (e.g., `res.body.to_s.include?('...')`)
- [ ] Payload-triggering requests use `timeout: 0` or `1`
- [ ] JSON parsed via `res.get_json_document`, not `JSON.parse`
- [ ] XML/SOAP modules XML-escape all user input injected into request bodies
- [ ] All TCP `connect` calls handle `Rex::ConnectionError` (non-HTTP modules)
- [ ] Scanner modules implement `run_host(ip)`, not `run`
- [ ] Local exploit / post modules specify `SessionTypes`
- [ ] Uses `create_process` instead of `cmd_exec` with separate arguments (post/local modules)
- [ ] `check` method returns `CheckCode` constants, never calls `fail_with`
- [ ] `check` method tested against non-matching target to avoid false positives
- [ ] `CheckCode::Vulnerable` only used when vuln is confirmed; `CheckCode::Appears` only for version-based checks
- [ ] Uses `cleanup` method for artifact removal (not manual cleanup at end of exploit); `cleanup` calls `super`
- [ ] Uses `store_loot` for saving data (not `File.write`)
- [ ] Scanner/gather modules report findings via `store_loot` or `report_cred`
- [ ] Calls `report_vuln` in `run`/`exploit` after confirming the vulnerability
- [ ] Calls `report_service` when the module's purpose is to enumerate/detect services (scanner/gather modules)
- [ ] Uses `Faker` for fake usernames/accounts (not `rand_text_alphanumeric`)
- [ ] Uses `Rex::Socket.to_authority` for host:port formatting (not string interpolation)
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
| Non-HTTP module patterns (Scanner, TCP, Local, Post)      | [references/non-http-modules.md](./references/non-http-modules.md)             |

# Metadata Reference

## Notes Hash (REQUIRED)

Every module must include a `Notes` hash with all three keys. Enforced by `Lint/ModuleEnforceNotes` rubocop cop.

```ruby
'Notes' => {
  'Stability' => [CRASH_SAFE],
  'Reliability' => [],
  'SideEffects' => [IOC_IN_LOGS]
}
```

### Stability Constants

| Constant                 | Meaning                                                |
| ------------------------ | ------------------------------------------------------ |
| `CRASH_SAFE`             | Module will not crash the service                      |
| `CRASH_SERVICE_RESTARTS` | May crash the service, but it auto-restarts            |
| `CRASH_SERVICE_DOWN`     | May crash the service permanently                      |
| `CRASH_OS_RESTARTS`      | May crash the OS, but it auto-restarts                 |
| `CRASH_OS_DOWN`          | May crash or brick the OS                              |
| `SERVICE_RESOURCE_LOSS`  | May consume/destroy service resources (data, accounts) |
| `OS_RESOURCE_LOSS`       | May consume/destroy OS resources (disk, memory)        |

### Reliability Constants

| Constant             | Meaning                                                    |
| -------------------- | ---------------------------------------------------------- |
| `REPEATABLE_SESSION` | Consistently produces a session                            |
| `UNRELIABLE_SESSION` | May or may not produce a session                           |
| `FIRST_ATTEMPT_FAIL` | First attempt typically fails, subsequent attempts succeed |
| `EVENT_DEPENDENT`    | Requires a specific event (user action, cron job, etc.)    |

**Important**: `REPEATABLE_SESSION` is **only** for modules that actually create sessions (exploits with payloads). Auxiliary modules that gather data or admin modules use `'Reliability' => []`.

### SideEffects Constants

| Constant            | Meaning                                                    |
| ------------------- | ---------------------------------------------------------- |
| `IOC_IN_LOGS`       | Leaves indicators of compromise in application/system logs |
| `ARTIFACTS_ON_DISK` | Creates files on the target system                         |
| `CONFIG_CHANGES`    | Modifies target configuration                              |
| `ACCOUNT_LOCKOUTS`  | May trigger account lockout policies                       |
| `ACCOUNT_LOGOUT`    | May log users out                                          |
| `SCREEN_EFFECTS`    | Produces visible changes on screen                         |

Use `UNKNOWN_STABILITY`, `UNKNOWN_RELIABILITY`, or `UNKNOWN_SIDE_EFFECTS` (arrays) if genuinely unsure — but these should be replaced before PR submission.

## References

```ruby
'References' => [
  ['CVE', '2024-12345'],        # CVE ID (no "CVE-" prefix)
  ['EDB', '12345'],             # Exploit-DB ID
  ['URL', 'https://...'],       # Any URL
  ['BID', '12345'],             # Bugtraq ID
  ['MSB', 'MS17-010'],          # Microsoft Security Bulletin
  ['US-CERT-VU', '123456'],     # US-CERT Vulnerability Note
  ['ZDI', '20-1234'],           # Zero Day Initiative
  ['WPVDB', '12345'],           # WPScan Vulnerability Database
  ['PACKETSTORM', '123456'],    # Packet Storm
  ['GHSA', 'xxxx-yyyy-zzzz'],  # GitHub Security Advisory
  ['OSV', 'GHSA-xxxx-yyyy'],   # Open Source Vulnerabilities
]
```

**msftidy validates**: CVE format must be `YYYY-NNNN+` (4-digit year, 4+ digit ID). EDB and BID must be numeric.

## Rankings (Exploit modules only)

**Rank is ONLY for exploit modules**. Never set `Rank` on auxiliary or post modules — reviewers will flag it immediately.

```ruby
class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking
```

| Ranking            | Value | Use when                                                            |
| ------------------ | ----- | ------------------------------------------------------------------- |
| `ExcellentRanking` | 600   | No memory corruption; works every time with no side effects         |
| `GreatRanking`     | 500   | Has default target config; works in common cases automatically      |
| `GoodRanking`      | 400   | Has default target but may not auto-detect; reliable in general     |
| `NormalRanking`    | 300   | Generally reliable but requires specific config or version matching |
| `AverageRanking`   | 200   | Sometimes works; may require multiple attempts                      |
| `LowRanking`       | 100   | Rarely works; very specific conditions                              |
| `ManualRanking`    | 0     | Essentially a DoS or requires significant manual intervention       |

## Option Types

```ruby
# Standard options
OptString.new('NAME', [required?, 'Description', 'default'])
OptInt.new('COUNT', [true, 'Number of items', 10])
OptBool.new('SSL', [false, 'Use TLS', false])
OptEnum.new('MODE', [true, 'Operation mode', 'check', ['check', 'exploit', 'dump']])

# Multi-line form (use when line exceeds ~120 chars):
# First element MUST start on a new line after [ (Layout/FirstArrayElementLineBreak)
OptEnum.new('PAYLOAD_TYPE', [
  true, 'Payload format to use', 'alert',
  ['alert', 'steal_data', 'custom']
])
OptPath.new('WORDLIST', [false, 'Path to wordlist file'])
OptAddress.new('SRVHOST', [true, 'Callback listener address', '0.0.0.0'])
OptPort.new('SRVPORT', [true, 'Callback listener port', 8080])
OptAddressRange.new('RHOSTS', [true, 'Target address range'])
OptRegexp.new('PATTERN', [false, 'Regex filter pattern'])

# Convenience shortcuts
Opt::RHOST         # Remote host
Opt::RPORT(443)    # Remote port with default
Opt::LHOST         # Local host (for payloads/callbacks)
Opt::LPORT(4444)   # Local port
Opt::Proxies       # Proxy chain support
```

## DefaultOptions

```ruby
'DefaultOptions' => {
  'RPORT' => 443,
  'SSL' => true,
  'VHOST' => '',
  'HttpClientTimeout' => 15,
  'WfsDelay' => 10       # Wait-for-session delay (exploit modules)
}
```

## Actions (Auxiliary modules with multiple modes)

```ruby
'Actions' => [
  ['Check', { 'Description' => 'Verify the target is vulnerable' }],
  ['Dump', { 'Description' => 'Exfiltrate data from the target' }]
],
'DefaultAction' => 'Check'
```

Access in `run` via `action.name`.

## Platform and Arch (Exploit modules)

```ruby
'Platform' => %w[linux unix win],
'Arch' => [ARCH_CMD],           # For command injection
# or
'Arch' => [ARCH_X64, ARCH_X86], # For binary payloads

'Targets' => [
  ['Automatic', {}],
  ['Linux x64', { 'Platform' => 'linux', 'Arch' => ARCH_X64 }]
],
'DefaultTarget' => 0,
'Privileged' => false
```

## DisclosureDate

Format: `'YYYY-MM-DD'` (ISO 8601). Enforced by `Lint/ModuleDisclosureDateFormat`.

- **Required** for exploit modules (enforced by `Lint/ModuleDisclosureDatePresent`)
- **Recommended** for auxiliary modules
- Set to the date the vulnerability was publicly disclosed or patched

# Metadata, options, targets, and payloads

Use the current module schema, RuboCop cops, msftidy checks, and nearby current modules together. Metadata describes real behavior; it is not a box-ticking exercise.

## Core metadata

```ruby
super(
  update_info(
    info,
    'Name' => 'Acme App Unauthenticated Command Injection',
    'Description' => %q{
      This module exploits an unauthenticated command injection in Acme App.
      Versions 4.0.0 through 4.2.1 are affected. Version 4.2.2 fixes the issue.
    },
    'Author' => [
      'Researcher', # Vulnerability discovery
      'Contributor' # Metasploit module
    ],
    'License' => MSF_LICENSE,
    'References' => [
      ['CVE', '2026-12345'],
      ['URL', 'https://vendor.example/advisory']
    ],
    'DisclosureDate' => '2026-01-15',
    'Notes' => {
      'Stability' => [CRASH_SAFE],
      'Reliability' => [REPEATABLE_SESSION],
      'SideEffects' => [IOC_IN_LOGS, ARTIFACTS_ON_DISK]
    }
  )
)
```

- `Name`: vendor/product plus vulnerability/effect, concise and searchable.
- `Description`: `%q{}` for a long multiline description, ASCII only. Explain prerequisites, vulnerable range, and fixed version when known.
- `Author`: role comments belong in Ruby comments, not in the author string.
- `References`: each reference is its own two-element array. CVE values omit the `CVE-` prefix.
- `DisclosureDate`: ISO `YYYY-MM-DD`; required for exploits.
- `License`: new Metasploit modules normally use `MSF_LICENSE`.
- `Rank`: exploit modules only. Select it from demonstrated reliability and side effects; do not add Rank to auxiliary/post modules.

## Notes reflect behavior

Exploit, auxiliary, and post modules require Notes with `SideEffects`; current lint may require the full Stability/Reliability/SideEffects structure. Choose values based on every code path.

### Stability

| Constant | Meaning |
| --- | --- |
| `CRASH_SAFE` | Expected not to crash the target |
| `CRASH_SERVICE_RESTARTS` | Service may crash and restart |
| `CRASH_SERVICE_DOWN` | Service may remain unavailable |
| `CRASH_OS_RESTARTS` | OS may crash and restart |
| `CRASH_OS_DOWN` | OS may remain unavailable |
| `SERVICE_RESOURCE_LOSS` | Service data/resources may be consumed or lost |
| `OS_RESOURCE_LOSS` | Host resources may be consumed or lost |

### Reliability

| Constant | Meaning |
| --- | --- |
| `REPEATABLE_SESSION` | Demonstrably creates repeatable sessions |
| `UNRELIABLE_SESSION` | Session creation is unreliable |
| `FIRST_ATTEMPT_FAIL` | The first attempt commonly fails |
| `EVENT_DEPENDENT` | Session depends on an external event |

Do not use `REPEATABLE_SESSION` on a scanner/gather/admin module that never creates a session; use an empty Reliability array where appropriate.

### Side effects

| Constant | Typical evidence |
| --- | --- |
| `IOC_IN_LOGS` | Authentication attempts, exploit requests, process execution, or other logged activity |
| `ARTIFACTS_ON_DISK` | Any temporary or persistent target file |
| `CONFIG_CHANGES` | Accounts, settings, services, registry/configuration, or persistent state changes |
| `ACCOUNT_LOCKOUTS` | Brute-force/password-spray attempts can lock accounts |
| `ACCOUNT_LOGOUT` | Existing sessions may be invalidated |
| `SCREEN_EFFECTS` | Visible UI/display changes |

Examples:

- An `AuthBrute` scanner normally has `IOC_IN_LOGS` and `ACCOUNT_LOCKOUTS`.
- A write-access probe that creates and removes a file still has `ARTIFACTS_ON_DISK`.
- Cleanup does not erase the fact that an artifact/config change occurred.

## Options

Normal options use `SCREAMING_SNAKE_CASE`; advanced options use `CamelCase`:

```ruby
register_options(
  [
    OptString.new('USERNAME', [true, 'Username for Acme App', nil]),
    OptString.new('PASSWORD', [true, 'Password for Acme App', nil]),
    OptInt.new('COUNT', [true, 'Number of records to retrieve', 100]),
    OptEnum.new('MODE', [true, 'Operation mode', 'read', %w[read write]])
  ]
)

register_advanced_options(
  [
    OptInt.new('VerifyTimeout', [true, 'Seconds to wait for asynchronous completion', 60])
  ]
)
```

- Do not re-register options a mixin provides.
- Use current mixin accessors such as `srvhost` instead of direct `datastore['SRVHOST']` access when the repository lint requires them.
- Required authentication secrets normally default to `nil`, not `''`. A blank default is valid only when a blank secret is a documented product default.
- This rule is not about credentials created by an exploit. Generate created usernames/passwords with `Faker` and expose override options only where useful.
- Avoid custom sleep/timeout options when `WfsDelay`, `HttpClientTimeout`, or another framework option already owns the behavior.
- Option descriptions explain units and scope. Error messages for a configurable timeout name the option the user can change.
- Use option `conditions` when a value is relevant only under another mode/target.

## DefaultOptions

Set only defaults that describe the service or are required for correct execution:

```ruby
{
  'DefaultOptions' => {
    'RPORT' => 443,
    'SSL' => true
  }
}
```

Avoid arbitrary `HttpClientTimeout`, `WfsDelay`, `VHOST`, and `PAYLOAD` defaults. Use them only when target behavior establishes a need and document/test the reason.

Do not set a default payload merely because it was the payload used during development. Let the framework choose where possible. A target-specific default can be justified when framework selection cannot otherwise choose correctly, but it must be tested while switching targets without explicitly resetting `PAYLOAD`.

## Targets, architecture, and platform

```ruby
{
  'Targets' => [
    [
      'Unix/Linux Command',
      {
        'Platform' => %w[linux unix],
        'Arch' => ARCH_CMD,
        'Type' => :unix_cmd
      }
    ],
    [
      'Linux Dropper',
      {
        'Platform' => 'linux',
        'Arch' => [ARCH_X64, ARCH_AARCH64],
        'Type' => :linux_dropper
      }
    ]
  ],
  'DefaultTarget' => 0
}
```

- Name command targets “Unix/Linux Command” or “Windows Command,” not “shell,” when they execute `ARCH_CMD` payloads.
- Do not repeat top-level `Platform`/`Arch` when every target already defines them; the current linter flags redundant metadata.
- Use target metadata such as `target['Type']` for dispatch. Do not store sentinel values in unrelated instance variables.
- Use architecture constants (`ARCH_X64`, `ARCH_AARCH64`, and so on), not regexes over `sysinfo` strings.
- Include only platforms/architectures the primitive and delivery path support.
- Use an Automatic target only when the module truly detects and selects another behavior.
- Prefer a command/fetch-capable target to a CmdStager dropper when the only staging mechanism is curl/wget. Retain a binary/dropper target only for a demonstrated capability or restriction.
- Define bad characters in payload metadata instead of manually base64-encoding payloads to work around them.
- Do not call `handler` manually unless the current exploit flow specifically requires it; normal framework execution owns the handler.

## Actions

Use actions only for genuinely distinct auxiliary/module operations:

```ruby
{
  'Actions' => [
    ['ENUMERATE', { 'Description' => 'Enumerate Acme App users' }],
    ['RESET', { 'Description' => 'Reset an Acme App account password' }]
  ],
  'DefaultAction' => 'ENUMERATE'
}
```

Dispatch with `action.name`. If an exploit has only one action, omit the action metadata.

## DefangedMode

For materially destructive or irreversible behavior, follow current framework precedent and register the standard guard as an advanced option:

```ruby
register_advanced_options(
  [
    OptBool.new('DefangedMode', [true, 'Run in defanged mode', true])
  ]
)

fail_with(
  Failure::BadConfig,
  'This action irreversibly changes target data; set DefangedMode false to continue'
) if datastore['DefangedMode']
```

Do not add a second acknowledgement option. Do not require DefangedMode merely because a normal exploit creates cleanup-managed, reversible artifacts.

## Metadata QA

- Compare metadata with the complete exploit path and cleanup ledger.
- Verify SideEffects for brute force, remote writes, account creation, configuration changes, and cleanup-managed artifacts.
- Test every target/platform/architecture and target switching without a manually pinned payload.
- Ensure module and documentation option names/defaults match exactly.
- Re-run msftidy/RuboCop after changing metadata; lint rules evolve.

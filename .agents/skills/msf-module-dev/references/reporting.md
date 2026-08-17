# Reporting services, vulnerabilities, credentials, and loot

Include `Msf::Auxiliary::Report` unless an inherited mixin already provides the reporting API.

Reporting should describe the object actually discovered and preserve its relationships. A host and port alone are not enough when an application, HTTP/TLS layer, and TCP transport all share the same endpoint.

## Report the application service early

Report the application as soon as it is positively identified. Do not wait until credentials are found or exploitation finishes. Keep the helper safe to call from both `check` and `exploit`, because `AutoCheck false` bypasses `check`.

```ruby
def report_acme_service(version: nil)
  common = { host: rhost, port: rport, proto: 'tcp' }
  tcp_service = common.merge(name: 'tcp', parents: nil)

  web_service = if ssl
                  common.merge(
                    name: 'https',
                    parents: common.merge(name: 'ssl', parents: tcp_service)
                  )
                else
                  common.merge(name: 'http', parents: tcp_service)
                end

  report_service(
    common.merge(
      name: 'acme-app',
      info: version ? "Acme App #{version}" : 'Acme App',
      resource: { uri: normalize_uri(target_uri.path) },
      parents: web_service
    )
  )
end
```

The resulting hierarchy is:

```text
acme-app(resource: base URI) -> http -> tcp
acme-app(resource: base URI) -> https -> ssl -> tcp
```

Rules:

- `name` is the application/service identity; `info` is a banner or version description.
- Use `resource` to distinguish applications or resources sharing an endpoint, such as a base URI, LDAP DN, SMB share, or named pipe.
- `parents` accepts a service hash or array of service hashes. Give the terminal transport `parents: nil` when constructing a full chain.
- The database makes `report_service` idempotent for the same identifying attributes. Memoization is useful, but code must still work when no database is active and the helper returns `nil`.
- Protocol mixins may already own transport reporting. Inspect the current mixin before duplicating its service or host record.
- A separate `report_host` call is normally redundant: service, vulnerability, and credential APIs report the host as needed.
- If reachability is known before any service/product can be identified, a host-only report can be appropriate; do not repeat it once a later reporting API owns the same host evidence.

## Associate a confirmed check result

`Vulnerable` requires direct, vulnerability-specific evidence obtained by safely exercising the flaw. Product identification alone is `Detected`; a version-range match alone is `Appears`.

For an interactive `check` command, attach the exact service to the CheckCode vulnerability metadata:

```ruby
service = @acme_service ||= report_acme_service(version: version)

Exploit::CheckCode::Vulnerable(
  'Confirmed command injection with a harmless marker',
  vuln: { service: service }
)
```

The console check dispatcher uses `vuln:` when reporting the result. AutoCheck separately calls the module's `report_vuln`. If AutoCheck must always associate the same application service, use a narrowly scoped override:

```ruby
def report_vuln(opts = {})
  service = opts[:service] || @acme_service || report_acme_service
  super(opts.merge(service: service))
end
```

Only use this override when the service helper is safe before full identification; otherwise report after identification and pass `service:` explicitly. Do not rely on `port`/`proto` fallback: the database may select the first same-port service, commonly the TCP parent.

Current limitation: only `Exploit::CheckCode::Vulnerable` accepts `vuln:` metadata. A manual `Appears` check is reported by the console dispatcher using generic host/port data and does not call the module's override, so exact application-service linkage is not currently expressible there. Do not pass an unsupported `vuln:` keyword to `Appears`; ensure AutoCheck and explicit exploit-time reporting are exact and document this manual-check limitation when it matters.

## Report a vulnerability explicitly

Call `report_vuln` only after the module has evidence for the vulnerability, not merely ordinary successful authentication or a generic product banner.

```ruby
report_vuln(
  host: rhost,
  port: rport,
  proto: 'tcp',
  service: @acme_service,
  name: name,
  refs: references,
  info: 'Unauthenticated command injection was confirmed with a harmless marker'
)
```

Use `refs: references` to preserve module references. Put vulnerability-specific evidence in the vulnerability `info`; do not overload the service banner with it.

Avoid duplicate reporting. AutoCheck already reports `Appears` and `Vulnerable` results through `report_vuln`. An explicit later report is appropriate when it adds exploit-time confirmation or a different resource, but should identify the same service and avoid creating semantically duplicate records.

## Credential and login reporting

Use `create_credential` for a credential whose validity is unknown. Add a login only when it is meaningful to associate the credential with a service and status. `create_credential_and_login` is convenient when both are known.

```ruby
def report_admin(username, password)
  service = @acme_service || report_acme_service

  data = {
    workspace_id: myworkspace_id,
    origin_type: :service,
    module_fullname: fullname,
    username: username,
    private_type: :password,
    private_data: password,
    address: rhost,
    port: rport,
    protocol: 'tcp',
    service_name: service&.name || 'acme-app',
    access_level: 'administrator',
    status: Metasploit::Model::Login::Status::SUCCESSFUL,
    last_attempted_at: Time.now
  }
  data[:service_id] = service.id if service

  create_credential_and_login(data)
end
```

Important fields:

| Field | Guidance |
| --- | --- |
| `origin_type` | Usually `:service` for remote modules; post modules commonly use `:session` with session provenance |
| `private_type` | `:password`, `:nonreplayable_hash`, `:ssh_key`, or the correct supported type |
| `service_name` | Match the application service, not a generic same-port TCP record |
| `service_id` | Use the exact reported service ID when available |
| `access_level` | Set when known, such as `administrator` or `user` |
| `status` | `SUCCESSFUL` only when authentication or account usability was demonstrated; otherwise use the appropriate status/UNTRIED flow |
| `last_attempted_at` | Set for a demonstrated login attempt, especially `SUCCESSFUL` |

Do not report the same credential, access level, or banner again as a note. Do not report a normal successful login as a vulnerability unless the login itself demonstrates a vulnerability, such as default credentials that are explicitly the module's finding.

Login scanners have a stronger contract: connection errors and timeouts are `UNABLE_TO_CONNECT`, not incorrect credentials. Inspect the current LoginScanner result APIs and shared specs before implementing a scanner.

## Store loot

Use `store_loot` for retrieved data instead of writing directly into an arbitrary local path:

```ruby
loot_path = store_loot(
  'acme.config',
  'application/json',
  rhost,
  config_json,
  'acme-config.json',
  'Acme App configuration export',
  @acme_service
)
print_good("Configuration saved to #{loot_path}")
```

Signature:

```ruby
store_loot(type, content_type, host, data, filename = nil, info = nil, service = nil)
```

- Pass the application service as the final argument when available.
- For post modules, pass the session (or the host form required by the current mixin) rather than assuming `rhost` exists.
- `store_loot` remains useful without a database and returns the saved path.
- `File.write` is acceptable only for a temporary local artifact needed by exploitation, not for user-facing loot. Place temporary data in a private temporary directory and remove it in `cleanup`/`ensure`.
- Never store secrets in console output when a credential/loot record is the safer representation, unless the module's established UX requires showing newly created credentials.

## Tables

Use `Rex::Text::Table` when multiple records need readable console output and optionally convert the same table to CSV for loot:

```ruby
table = Rex::Text::Table.new(
  'Header' => 'Acme Accounts',
  'Indent' => 1,
  'Columns' => %w[Username Role]
)

accounts.each { |account| table << [account[:username], account[:role]] }
print_line(table.to_s)

store_loot(
  'acme.accounts',
  'text/csv',
  rhost,
  table.to_csv,
  'acme-accounts.csv',
  'Acme App accounts',
  @acme_service
)
```

## Reporting QA

- Report the application during identification and ensure the exploit path reports it when AutoCheck is disabled.
- Query or inspect database output to verify application, HTTP/TLS, and TCP records form the expected hierarchy.
- Verify vulnerability, credential login, and loot point to the application service, not the transport.
- Run without a database and confirm reporting helpers do not become control-flow dependencies.
- Avoid duplicate service, credential, note, and vulnerability records on immediate reruns.

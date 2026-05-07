# Reporting & Loot Reference

## Mandatory Reporting Rule

**Every module that extracts data or discovers credentials MUST report its findings** using `store_loot` and/or `create_credential_login`. Modules that only `print_good` their findings without reporting to the database will be rejected in review.

- **NEVER use `File.write` or `File.open`** to save data — always use `store_loot`, which saves data to the proper loot directory and records it in the MSF database.
- `auxiliary/gather/` modules must call `store_loot` for extracted data.
- Modules that discover credentials must call `create_credential_login` (or the helper pattern below).
- `auxiliary/scanner/` modules that discover services must call `report_service`.

---

## store_loot

Saves exfiltrated data to disk and records it in the database.

```ruby
path = store_loot(ltype, ctype, host, data, filename, info, service)
```

| Parameter  | Type         | Required | Description                                                       |
| ---------- | ------------ | -------- | ----------------------------------------------------------------- |
| `ltype`    | String       | Yes      | OID-style loot type, e.g. `'webapp.emails'`, `'cisco.ios.config'` |
| `ctype`    | String       | Yes      | MIME content type. `text/*` → `.txt`, else `.bin`                 |
| `host`     | String       | Yes      | Target IP address or session object                               |
| `data`     | String       | Yes      | The file contents to save                                         |
| `filename` | String       | No       | Original filename (metadata only)                                 |
| `info`     | String       | No       | Descriptive text about the loot                                   |
| `service`  | Mdm::Service | No       | Service object to associate with                                  |

**Returns:** Full local file path where loot was saved.

### Usage pattern

```ruby
path = store_loot(
  'webapp.user.creds',
  'text/csv',
  rhost,
  cred_table.to_csv,
  'user_credentials.csv',
  'Extracted user credentials'
)
print_status("Credentials saved in: #{path}")
```

For plain text:

```ruby
store_loot('app.config', 'text/plain', rhost, config_data, 'app.conf', 'Application config')
```

For binary/JSON:

```ruby
store_loot('app.database', 'application/octet-stream', rhost, db_dump, 'dump.db')
store_loot('app.api.data', 'application/json', rhost, json_data, 'data.json')
```

---

## Credential Reporting

Include `Msf::Auxiliary::Report` (already included by default in auxiliary modules).

### Recommended helper pattern

Define a private helper method to keep `run` clean:

```ruby
def report_cred(opts)
  service_data = {
    address: opts[:ip],
    port: opts[:port],
    service_name: opts[:service_name],
    protocol: 'tcp',
    workspace_id: myworkspace_id
  }

  credential_data = {
    origin_type: :service,
    module_fullname: fullname,
    username: opts[:user],
    private_data: opts[:password],
    private_type: :password
  }.merge(service_data)

  login_data = {
    core: create_credential(credential_data),
    status: Metasploit::Model::Login::Status::UNTRIED,
    proof: opts[:proof]
  }.merge(service_data)
```

**Important**: When `status` is `SUCCESSFUL` (i.e., the module verified the credential), you **must** also include `last_attempted_at: DateTime.now`. Without it, `create_credential_login` raises `ActiveRecord::RecordInvalid` ("Last attempted at can't be nil if status is tried"):

```ruby
  login_data = {
    core: create_credential(credential_data),
    status: Metasploit::Model::Login::Status::SUCCESSFUL,
    last_attempted_at: DateTime.now,
    proof: opts[:proof]
  }.merge(service_data)
```

  create_credential_login(login_data)
end
```

Call from `run`:

```ruby
report_cred(
  ip: rhost,
  port: rport,
  service_name: (ssl ? 'https' : 'http'),
  user: username,
  password: password,
  proof: response_body
)
```

### create_credential keys

| Key               | Type    | Description                                                             |
| ----------------- | ------- | ----------------------------------------------------------------------- |
| `origin_type`     | Symbol  | `:service`, `:import`, `:manual`, `:session`                            |
| `module_fullname` | String  | Use `fullname` (auto-populated)                                         |
| `username`        | String  | The username                                                            |
| `private_data`    | String  | The credential value                                                    |
| `private_type`    | Symbol  | `:password`, `:ssh_key`, `:ntlm_hash`, `:nonreplayable_hash`, `:pkcs12` |
| `address`         | String  | Target IP                                                               |
| `port`            | Integer | Target port                                                             |
| `service_name`    | String  | Service name (`'http'`, `'ssh'`, etc.)                                  |
| `protocol`        | String  | `'tcp'` or `'udp'`                                                      |
| `workspace_id`    | Integer | Use `myworkspace_id`                                                    |
| `realm_key`       | String  | Optional domain/realm type                                              |
| `realm_value`     | String  | Optional domain/realm name                                              |
| `jtr_format`      | String  | Optional John the Ripper format hint                                    |

### create_credential_login keys

| Key                                                           | Type             | Description                                                                |
| ------------------------------------------------------------- | ---------------- | -------------------------------------------------------------------------- |
| `core`                                                        | Credential::Core | Return value from `create_credential`                                      |
| `status`                                                      | String           | `Metasploit::Model::Login::Status::UNTRIED`, `::SUCCESSFUL`, `::INCORRECT` |
| `access_level`                                                | String           | Optional (`'admin'`, `'user'`, etc.)                                       |
| `proof`                                                       | String           | Optional proof string (response body, cookie, etc.)                        |
| `address`, `port`, `service_name`, `protocol`, `workspace_id` | Various          | Same as above                                                              |

### create_credential_and_login (convenience combo)

Combines both calls — accepts all keys from both methods:

```ruby
create_credential_and_login(
  origin_type: :service,
  module_fullname: fullname,
  username: user,
  private_data: pass,
  private_type: :password,
  address: rhost,
  port: rport,
  service_name: 'http',
  protocol: 'tcp',
  workspace_id: myworkspace_id,
  status: Metasploit::Model::Login::Status::UNTRIED
)
```

---

## report_service

Record a discovered service in the database:

```ruby
report_service(
  host: rhost,
  port: rport,
  proto: 'tcp',
  name: 'http',
  info: 'Acme WebApp 4.2.1'
)
```

| Key      | Type    | Description                                  |
| -------- | ------- | -------------------------------------------- |
| `:host`  | String  | **Required.** IP address                     |
| `:port`  | Integer | **Required.** Port number                    |
| `:proto` | String  | `'tcp'` (default) or `'udp'`                 |
| `:name`  | String  | Service name (auto-downcased)                |
| `:info`  | String  | Version/banner info                          |
| `:state` | String  | `'open'` (default), `'closed'`, `'filtered'` |

---

## store_valid_credential (Msf::Module::Auth)

A simpler alternative to the full `create_credential_and_login` pattern. Uses keyword arguments. **Defined in `Msf::Module::Auth`** (auto-included in all modules). Stores credentials and associates them with a service in the database.

```ruby
store_valid_credential(
  user: 'admin',
  private: 'secret',
  private_type: :password,         # default; :ssh_key, :ntlm_hash also valid
  service_data: {
    address: rhost,
    port: rport,
    service_name: 'ssh',           # match the actual service, not the exploit transport
    protocol: 'tcp',
    workspace_id: myworkspace_id
  }
)
```

- If `service_data` is omitted and the module includes `HttpClient`, defaults to `service_details` (the HTTP service on `rhost:rport`). **Always pass explicit `service_data`** when the credential belongs to a different port/protocol (e.g., SSH on port 22 when the exploit went through HTTP on port 443).
- Reviewers will flag exploit modules that discover or set credentials without saving them.
- **When the module sets a known credential** (e.g., password change during exploitation), store the final persistent credential, not the intermediate one. If rotation fails, at minimum log a warning so the operator knows what credential is active.
- See `lib/msf/core/module/auth.rb` for the full implementation.

---

## Msf::Exploit::Retry — retry_until_truthy

For polling operations that require waiting on asynchronous target-side state, use `retry_until_truthy` instead of manual `N.times` + `Rex.sleep` loops. This mixin is in `lib/msf/core/exploit/retry.rb`.

```ruby
include Msf::Exploit::Retry

# Register an advanced option:
register_advanced_options([
  OptInt.new('OPERATION_TIMEOUT', [true, 'Seconds to wait for the operation to complete', 30]),
])

# Use in the module:
result = retry_until_truthy(timeout: datastore['OPERATION_TIMEOUT']) do
  res = send_request_cgi(...)
  res&.code == 200 && res.body.to_s.include?('expected_marker')
end
fail_with(Failure::Unknown, 'Operation did not complete within timeout') unless result
```

- Expose the timeout as an **advanced option** (not a regular option) so it doesn't clutter `show options` but remains tunable.
- The block must return a **truthy value** to stop retrying, or `false`/`nil` to keep trying.
- Uses exponential backoff internally — no manual `sleep` needed.
- Reference: `modules/exploits/linux/misc/cisco_ios_xe_rce.rb`

---

## report_vuln

Records a confirmed vulnerability in the MSF database. Call this when `check` (or `run`) has positively confirmed the vulnerability, so workspace operators know which hosts are affected.

```ruby
report_vuln(
  host: rhost,
  port: rport,
  proto: 'tcp',
  name: 'Vendor Product Unauthenticated RCE',
  info: 'Confirmed via out-of-band callback',
  refs: references
)
```

| Key      | Type    | Description                                                              |
| -------- | ------- | ------------------------------------------------------------------------ |
| `:host`  | String  | **Required.** Target IP address                                          |
| `:port`  | Integer | Port number                                                              |
| `:proto` | String  | `'tcp'` or `'udp'`                                                       |
| `:name`  | String  | Vulnerability name (human-readable)                                      |
| `:info`  | String  | Additional detail on how it was confirmed                                |
| `:refs`  | Array   | Pass `references` (auto-populated from module's `References` metadata)   |

**When to call it**: whenever your module confirms the vulnerability is present, not just that the service is running. Usually call it from `run`, `run_host`, or `exploit` once confirmation is complete. Calling it from `check` is acceptable when the module intentionally records confirmation there. Reviewers will flag gather/scanner modules that confirm vulnerabilities but don't call `report_vuln`.

---

## Rex::Text::Table

For formatted console output and CSV export of results.

### Construction

```ruby
tbl = Rex::Text::Table.new(
  'Header'  => 'Discovered Credentials',
  'Indent'  => 1,
  'Columns' => ['Username', 'Password', 'Admin', 'Email']
)
```

### Full options

| Key           | Type    | Default | Description                      |
| ------------- | ------- | ------- | -------------------------------- |
| `'Header'`    | String  | nil     | Table heading                    |
| `'Columns'`   | Array   | `[]`    | Column names                     |
| `'Rows'`      | Array   | `[]`    | Initial rows                     |
| `'Indent'`    | Integer | 0       | Left indent spaces               |
| `'CellPad'`   | Integer | 2       | Padding between columns          |
| `'Width'`     | Integer | 80      | Max table width                  |
| `'SortIndex'` | Integer | 0       | Column to sort by (-1 = no sort) |
| `'Prefix'`    | String  | `''`    | Text prepended before table      |
| `'Postfix'`   | String  | `''`    | Text appended after table        |

### Methods

| Method                 | Description                   |
| ---------------------- | ----------------------------- |
| `<< [val1, val2, ...]` | Add a row                     |
| `to_s`                 | Render formatted table string |
| `to_csv`               | Render as CSV                 |
| `print`                | Print to stdout               |

### Complete pattern (table + loot + creds)

```ruby
tbl = Rex::Text::Table.new(
  'Header'  => 'Found Credentials',
  'Indent'  => 1,
  'Columns' => ['Username', 'Password', 'Role']
)

credentials.each do |cred|
  tbl << [cred[:user], cred[:pass], cred[:role]]
  report_cred(
    ip: rhost,
    port: rport,
    service_name: (ssl ? 'https' : 'http'),
    user: cred[:user],
    password: cred[:pass],
    proof: cred[:proof]
  )
end

print_line
print_line(tbl.to_s)

path = store_loot(
  'app.user.creds',
  'text/csv',
  rhost,
  tbl.to_csv,
  'credentials.csv',
  'Extracted credentials'
)
print_status("Credentials saved in: #{path}")
```

# Reporting & Loot Reference

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

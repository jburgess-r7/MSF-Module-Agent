# HTTP Client Mixin Reference

## Setup

```ruby
include Msf::Exploit::Remote::HttpClient
```

This auto-registers: `RHOSTS`, `RPORT`, `VHOST`, `SSL`, `SSLCert`, `TARGETURI`, `HttpUsername`, `HttpPassword`, `UserAgent`, evasion options, and more.

**Do NOT re-register these** — adding `Opt::RHOST`, `Opt::RPORT(80)`, `OptString.new('VHOST', ...)`, or `OptBool.new('SSL', ...)` in `register_options` causes duplicates and will be rejected in review.

**Do NOT override `target_uri`** — the mixin defines this method. If you need the raw datastore value, use `datastore['TARGETURI']` directly.

---

## send_request_cgi(opts = {}, timeout = 20, disconnect = true)

Primary method for making HTTP requests. Returns `Rex::Proto::Http::Response` or `nil` on failure.

### Options hash

| Key                | Type   | Default   | Description                                                                                    |
| ------------------ | ------ | --------- | ---------------------------------------------------------------------------------------------- |
| `'method'`         | String | `'GET'`   | HTTP method (`GET`, `POST`, `PUT`, `DELETE`, etc.)                                             |
| `'uri'`            | String | `'/'`     | Request URI path                                                                               |
| `'vars_get'`       | Hash   | `{}`      | Query string parameters `{ 'key' => 'val' }`                                                   |
| `'vars_post'`      | Hash   | `{}`      | POST form parameters `{ 'key' => 'val' }`                                                      |
| `'data'`           | String | `''`      | Raw POST body (use instead of `vars_post` for JSON/XML/raw)                                    |
| `'ctype'`          | String | auto      | Content-Type header. Auto-set to `application/x-www-form-urlencoded` for POST with `vars_post` |
| `'headers'`        | Hash   | `{}`      | Additional HTTP headers `{ 'X-Custom' => 'value' }`                                            |
| `'cookie'`         | String | nil       | Cookie header value `'key=val; key2=val2'`                                                     |
| `'agent'`          | String | datastore | User-Agent header                                                                              |
| `'vhost'`          | String | nil       | Host header override                                                                           |
| `'keep_cookies'`   | Bool   | false     | Auto-store `Set-Cookie` responses in `cookie_jar`                                              |
| `'vars_form_data'` | Array  | `[]`      | Multipart form fields                                                                          |
| `'encode_params'`  | Bool   | true      | URI-encode parameter names and values                                                          |
| `'version'`        | String | `'1.1'`   | HTTP version                                                                                   |

### Common patterns

**GET with query parameters:**

```ruby
res = send_request_cgi(
  'method' => 'GET',
  'uri'    => normalize_uri(target_uri.path, 'api', 'users'),
  'vars_get' => {
    'page'  => '1',
    'limit' => '100'
  }
)
```

**POST with form data:**

```ruby
res = send_request_cgi(
  'method'    => 'POST',
  'uri'       => normalize_uri(target_uri.path, 'login'),
  'vars_post' => {
    'username' => datastore['USERNAME'],
    'password' => datastore['PASSWORD']
  }
)
```

**POST with JSON body:**

```ruby
res = send_request_cgi(
  'method' => 'POST',
  'uri'    => normalize_uri(target_uri.path, 'api', 'auth'),
  'ctype'  => 'application/json',
  'data'   => { username: user, password: pass }.to_json
)
```

**POST with XML/SOAP body:**

When constructing SOAP envelopes that embed user-supplied values (usernames, passwords, search terms), **always XML-escape** the values to prevent XML injection. Define a private helper:

```ruby
def xml_escape(str)
  str.to_s.gsub('&', '&amp;').gsub('<', '&lt;').gsub('>', '&gt;').gsub('"', '&quot;').gsub("'", '&apos;')
end
```

SOAP 1.2 uses `application/soap+xml`; SOAP 1.1 uses `text/xml`. Match the target service's protocol version.

```ruby
# SOAP 1.2 example (application/soap+xml)
body = '<AuthRequest xmlns="urn:vendorAuth">' \
       "<account by=\"name\">#{xml_escape(datastore['USERNAME'])}</account>" \
       "<password>#{xml_escape(datastore['PASSWORD'])}</password>" \
       '</AuthRequest>'

envelope = '<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">' \
           "<soap:Body>#{body}</soap:Body>" \
           '</soap:Envelope>'

res = send_request_cgi(
  'method' => 'POST',
  'uri'    => normalize_uri(target_uri.path, 'service', 'soap'),
  'ctype'  => 'application/soap+xml; charset=UTF-8',
  'data'   => envelope
)
```

**Request with custom headers and cookie:**

```ruby
res = send_request_cgi(
  'method'  => 'GET',
  'uri'     => normalize_uri(target_uri.path, 'admin', 'config'),
  'cookie'  => "SESSION_TOKEN=#{@auth_token}",
  'headers' => {
    'X-CSRF-Token' => @csrf_token,
    'Accept' => 'application/json'
  }
)
```

---

## send_request_cgi!(opts = {}, timeout = 20, redirect_depth = 1)

Same as `send_request_cgi` but **auto-follows redirects** (301/302/303/307/308). Use when login endpoints redirect after auth.

### Timeout for payload-triggering requests

When the request that triggers a payload (e.g., RCE) will hang or never return, set `timeout: 0` so the framework doesn't wait:

```ruby
# The exploit-triggering request — server won't respond
send_request_cgi(
  'method' => 'POST',
  'uri'    => normalize_uri(target_uri.path, 'vulnerable'),
  'data'   => payload_data
), 0  # timeout = 0, don't wait for response
```

For session-based exploits, rely on `WfsDelay` (Wait For Session Delay) — it's already a registered option. Don't create a custom `WAIT` option that duplicates it.

```ruby
res = send_request_cgi!(
  { 'method' => 'POST', 'uri' => '/login', 'vars_post' => creds },
  20,  # timeout
  3    # follow up to 3 redirects
)
```

---

## send_request_raw(opts = {}, timeout = 20, disconnect = false)

Low-level request without CGI processing. Does not handle `vars_get`/`vars_post` — you build the full request manually. Use for protocol-level edge cases.

---

## Response Object (Rex::Proto::Http::Response)

**Always check for nil** — connection failures return `nil`:

```ruby
res = send_request_cgi(...)
fail_with(Failure::Unreachable, 'Connection failed') unless res
```

**Body can be nil** — always guard with `.to_s` when pattern matching:

```ruby
# WRONG — raises NoMethodError if body is nil
res.body.include?('success')

# CORRECT — safe even when body is nil
res.body.to_s.include?('success')
```

### Attributes

| Method    | Type      | Description                                                       |
| --------- | --------- | ----------------------------------------------------------------- |
| `code`    | Integer   | HTTP status code (200, 301, 404, 500...)                          |
| `message` | String    | Status reason phrase ("OK", "Not Found")                          |
| `body`    | String    | Response body                                                     |
| `headers` | Hash-like | Response headers (`headers['Location']`, `headers['Set-Cookie']`) |

### Parsing methods

| Method               | Returns        | Description                                                                                       |
| -------------------- | -------------- | ------------------------------------------------------------------------------------------------- |
| `get_json_document`  | Hash           | `JSON.parse(body)`, returns `{}` on parse error — **always prefer this over manual `JSON.parse`** |
| `get_html_document`  | Nokogiri::HTML | Parsed HTML document                                                                              |
| `get_xml_document`   | Nokogiri::XML  | Parsed XML document                                                                               |
| `get_cookies`        | String         | Parsed Set-Cookie as `"k=v; k2=v2"` for reuse                                                     |
| `get_cookies_parsed` | Hash           | Parsed cookie key-value pairs                                                                     |
| `get_hidden_inputs`  | Array\<Hash\>  | Hidden form inputs per form                                                                       |
| `redirect?`          | Boolean        | True if response is a redirect                                                                    |
| `redirection`        | URI            | Location header as URI                                                                            |

### Response handling patterns

```ruby
# Basic nil + status check
res = send_request_cgi('uri' => '/api/data')
fail_with(Failure::Unreachable, 'Connection failed') unless res
fail_with(Failure::UnexpectedReply, "Unexpected status: #{res.code}") unless res.code == 200

# JSON response — ALWAYS use get_json_document, never manual JSON.parse
json = res.get_json_document
fail_with(Failure::UnexpectedReply, 'Invalid JSON response') if json.empty?
token = json['authToken'] || fail_with(Failure::UnexpectedReply, 'No auth token in response')

# XML/SOAP response — use get_xml_document or Nokogiri::XML directly
xml = res.get_xml_document
token = xml.at_xpath('//authToken')&.text
fail_with(Failure::UnexpectedReply, 'Auth token not found in SOAP response') unless token

# For SOAP responses with multiple namespaces, remove_namespaces! simplifies XPath:
doc = Nokogiri::XML(res.body.to_s)
doc.remove_namespaces!
subject = doc.at_xpath('//Subject')&.text
items = doc.xpath('//ItemId').map { |n| n['Id'] }

# HTML scraping
html = res.get_html_document
csrf = html.at_css('input[name="csrf_token"]')&.attr('value')

# Cookie extraction
cookies = res.get_cookies
# Use in subsequent request:
send_request_cgi('uri' => '/dashboard', 'cookie' => cookies)
```

---

## Cookie Handling

### Automatic (recommended for multi-step flows)

When `keep_cookies: true` is set, the cookie jar automatically stores and replays cookies — **do NOT manually extract and replay cookies alongside `keep_cookies`** (this causes duplicate cookies).

```ruby
# First request — store cookies automatically
res = send_request_cgi(
  'uri' => '/login',
  'method' => 'POST',
  'vars_post' => { 'user' => u, 'pass' => p },
  'keep_cookies' => true
)

# Subsequent requests automatically include stored cookies
res = send_request_cgi(
  'uri' => '/dashboard',
  'keep_cookies' => true
)
```

### Manual

```ruby
@auth_cookie = res.get_cookies
# ...
res = send_request_cgi('uri' => '/api', 'cookie' => @auth_cookie)
```

### Cookie jar methods

| Method               | Description            |
| -------------------- | ---------------------- |
| `cookie_jar.cookies` | All stored cookies     |
| `cookie_jar.clear`   | Remove all cookies     |
| `cookie_jar.empty?`  | Check if jar is empty  |
| `cookie_jar.cleanup` | Remove expired cookies |

---

## URI Helpers

### normalize_uri(\*segments)

Joins path segments, collapses double slashes, ensures leading slash:

```ruby
normalize_uri(target_uri.path, 'service', 'soap')
# => "/service/soap" (if TARGETURI is "/")
# => "/webapp/service/soap" (if TARGETURI is "/webapp")
```

### target_uri

Returns a `URI` object from `datastore['TARGETURI']`:

```ruby
target_uri.path  # => "/webapp"
```

### full_uri(path = nil)

Returns the full URL string:

```ruby
full_uri('/api/v1')  # => "https://target:443/api/v1"
```

---

## Authentication Helpers

### Basic auth

```ruby
# Auto-populated from HttpUsername/HttpPassword datastore options
# OR set manually:
send_request_cgi(
  'uri' => '/api/secure',
  'headers' => {
    'Authorization' => basic_auth(user, pass)
  }
)
```

---

## SSL

SSL is handled automatically via the `SSL` and `RPORT` datastore options. Common setup:

```ruby
'DefaultOptions' => {
  'RPORT' => 443,
  'SSL'   => true
}
```

Check at runtime with `ssl` (boolean) or `datastore['SSL']`.

---

## Common Datastore Accessors

| Method                      | Description                        |
| --------------------------- | ---------------------------------- |
| `rhost`                     | Remote host (`RHOSTS` first value) |
| `rport`                     | Remote port                        |
| `ssl`                       | Whether SSL is enabled (boolean)   |
| `vhost`                     | Virtual host                       |
| `datastore['TARGETURI']`    | Target base URI                    |
| `datastore['USERNAME']`     | Custom option value                |
| `datastore['HttpUsername']` | HTTP basic auth user               |
| `datastore['HttpPassword']` | HTTP basic auth pass               |

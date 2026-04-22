# Non-HTTP Module Patterns

Reference for writing MSF modules that don't use the HttpClient mixin: scanners, TCP/UDP services, local exploits, and post-exploitation modules.

---

## Scanner Modules (`Msf::Auxiliary::Scanner`)

The Scanner mixin handles RHOSTS iteration, threading, and progress display. You implement `run_host(ip)` instead of `run`.

### Key rules

- `include Msf::Auxiliary::Scanner` **after** protocol mixins (order matters)
- Implement `def run_host(ip)` — called once per host in RHOSTS
- Do NOT define `def run` — the Scanner mixin provides it
- `RHOSTS` and `THREADS` are auto-registered
- Use `peer` for log prefixing (returns `"host:port"`)

### Scanner template

```ruby
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient  # or Tcp, Ftp, SMB, etc.
  include Msf::Auxiliary::Scanner            # MUST come after protocol mixins
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Product Version Scanner',
        'Description' => %q{
          Scans for Vendor Product instances and fingerprints the version.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )

    register_options([
      Opt::RPORT(8080)
    ])
  end

  def run_host(ip)
    res = send_request_cgi('uri' => normalize_uri(target_uri.path))
    return unless res

    if res.body =~ /Version: (\d+\.\d+\.\d+)/
      version = Regexp.last_match(1)
      print_good("#{peer} - Detected version #{version}")
      report_service(host: ip, port: rport, proto: 'tcp', name: 'http', info: "Product #{version}")
    else
      vprint_status("#{peer} - Service detected but version unknown")
    end
  end
end
```

### Batch scanner (alternative)

For protocols where per-host connections are expensive, implement `run_batch` + `run_batch_size` instead of `run_host`:

```ruby
def run_batch_size
  50
end

def run_batch(hosts)
  # hosts is an array of IP strings
  hosts.each do |ip|
    # process batch
  end
end
```

---

## TCP-Based Modules (`Msf::Exploit::Remote::Tcp`)

For raw TCP services without a dedicated protocol mixin.

### Key rules

- `include Msf::Exploit::Remote::Tcp` — auto-registers `RHOSTS`, `RPORT`, `SSL`, `ConnectTimeout`
- Use `connect` to open a socket, `disconnect` to close
- Use `sock.put(data)` to send, `sock.get_once(length, timeout)` to receive
- Always handle connection failures

### TCP module pattern

```ruby
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::Tcp
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Service Banner Grab',
        'Description' => %q{
          Connects to the target service and grabs the banner.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )

    register_options([
      Opt::RPORT(9090)
    ])
  end

  def run
    connect
    banner = sock.get_once(1024, 10)
    disconnect

    if banner && !banner.empty?
      print_good("Banner: #{banner.strip}")
      report_service(host: rhost, port: rport, proto: 'tcp', name: 'vendor_svc', info: banner.strip)
    else
      print_error('No banner received')
    end
  rescue Rex::ConnectionError => e
    fail_with(Failure::Unreachable, e.message)
  end
end
```

---

## FTP Modules (`Msf::Exploit::Remote::Ftp`)

### Key mixins and methods

```ruby
include Msf::Exploit::Remote::Ftp
```

Auto-registers: `RPORT` (21), `FTPUSER`, `FTPPASS`, `FTPTimeout`, `FTPDEBUG`.

| Method                                  | Description                                 |
| --------------------------------------- | ------------------------------------------- |
| `connect_login`                         | Connect + authenticate (returns true/false) |
| `connect`                               | Open FTP control connection                 |
| `send_cmd(args, recv = true)`           | Send an FTP command                         |
| `send_cmd_data(args, data, mode = 'a')` | Send data via a data channel                |
| `data_connect(mode = 'a')`              | Open a data channel                         |
| `disconnect`                            | Close connections                           |

---

## SMB Modules (`Msf::Exploit::Remote::SMB`)

```ruby
include Msf::Exploit::Remote::DCERPC
include Msf::Exploit::Remote::SMB::Client
```

Auto-registers: `RPORT` (445), `SMBUser`, `SMBPass`, `SMBDomain`, `SMB::ProtocolVersion`.

Common methods: `connect`, `smb_login`, `smb_create(filename)`, `smb_write`.

---

## SSH Modules

SSH modules commonly use `Net::SSH` via the framework's SSH mixin:

```ruby
include Msf::Exploit::Remote::SSH
```

For scanner-style SSH modules, look at existing modules in `modules/auxiliary/scanner/ssh/`.

---

## Local Exploit Modules (`Msf::Exploit::Local`)

For privilege escalation and post-exploitation exploits that require an existing session.

### Key rules

- Inherit from `Msf::Exploit::Local` (not `Msf::Exploit::Remote`)
- **MUST** specify `'SessionTypes'` in metadata (e.g., `['shell', 'meterpreter']`)
- Use `prepend Msf::Exploit::Remote::AutoCheck` — always prepend, never include
- Common mixins: `Msf::Post::File`, `Msf::Post::Linux::Priv`, `Msf::Post::Linux::Kernel`, `Msf::Exploit::EXE`, `Msf::Exploit::FileDropper`
- Use `session` object for all target interaction
- Use `cmd_exec(command)` to run commands on target
- Use `write_file(path, data)` / `read_file(path)` from `Msf::Post::File`

### Local exploit template

```ruby
class MetasploitModule < Msf::Exploit::Local
  Rank = ExcellentRanking

  include Msf::Post::File
  include Msf::Post::Linux::Priv
  include Msf::Exploit::EXE
  include Msf::Exploit::FileDropper

  prepend Msf::Exploit::Remote::AutoCheck

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Product Local Privilege Escalation',
        'Description' => %q{
          Description of the local vulnerability and how it achieves
          privilege escalation.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'References' => [
          ['CVE', 'YYYY-NNNNN']
        ],
        'DisclosureDate' => '2026-01-15',
        'Platform' => ['unix', 'linux'],
        'SessionTypes' => ['shell', 'meterpreter'],
        'Arch' => [ARCH_X64, ARCH_CMD],
        'Targets' => [
          ['Automatic', {}]
        ],
        'DefaultTarget' => 0,
        'DefaultOptions' => {
          'PAYLOAD' => 'linux/x64/meterpreter/reverse_tcp'
        },
        'Privileged' => true,
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [REPEATABLE_SESSION],
          'SideEffects' => [ARTIFACTS_ON_DISK]
        }
      )
    )
  end

  def check
    output = cmd_exec('cat /etc/product_version')
    if output =~ /(\d+\.\d+)/
      version = Rex::Version.new(Regexp.last_match(1))
      if version < Rex::Version.new('2.0')
        return CheckCode::Appears("Version #{version} is vulnerable")
      end
    end
    CheckCode::Safe
  end

  def exploit
    unless is_root?
      fail_with(Failure::NoAccess, 'Must run as root or with sudo')
    end

    payload_path = '/tmp/.payload'
    write_file(payload_path, generate_payload_exe)
    register_file_for_cleanup(payload_path)
    cmd_exec("chmod +x #{payload_path}")
    cmd_exec(payload_path)
  end
end
```

### AutoCheck (`prepend Msf::Exploit::Remote::AutoCheck`)

- **Always use `prepend`**, never `include` — it wraps `exploit`/`run` to call `check` first
- Auto-registers `AutoCheck` (default: true) and `ForceExploit` (default: false) advanced options
- If `check` returns `Safe`, the exploit aborts unless `ForceExploit` is set
- If `check` returns `Appears` or `Vulnerable`, exploitation proceeds

### FileDropper (`Msf::Exploit::FileDropper`)

Tracks files/dirs created on target and cleans them up after exploitation:

```ruby
include Msf::Exploit::FileDropper

# In exploit method:
register_file_for_cleanup('/tmp/payload.bin')
register_dir_for_cleanup('/tmp/exploit_dir')
```

### Cleanup Method

For custom cleanup logic (removing artifacts, restoring state), define a `cleanup` method. **Always call `super`** to ensure framework cleanup (including FileDropper) runs:

```ruby
def cleanup
  # Custom cleanup logic here
  remove_payload if @payload_deployed
ensure
  super
end
```

Never put cleanup logic at the end of `exploit`/`run` — if the method raises an exception, that code won't execute. The `cleanup` method is called automatically regardless of success/failure.

---

## Post-Exploitation Modules (`Msf::Post`)

For modules that run after initial access via an existing session.

### Key rules

- Inherit from `Msf::Post`
- **MUST** specify `'Platform'` and `'SessionTypes'`
- `'SessionTypes'` can be: `'shell'`, `'meterpreter'`, `'powershell'`
- Use `session` to interact: `session.type`, `session.platform`, `session.session_host`
- Common mixins: `Msf::Post::File`, `Msf::Post::Windows::Registry`, `Msf::Post::Linux::Priv`
- Use `sysinfo` for system information (meterpreter sessions)
- Use `cmd_exec(command)` to run system commands

### Post module template

```ruby
class MetasploitModule < Msf::Post
  include Msf::Post::File
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Multi Gather Application Credentials',
        'Description' => %q{
          Extracts saved credentials from Application X config files.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'Platform' => %w[linux win],
        'SessionTypes' => %w[shell meterpreter],
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => []
        }
      )
    )
  end

  def run
    hostname = sysinfo.nil? ? cmd_exec('hostname') : sysinfo['Computer']
    print_status("Running on #{hostname} (#{session.session_host})")

    config = read_file('/etc/app/config.yml')
    if config.nil? || config.empty?
      print_error('Config file not found')
      return
    end

    print_good('Found config file, extracting credentials...')
    path = store_loot(
      'app.config',
      'text/plain',
      session,
      config,
      'config.yml',
      'Application config file'
    )
    print_good("Config saved to: #{path}")
  end
end
```

### Post module mixins reference

| Mixin                          | Purpose                                                             |
| ------------------------------ | ------------------------------------------------------------------- |
| `Msf::Post::File`              | `read_file`, `write_file`, `file_exist?`, `directory?`, `readable?` |
| `Msf::Post::Linux::Priv`       | `is_root?`, `id`, `whoami`                                          |
| `Msf::Post::Linux::Kernel`     | `uname`, `kernel_release`, `kernel_modules`                         |
| `Msf::Post::Windows::Registry` | `registry_getvaldata`, `registry_enumkeys`, etc.                    |
| `Msf::Post::Windows::Priv`     | `is_admin?`, `is_system?`, `getsystem`                              |
| `Msf::Post::Common`            | `cmd_exec(command)`, `get_env(name)`                                |
| `Msf::Post::Unix`              | `get_users`, `get_groups`, `enum_user_directories`                  |

---

## CheckCode Constants

Used in `def check` for exploit and scanner modules:

| Constant                 | Code            | When to return                                          |
| ------------------------ | --------------- | ------------------------------------------------------- |
| `CheckCode::Unknown`     | `'unknown'`     | Cannot determine vulnerability status (timeout, error)  |
| `CheckCode::Safe`        | `'safe'`        | Target is NOT vulnerable (patched, not running service) |
| `CheckCode::Detected`    | `'detected'`    | Service is running but vuln status unknown              |
| `CheckCode::Appears`     | `'appears'`     | Likely vulnerable (version/banner-based)                |
| `CheckCode::Vulnerable`  | `'vulnerable'`  | Confirmed vulnerable (active verification)              |
| `CheckCode::Unsupported` | `'unsupported'` | Module does not support check                           |

CheckCode accepts an optional reason string:

```ruby
CheckCode::Appears("Version #{version} is in the vulnerable range")
CheckCode::Safe('Target is running patched version 2.1.0')
```

---

## Multipart File Upload (`vars_form_data`)

For HTTP modules that upload files via multipart/form-data:

```ruby
res = send_request_cgi(
  'method' => 'POST',
  'uri' => normalize_uri(target_uri.path, 'upload'),
  'vars_form_data' => [
    {
      'name' => 'file',
      'filename' => 'shell.php',
      'content_type' => 'application/octet-stream',
      'data' => payload_content,
      'encoding' => 'binary'
    },
    {
      'name' => 'description',
      'data' => 'Uploaded file'
    }
  ]
)
```

### `vars_form_data` field keys

| Key              | Required | Description                               |
| ---------------- | -------- | ----------------------------------------- |
| `'name'`         | Yes      | Form field name                           |
| `'data'`         | Yes      | Field value or file content               |
| `'filename'`     | No       | Filename (presence makes it a file field) |
| `'content_type'` | No       | MIME type for file fields                 |
| `'encoding'`     | No       | `'binary'` for binary content             |

---

## UDP-Based Modules with Native DTLS (`Msf::Exploit::Remote::Udp` + `Fiddle`)

Ruby's standard library has no DTLS support. When a target service requires DTLS 1.2 (or DTLS 1.0), you must drive OpenSSL's C API directly via `require 'fiddle'` and memory BIOs.

This is the established pattern for vdaemon-family (Cisco SD-WAN) modules and any other UDP service requiring DTLS.

### Key rules

- `include Msf::Exploit::Remote::Udp` — registers `RHOSTS`, `RPORT`, `UDP::Timeout`. Use `connect_udp`/`disconnect_udp` for the raw UDP socket; `udp_sock.sendto` / `udp_sock.recvfrom` for I/O
- `require 'fiddle'` — the only non-autoloaded standard library needed
- All DTLS work is done through in-memory BIOs: the Ruby code writes/reads raw UDP frames by hand and feeds them through `BIO_write`/`BIO_read` while `SSL_connect`/`SSL_read`/`SSL_write` operate on the SSL object
- Always call `require 'fiddle'` at the top of the class file (just before or after the class declaration)
- Always wrap `load_openssl_ffi` and DTLS teardown in an `ensure` block so sockets and SSL objects are freed even if `run` raises

### DTLS architecture

```
Ruby (MSF)                        OpenSSL C library
──────────────────────────────    ─────────────────────────
udp_sock.sendto(frame) ──────►  BIO_write(wbio, frame)
                                  SSL_connect / SSL_read / SSL_write
udp_sock.recvfrom    ◄────────  BIO_read(rbio)
```

Two `BIO_s_mem()` (in-memory) BIOs are created — one for reading (`rbio`, data coming from the network) and one for writing (`wbio`, data going to the network). After every `SSL_read`/`SSL_write`/`SSL_connect` call you must drain the `wbio` and send any accumulated bytes over the UDP socket, and before every SSL call you must write any pending UDP bytes into `rbio`.

### Loading the OpenSSL shared library

```ruby
require 'fiddle'

LIBSSL_NAMES = {
  'windows' => ['libssl-3-x64.dll', 'libssl-1_1-x64.dll'],
  'osx'     => ['libssl.dylib', '/usr/local/opt/openssl/lib/libssl.dylib'],
  'linux'   => ['libssl.so', 'libssl.so.3', 'libssl.so.1.1']
}.freeze

LIBCRYPTO_NAMES = {
  'windows' => ['libcrypto-3-x64.dll', 'libcrypto-1_1-x64.dll'],
  'osx'     => ['libcrypto.dylib', '/usr/local/opt/openssl/lib/libcrypto.dylib'],
  'linux'   => ['libcrypto.so', 'libcrypto.so.3', 'libcrypto.so.1.1']
}.freeze

def load_native_lib(names)
  names.each do |name|
    return Fiddle.dlopen(name)
  rescue Fiddle::DLError
    next
  end
  nil
end

def load_openssl_ffi
  return if @ffi_loaded

  platform = if RUBY_PLATFORM =~ /mingw|mswin/
               'windows'
             elsif RUBY_PLATFORM =~ /darwin/
               'osx'
             else
               'linux'
             end

  libssl    = load_native_lib(LIBSSL_NAMES[platform])
  libcrypto = load_native_lib(LIBCRYPTO_NAMES[platform])

  unless libssl && libcrypto
    fail_with(Failure::NotFound, 'Could not load libssl/libcrypto — DTLS not supported on this platform')
  end

  bind = ->(lib, sym, args, ret) { Fiddle::Function.new(lib[sym], args, ret) }

  # SSL_CTX / SSL lifecycle
  @f_dtls_client_method  = bind.(libssl, 'DTLS_client_method',            [], Fiddle::TYPE_VOIDP)
  @f_ssl_ctx_new         = bind.(libssl, 'SSL_CTX_new',                   [Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOIDP)
  @f_ssl_ctx_set_verify  = bind.(libssl, 'SSL_CTX_set_verify',            [Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT, Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOID)
  @f_ssl_ctx_use_cert    = bind.(libssl, 'SSL_CTX_use_certificate_ASN1',  [Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT, Fiddle::TYPE_VOIDP], Fiddle::TYPE_INT)
  @f_ssl_ctx_use_pkey    = bind.(libssl, 'SSL_CTX_use_PrivateKey_ASN1',   [Fiddle::TYPE_INT, Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_LONG], Fiddle::TYPE_INT)
  @f_ssl_new             = bind.(libssl, 'SSL_new',                       [Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOIDP)
  @f_ssl_set_bio         = bind.(libssl, 'SSL_set_bio',                   [Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOID)
  @f_ssl_connect         = bind.(libssl, 'SSL_connect',                   [Fiddle::TYPE_VOIDP], Fiddle::TYPE_INT)
  @f_ssl_read            = bind.(libssl, 'SSL_read',                      [Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT], Fiddle::TYPE_INT)
  @f_ssl_write           = bind.(libssl, 'SSL_write',                     [Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT], Fiddle::TYPE_INT)
  @f_ssl_get_error       = bind.(libssl, 'SSL_get_error',                 [Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT], Fiddle::TYPE_INT)
  @f_ssl_free            = bind.(libssl, 'SSL_free',                      [Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOID)
  @f_ssl_ctx_free        = bind.(libssl, 'SSL_CTX_free',                  [Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOID)
  @f_ssl_ctrl            = bind.(libssl, 'SSL_ctrl',                      [Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT, Fiddle::TYPE_LONG, Fiddle::TYPE_VOIDP], Fiddle::TYPE_LONG)

  # BIO / memory buffer
  @f_bio_new             = bind.(libcrypto, 'BIO_new',   [Fiddle::TYPE_VOIDP], Fiddle::TYPE_VOIDP)
  @f_bio_s_mem           = bind.(libcrypto, 'BIO_s_mem', [], Fiddle::TYPE_VOIDP)
  @f_bio_read            = bind.(libcrypto, 'BIO_read',  [Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT], Fiddle::TYPE_INT)
  @f_bio_write           = bind.(libcrypto, 'BIO_write', [Fiddle::TYPE_VOIDP, Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT], Fiddle::TYPE_INT)
  @f_bio_ctrl            = bind.(libcrypto, 'BIO_ctrl',  [Fiddle::TYPE_VOIDP, Fiddle::TYPE_INT, Fiddle::TYPE_LONG, Fiddle::TYPE_VOIDP], Fiddle::TYPE_LONG)

  @ffi_loaded = true
end
```

### DTLS handshake state machine

```ruby
SSL_ERROR_WANT_READ  = 2
SSL_ERROR_WANT_WRITE = 3
DTLS_CTRL_SET_LINK_MTU = 120   # avoids IP fragmentation

def dtls_handshake(cert_der, pkey_der)
  # 1. Build SSL context with self-signed cert (server won't verify us)
  method  = @f_dtls_client_method.call
  ctx     = @f_ssl_ctx_new.call(method)
  @f_ssl_ctx_set_verify.call(ctx, 0, 0)              # SSL_VERIFY_NONE
  @f_ssl_ctx_use_cert.call(ctx, cert_der.bytesize, Fiddle::Pointer[cert_der])
  @f_ssl_ctx_use_pkey.call(6, ctx,                   # 6 = EVP_PKEY_RSA
                            Fiddle::Pointer[pkey_der], pkey_der.bytesize)

  # 2. Create in-memory BIOs
  rbio = @f_bio_new.call(@f_bio_s_mem.call)
  wbio = @f_bio_new.call(@f_bio_s_mem.call)
  ssl  = @f_ssl_new.call(ctx)
  @f_ssl_set_bio.call(ssl, rbio, wbio)
  @f_ssl_ctrl.call(ssl, DTLS_CTRL_SET_LINK_MTU, 1200, 0)

  @ssl_ptr = ssl
  @ctx_ptr = ctx

  # 3. Drive handshake (WANT_READ loop)
  loop do
    rc = @f_ssl_connect.call(ssl)
    flush_wbio_to_udp(wbio)           # always drain output BIO
    break if rc == 1                  # handshake complete

    err = @f_ssl_get_error.call(ssl, rc)
    unless [SSL_ERROR_WANT_READ, SSL_ERROR_WANT_WRITE].include?(err)
      fail_with(Failure::UnexpectedReply, "DTLS handshake failed (SSL_get_error=#{err})")
    end

    # Feed next UDP datagram into rbio
    frame = udp_recv_frame
    fail_with(Failure::Unreachable, 'DTLS handshake: no data from server') unless frame

    @f_bio_write.call(rbio, Fiddle::Pointer[frame], frame.bytesize)
  end
end
```

### Send / receive helpers

```ruby
MAX_UDP = 65_507

def dtls_send(plaintext)
  @f_ssl_write.call(@ssl_ptr, Fiddle::Pointer[plaintext], plaintext.bytesize)
  flush_wbio_to_udp(@wbio)
end

def dtls_recv(timeout: 5)
  deadline = Time.now + timeout
  loop do
    frame = udp_recv_frame(timeout: [deadline - Time.now, 0.1].max)
    return nil unless frame

    @f_bio_write.call(@rbio, Fiddle::Pointer[frame], frame.bytesize)
    buf = Fiddle::Pointer.malloc(MAX_UDP)
    n   = @f_ssl_read.call(@ssl_ptr, buf, MAX_UDP)
    next if n <= 0

    return buf[0, n]
  end
end

def flush_wbio_to_udp(wbio)
  loop do
    buf = Fiddle::Pointer.malloc(MAX_UDP)
    n   = @f_bio_read.call(wbio, buf, MAX_UDP)
    break if n <= 0

    udp_sock.sendto(buf[0, n], 0, rhost, rport)
  end
end

def udp_recv_frame(timeout: 5)
  ready = IO.select([udp_sock.fd], nil, nil, timeout)
  return nil unless ready

  data, = udp_sock.recvfrom(MAX_UDP)
  data
end
```

### Cleanup

Always free SSL objects in an `ensure` block — SSL_free also frees the BIOs:

```ruby
def cleanup_dtls
  @f_ssl_free.call(@ssl_ptr) if @ssl_ptr && @ffi_loaded
  @f_ssl_ctx_free.call(@ctx_ptr) if @ctx_ptr && @ffi_loaded
rescue StandardError
  nil
ensure
  disconnect_udp rescue nil
end
```

Call `cleanup_dtls` from an `ensure` in `perform_*` or `run`:

```ruby
def run
  connect_udp
  load_openssl_ffi
  perform_exploit
ensure
  cleanup_dtls
end
```

### Generating a self-signed certificate for the DTLS handshake

Many services accept any client certificate during DTLS handshake — they only authenticate at the application layer. Generate a fresh RSA self-signed cert for each session:

```ruby
require 'openssl'   # already available in MSF Ruby runtime

def generate_self_signed_cert
  key  = OpenSSL::PKey::RSA.generate(2048)
  cert = OpenSSL::X509::Certificate.new
  cert.version   = 2
  cert.serial    = rand(0xffffffff)
  cert.subject   = OpenSSL::X509::Name.parse('/CN=unknown')
  cert.issuer    = cert.subject
  cert.public_key = key.public_key
  now = Time.now
  cert.not_before = now - 60
  cert.not_after  = now + 365 * 24 * 3600
  cert.sign(key, OpenSSL::Digest::SHA256.new)
  [cert.to_der, key.to_der]   # both in DER (ASN.1) for the C API
end
```

### Module skeleton

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

require 'fiddle'

class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::Udp
  include Msf::Auxiliary::Report

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'Vendor Product DTLS Authentication Bypass',
        'Description' => %q{
          Exploits a flaw in the vdaemon DTLS authentication handshake.
        },
        'Author' => ['Author Name'],
        'License' => MSF_LICENSE,
        'References' => [
          ['CVE', 'YYYY-NNNNN']
        ],
        'DisclosureDate' => 'YYYY-MM-DD',
        'Notes' => {
          'Stability' => [CRASH_SAFE],
          'Reliability' => [],
          'SideEffects' => [ARTIFACTS_ON_DISK, IOC_IN_LOGS]
        }
      )
    )

    register_options([
      Opt::RPORT(12346)
    ])
  end

  def check
    perform_bypass(inject: false)
    CheckCode::Vulnerable('Bypass succeeded')
  rescue StandardError => e
    CheckCode::Unknown(e.message)
  end

  def run
    connect_udp
    load_openssl_ffi
    perform_bypass(inject: true)
  ensure
    cleanup_dtls
  end

  private

  def perform_bypass(inject:)
    cert_der, pkey_der = generate_self_signed_cert
    dtls_handshake(cert_der, pkey_der)
    # ... application-layer exchange ...
  end

  # include all helpers: load_openssl_ffi, dtls_handshake, dtls_send,
  # dtls_recv, flush_wbio_to_udp, udp_recv_frame, cleanup_dtls,
  # generate_self_signed_cert
end
```

---

## Verbose Output

Use `vprint_*` methods for output only shown when `VERBOSE` is true:

```ruby
vprint_status('Attempting connection...')    # Informational
vprint_good('Token obtained')               # Success
vprint_warning('Retrying request...')       # Warning
vprint_error('Unexpected response code')    # Error (non-fatal)
```

These are identical to `print_*` but suppressed unless `datastore['VERBOSE']` is true. Use them for detailed debugging info that would be noisy in normal operation.

---

## Common Protocol Mixins Quick Reference

| Protocol   | Mixin                               | Default Port | Key Methods                                          |
| ---------- | ----------------------------------- | ------------ | ---------------------------------------------------- |
| HTTP       | `Msf::Exploit::Remote::HttpClient`  | 80/443       | `send_request_cgi`, `normalize_uri`                  |
| TCP (raw)  | `Msf::Exploit::Remote::Tcp`         | -            | `connect`, `disconnect`, `sock.put`, `sock.get_once` |
| UDP        | `Msf::Exploit::Remote::Udp`         | -            | `connect_udp`, `disconnect_udp`, `udp_sock`          |
| FTP        | `Msf::Exploit::Remote::Ftp`         | 21           | `connect_login`, `send_cmd`, `data_connect`          |
| SMB        | `Msf::Exploit::Remote::SMB::Client` | 445          | `connect`, `smb_login`, `smb_create`                 |
| SSH        | `Msf::Exploit::Remote::SSH`         | 22           | Framework SSH wrapper                                |
| SMTP       | `Msf::Exploit::Remote::Smtp`        | 25           | `connect`, `raw_send_recv`                           |
| MySQL      | `Msf::Exploit::Remote::MYSQL`       | 3306         | `mysql_login`, `mysql_query`                         |
| PostgreSQL | `Msf::Exploit::Remote::Postgres`    | 5432         | `postgres_login`, `postgres_query`                   |
| MSSQL      | `Msf::Exploit::Remote::MSSQL`       | 1433         | `mssql_login`, `mssql_query`                         |
| SNMP       | `Msf::Exploit::Remote::SNMPClient`  | 161          | SNMP get/set/walk                                    |
| LDAP       | `Msf::Exploit::Remote::LDAP`        | 389/636      | LDAP bind/search                                     |
| Telnet     | `Msf::Exploit::Remote::Telnet`      | 23           | `connect`, `negotiate`                               |
| WinRM      | `Msf::Exploit::Remote::WinRM`       | 5985/5986    | `winrm_run_cmd`                                      |

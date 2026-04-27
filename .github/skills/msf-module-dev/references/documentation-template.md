# Module Documentation Template

Every module submission to metasploit-framework should include a companion documentation file. Documentation lives alongside the module at `documentation/modules/<type>/<path>/<module_name>.md`.

For example, module `modules/auxiliary/gather/webapp_data_dump.rb` gets documentation at `documentation/modules/auxiliary/gather/webapp_data_dump.md`.

---

## Template

The documentation file should contain these sections in order. Copy this structure when creating a new module doc file.

### Section 1: Vulnerable Application

```markdown
## Vulnerable Application

This module exploits an authentication bypass in Acme WebApp versions prior to 4.2.1.
An unauthenticated attacker can compute valid admin credentials via a weak key derivation
in the /api/login endpoint. The vulnerability was confirmed on version 4.2.0.

A dockerized test environment can be set up using the official Docker image:
https://hub.docker.com/r/acme/webapp
```

This section must describe the vulnerability being exploited: what it is, what an attacker can do with it, and what access is required. Follow with the affected product/versions and setup instructions. Include links to downloads, advisories, specific versions tested, and Docker/Vagrant setup if available. Write so someone can reproduce the environment 5+ years later.

Do NOT write a generic "this module targets X" without describing the actual vulnerability. Real examples from MSF:
- "This module exploits an improper access control vulnerability (CVE-2023-6329) in Control iD iDSecure <= v4.7.43.0. It allows an unauthenticated remote attacker to compute valid credentials and to add a new administrative user."
- "Attackers with knowledge of a valid username can provide a crafted S3 authentication header to the CrushFTP web API to authenticate as that user without valid credentials."

### Section 2: Verification Steps

```markdown
## Verification Steps

1. Install the application
1. Start msfconsole
1. Do: `use auxiliary/gather/webapp_data_dump`
1. Do: `set RHOSTS <target>`
1. Do: `set USERNAME admin@example.com`
1. Do: `set PASSWORD password123`
1. Do: `run`
1. You should see extracted data stored as loot.
```

Numbered list showing the basic usage flow. Must start with installation and end with expected result.

### Section 3: Options

```markdown
## Options

### TARGETURI

The base path to the web application. Default: `/`.

### USERNAME

The username or email address to authenticate with.

### PASSWORD

The password for the specified user account.
```

Document each custom option (not inherited ones like RHOSTS/RPORT). Include the default value if relevant.

### Section 4: Scenarios

````markdown
## Scenarios

### Acme WebApp 4.2.0 on Ubuntu 22.04

```
msf6 > use auxiliary/gather/webapp_data_dump
msf6 auxiliary(gather/webapp_data_dump) > set RHOSTS 192.168.1.100
RHOSTS => 192.168.1.100
msf6 auxiliary(gather/webapp_data_dump) > set USERNAME admin@corp.local
USERNAME => admin@corp.local
msf6 auxiliary(gather/webapp_data_dump) > set PASSWORD Summer2024!
PASSWORD => Summer2024!
msf6 auxiliary(gather/webapp_data_dump) > run

[*] Authenticating to API...
[+] Authenticated successfully. Token obtained.
[*] Enumerating resources...
[+] Found 12 resources with 847 entries
[*] Extracting data...
[+] Data saved in: /home/user/.msf4/loot/20240101120000_default_192.168.1.100_webapp.data_123456.txt
[*] Auxiliary module execution completed
```
````

Include the target OS/app version in the heading. Copy-paste actual msfconsole output. If the module works against multiple versions or OSes, include separate scenario sections for each.

---

## Documentation file path convention

| Module path                                      | Documentation path                                             |
| ------------------------------------------------ | -------------------------------------------------------------- |
| `modules/exploits/linux/http/webapp_rce.rb`      | `documentation/modules/exploits/linux/http/webapp_rce.md`      |
| `modules/auxiliary/gather/webapp_data_dump.rb`   | `documentation/modules/auxiliary/gather/webapp_data_dump.md`   |
| `modules/auxiliary/scanner/http/webapp_login.rb` | `documentation/modules/auxiliary/scanner/http/webapp_login.md` |

## Key guidelines

- **Be specific**: Include exact version numbers, OS versions, and configuration details
- **Include links**: Link to vulnerable software downloads, vendor advisories, CVE entries
- **Show real output**: Copy-paste actual msfconsole sessions, not hypothetical ones
- **Plan for the future**: Write as if someone will need to reproduce this setup in 5+ years
- **Multiple scenarios**: If the module works against multiple versions or OSes, include separate scenarios for each
- **Heading hierarchy**: Use `###` for OS/version headings under Scenarios, `####` for sub-scenarios. Never skip heading levels.
- **Only relevant output**: Do not include unrelated module output, msfconsole banner text, or prior commands — only the output from the module being documented
- **Document ALL custom options**: Every `register_options` entry needs a corresponding Options section entry with its default value
- **No excessive escaping**: Do not escape characters that don't need escaping in Markdown (e.g., `\_` is unnecessary, use `_`)

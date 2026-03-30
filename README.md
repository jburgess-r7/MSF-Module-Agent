# MSF Module Development Agent

A VS Code Copilot agent and skill for writing production-quality [Metasploit Framework](https://github.com/rapid7/metasploit-framework) modules that pass `msftidy`, RuboCop, and Rapid7 code review on the first submission.

## What It Does

When invoked, the `@msf-module` agent:

1. Reads your PoC script or vulnerability description from the workspace
2. Classifies the correct module type (`auxiliary/gather/`, `exploit/multi/http/`, etc.)
3. Writes a complete MSF module following Rapid7's exact conventions — metadata, mixins, error handling, credential reporting, loot storage
4. Validates the output against the built-in checklist (Notes hash, DisclosureDate format, nil response checks, etc.)
5. Generates companion documentation (Vulnerable Application, Verification Steps, Scenarios)

The skill encodes real MSF framework internals — method signatures, option types, RuboCop cops, msftidy checks — sourced directly from the framework source and real Rapid7 maintainer PR review feedback, not documentation.

## What's Included

```
.github/
├── agents/
│   └── msf-module.agent.md          # Agent definition (persona, workflow, constraints)
├── skills/
│   └── msf-module-dev/
│       ├── SKILL.md                  # Core knowledge base (rules, patterns, checklist)
│       └── references/
│           ├── module-template.md    # Annotated auxiliary + exploit templates
│           ├── metadata.md           # Notes, References, Rankings, Options
│           ├── reporting.md          # store_loot, credentials, Rex::Text::Table
│           ├── http-client.md        # HttpClient mixin API reference
│           ├── non-http-modules.md   # Scanner, TCP, FTP, SMB, Local, Post patterns
│           └── documentation-template.md  # Module documentation template
└── README.md                         # This file
```

## Installation

Copy the `agents/` and `skills/` folders into your workspace's `.github/` directory:

```
your-workspace/
├── .github/
│   ├── agents/
│   │   └── msf-module.agent.md
│   └── skills/
│       └── msf-module-dev/
│           ├── SKILL.md
│           └── references/
│               └── ...
├── your-poc.py
├── advisory.md
└── ...
```

The agent will appear in VS Code's Copilot Chat agent picker as `@msf-module`.

### Requirements

- VS Code with GitHub Copilot Chat extension
- Metasploit Framework installed locally (for validation via `msftidy` / `rubocop`). The skill references the default install path at `/opt/metasploit-framework/embedded/framework/` — adjust the agent file if yours differs.

## Usage

### Basic: Convert a PoC to an MSF module

Add your PoC script and any supporting files (advisory writeup, vulnerability details) to the workspace, then:

```
@msf-module Convert my PoC (exploit.py) into a Metasploit auxiliary gather module.
Target is Acme WebApp < 4.2.1, authenticated stored XSS that steals session tokens.
CVE-2024-12345.
```

### Specific: Write a module from a vulnerability description

```
@msf-module Write an auxiliary/gather module for CVE-2024-XXXXX.
The vuln is a path traversal in /api/v1/files?path=../../etc/passwd on Vendor Product < 3.0.
No auth required. HTTPS on port 8443 by default.
```

### Review: Check an existing module

```
@msf-module Review my module (modules/auxiliary/gather/webapp_dump.rb) for msftidy
and rubocop compliance. Fix any issues.
```

### Tips for better results

- **Include the PoC in the workspace** — the agent reads it to understand the attack flow, HTTP methods, data formats, and auth requirements
- **Specify the CVE** — it gets wired into References metadata automatically
- **Mention affected versions** — goes into the Description and documentation
- **State the module type if you know it** — otherwise the agent classifies it from the classification table
- **Include advisory/writeup files** — the more context, the better the Description and documentation

## Customization

### Different MSF install path

Edit the constraint in [agents/msf-module.agent.md](agents/msf-module.agent.md):

```diff
- the framework source at `/opt/metasploit-framework/embedded/framework/`
+ the framework source at `/usr/share/metasploit-framework/`
```

### Adding new reference material

Add `.md` files to `skills/msf-module-dev/references/` and link them from the Reference Files table at the bottom of [skills/msf-module-dev/SKILL.md](skills/msf-module-dev/SKILL.md).

### Extending the agent

The agent file supports standard VS Code custom agent properties — you can add or remove tools, adjust the description keywords for better invocation matching, or modify the workflow steps.

## Coverage

The skill covers the most common MSF module patterns:

- **Module types**: Auxiliary (gather, admin, scanner), Exploit (remote HTTP, local privesc), Post
- **HTTP mixins**: `HttpClient` — GET/POST/PUT, JSON, SOAP/XML, multipart file upload, cookies, redirects, basic auth
- **Non-HTTP mixins**: TCP, UDP, FTP, SMB, SSH, SMTP, MySQL, PostgreSQL, MSSQL, SNMP, LDAP, Telnet, WinRM
- **Scanner modules**: `Msf::Auxiliary::Scanner` with `run_host(ip)` and batch patterns
- **Local exploits**: `Msf::Exploit::Local`, `AutoCheck` (prepend), `FileDropper`, `SessionTypes`
- **Post modules**: `Msf::Post`, session interaction, `Msf::Post::File`, platform/OS-specific mixins
- **Check method**: All `CheckCode` constants with reason strings
- **Reporting**: `store_loot`, `create_credential`/`create_credential_login`, `report_service`, `Rex::Text::Table`
- **Metadata**: All Notes constants, Reference types, Rankings, Option types, DefaultOptions
- **Validation**: msftidy checks, RuboCop cops, manual checklist
- **Documentation**: Module doc template with Vulnerable Application, Verification Steps, Options, Scenarios

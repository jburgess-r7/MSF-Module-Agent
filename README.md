# MSF Module Development Agent

A cross-client agent skill for creating, reviewing, fixing, and manually validating [Metasploit Framework](https://github.com/rapid7/metasploit-framework) modules against the policy and APIs in the current checkout. It supports Codex, Claude Code, and GitHub Copilot from one canonical skill.

The package is designed to reduce avoidable review feedback, not guarantee maintainer acceptance. Metasploit conventions evolve, so applicable `AGENTS.md` files, current framework source and tests, official documentation, and recent Rapid7 review evidence take priority over a single older module.

## What It Does

When invoked, the **Metasploit Module Developer** agent or skill:

1. Reads applicable `AGENTS.md` instructions and scopes the requested work.
2. Classifies the module and checks for existing modules or open pull requests covering the same vulnerability.
3. Verifies mixin contracts, inherited options, helpers, and reporting APIs against the current checkout.
4. Traces detection, exploitation or gathering, failure paths, reporting, cleanup, repeatability, and documentation.
5. Creates the smallest coherent implementation or review fix while preserving unrelated work.
6. Runs source-checkout validation such as Ruby syntax, RuboCop, `msftidy`, documentation tidy, focused specs, and `git diff --check`.
7. Builds a manual QA matrix covering targets, payload selection, AutoCheck behavior, partial failure, cleanup, and immediate reruns.

It can work from a PoC or vulnerability description, review an existing module, respond to pull-request feedback, and reassess a module after a rebase or framework behavior change.

The agent does not fabricate documentation Scenarios, test output, sessions, or root-cause claims. A human must run the relevant lab scenarios and record real results. It also does not commit, push, or alter a pull request unless explicitly authorized.

## What's Included

```text
.
├── .agents/
│   └── skills/
│       └── msf-module-dev/
│           ├── SKILL.md
│           ├── agents/
│           │   └── openai.yaml
│           └── references/
│               ├── review-qa.md
│               ├── module-template.md
│               ├── metadata.md
│               ├── http-client.md
│               ├── reporting.md
│               ├── non-http-modules.md
│               └── documentation-template.md
├── .claude/
│   ├── agents/
│   │   └── msf-module-developer.md
│   └── skills/
│       └── msf-module-dev/
│           └── SKILL.md
├── .github/
│   └── agents/
│       └── msf-module.agent.md
└── README.md
```

- [The canonical `SKILL.md`](.agents/skills/msf-module-dev/SKILL.md) contains the shared workflow and routes every client to relevant references.
- [`msf-module.agent.md`](.github/agents/msf-module.agent.md) is the thin GitHub Copilot custom-agent launcher.
- [`msf-module-developer.md`](.claude/agents/msf-module-developer.md) and [the Claude skill adapter](.claude/skills/msf-module-dev/SKILL.md) expose the canonical skill to Claude Code.
- [`openai.yaml`](.agents/skills/msf-module-dev/agents/openai.yaml) provides Codex-facing display metadata.
- The canonical [`references/`](.agents/skills/msf-module-dev/references/) directory contains the detailed module patterns and QA guidance.

## Installation

Copy the directories for the clients you intend to use into the Metasploit checkout, or copy all three to support every client:

```text
metasploit-framework/
├── .agents/
│   └── skills/
│       └── msf-module-dev/
│           └── ...
├── .claude/
│   ├── agents/
│   │   └── msf-module-developer.md
│   └── skills/
│       └── msf-module-dev/
│           └── SKILL.md
└── .github/
    └── agents/
        └── msf-module.agent.md
```

Merge these directories with any existing configuration; do not replace an existing `.github/`, `.claude/`, or `.agents/` tree.

- **Codex:** Copy `.agents/`. Invoke `$msf-module-dev` explicitly or describe an MSF module-development task and allow the skill to trigger from its description.
- **GitHub Copilot:** Copy `.agents/` and `.github/agents/`, then select **Metasploit Module Developer**. Copilot also discovers the canonical skill directly from `.agents/skills/`.
- **Claude Code:** Copy `.agents/` and `.claude/`. Invoke `/msf-module-dev` or ask Claude to use the `msf-module-developer` subagent. The Claude adapter delegates to the canonical skill.

The `.claude/skills/` adapter depends on `.agents/skills/`; do not install the adapter by itself.

The layouts follow the current [Codex skill](https://learn.chatgpt.com/docs/build-skills), [GitHub Copilot agent-skill](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills), and [Claude Code skill](https://code.claude.com/docs/en/skills) conventions.

### Requirements

- Codex, Claude Code, or a GitHub Copilot environment with repository skill support.
- A Metasploit Framework source checkout containing the module under development.
- The checkout's Ruby dependencies when running RuboCop, specs, or bundled validators.
- An authorized test environment for manual vulnerability and cleanup testing.

The agent does not assume Metasploit is installed under `/opt`. It validates the active source checkout and warns about stale copies such as `~/.msf4/modules` when they could shadow the reviewed code.

## Usage

Invoke the `msf-module-dev` skill or select the **Metasploit Module Developer** agent, then describe the task and point it to the relevant local artifacts.

### Create a Module From a PoC

```text
Convert exploit.py into a Metasploit module for CVE-2026-12345.
The vulnerable product is Acme App 4.0.0 through 4.2.1; 4.2.2 is fixed.
Review advisory.md and include the companion module documentation.
```

### Review or Fix an Existing Module

```text
Review modules/exploits/multi/http/acme_app_rce.rb and its documentation.
Follow every applicable AGENTS.md, verify current mixin APIs, fix concrete issues,
and run the relevant validators. Do not commit or push anything.
```

### Respond to Pull-Request Feedback

```text
Investigate the reviewer feedback in review-notes.md against the current branch.
Reproduce behavioral claims before changing code, keep the fix focused, and give
me a manual regression matrix and a draft PR response. Do not commit or push.
```

### Reassess After a Rebase or Payload Change

```text
Re-review this module after the framework rebase. Test automatic payload selection,
every target, AutoCheck true/false, ForceExploit where relevant, cleanup after partial
failure, and an immediate rerun. Separate observations from proven causes.
```

Include the PoC, advisory, protocol notes, reviewer comments, affected/fixed versions, expected platforms and architectures, and available lab topology when known.

## Validation Workflow

From the Metasploit repository root, the agent uses the checkout's own tooling:

```bash
ruby -c modules/<type>/<path>/<module>.rb
bundle exec rubocop modules/<type>/<path>/<module>.rb
ruby tools/dev/msftidy.rb modules/<type>/<path>/<module>.rb
ruby tools/dev/msftidy_docs.rb documentation/modules/<doc-type>/<path>/<module>.md
git diff --check
```

Library changes should also receive focused RSpec coverage. Manual validation should exercise, where applicable:

- unrelated and patched targets;
- `check`, `AutoCheck true`, `AutoCheck false`, and `ForceExploit`;
- every target and action;
- automatic selection and representative explicit payloads;
- partial failure after each mutation, successful cleanup, and an immediate rerun;
- operation with the database disconnected;
- harmless interaction with an opened session rather than relying only on "session opened";
- documentation output refreshed from actual tested behavior.

An isolated console configuration helps prevent stale user modules and settings from affecting results:

```bash
MSF_CFGROOT_CONFIG="$(mktemp -d /tmp/msf-module-test.XXXXXX)" ./msfconsole -q
```

## Coverage

- Remote and local exploits, auxiliary gather/admin/scanner modules, and post modules.
- Reasoned `CheckCode` results and correct AutoCheck/ForceExploit behavior.
- HTTP requests, structured parsing, multipart data, cookies, redirects, timeouts, and asynchronous polling.
- Raw TCP and LoginScanner contracts plus reporting ownership for FTP, SMB, SSH, LDAP, and DCERPC mixins.
- Payload/target compatibility, framework payload selection, fetch payloads, and generated commands.
- Exact application-service reporting for vulnerabilities, credentials, logins, and loot.
- Filesystem and application-state cleanup, partial failures, repeatability, and rerun testing.
- Metadata, options, targets, architecture/platform declarations, Notes, and side effects.
- Module documentation, validation tools, pull-request review, and regression QA.

The current checkout remains the source of truth. Existing modules and past review comments are evidence to verify, not permanent policy.

## Customization and Maintenance

Keep domain guidance in [the canonical skill](.agents/skills/msf-module-dev/SKILL.md). The Copilot and Claude agent files should remain thin launchers rather than repeat the workflow.

Add focused material under [`.agents/skills/msf-module-dev/references/`](.agents/skills/msf-module-dev/references/) and link it from the References table in [the canonical `SKILL.md`](.agents/skills/msf-module-dev/SKILL.md). Keep the core skill concise and route detailed or variant-specific guidance to references.

When changing the skill name or trigger description, keep the Claude adapter metadata aligned and regenerate `agents/openai.yaml`. Before a substantial refresh, compare the package against applicable `AGENTS.md` history, current framework mixins/APIs/custom cops/specs, official developer documentation, recent Rapid7 feedback in the same module area, and the current Codex, Claude Code, and GitHub Copilot formats.

After editing the package, validate the skill metadata, relative links, Markdown fences, Ruby examples, and full module examples before publishing the update.

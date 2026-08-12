# Module documentation

Start from `documentation/modules/module_doc_template.md` in the current checkout. Documentation is an operational test record, not a paraphrase of metadata.

## File path

Mirror the module path, using the documentation tree's naming:

```text
modules/exploits/multi/http/acme_rce.rb
documentation/modules/exploit/multi/http/acme_rce.md

modules/auxiliary/scanner/http/acme_version.rb
documentation/modules/auxiliary/scanner/http/acme_version.md
```

Notice that exploit documentation uses singular `exploit`, while the source directory is `exploits`.

## Required content

### Vulnerable Application

Include enough information to recreate the target years later:

- product and component;
- precise vulnerable versions and fixed version when known;
- required authentication, privileges, plugins, features, and configuration;
- target OS/architecture/runtime constraints;
- authoritative download/advisory links;
- complete setup and teardown steps.

Distinguish the advisory's affected range, the module's supported range, and the exact versions manually exercised. For multi-stage chains, attribute prerequisites and released or proposed fixes to the stage they affect.

When a reproducible Docker/Compose lab is used, keep the runnable definition in the module document when current project automation/review expects it. Pin vulnerable image/package versions. Do not rely only on an external repository that may disappear or change.

Example shape:

````markdown
## Vulnerable Application

Acme App versions 4.0.0 through 4.2.1 are vulnerable. Version 4.2.2
fixes the issue. The vulnerable endpoint is enabled by default and does not
require authentication.

Save the following as `compose.yml`:

```yaml
services:
  acme:
    image: example/acme:4.2.1
    ports:
      - "8080:8080"
```

Start the target with `docker compose up --detach`.
````

Use valid nested Markdown fencing in the real document.

### Verification Steps

Write exact operator steps with the final module path and required options:

```markdown
## Verification Steps

1. Start a vulnerable Acme App instance as described above.
2. Start Metasploit Framework from the source checkout.
3. Run: `use exploit/multi/http/acme_rce`
4. Run: `set RHOSTS <target-host>`
5. Run: `set RPORT 8080`
6. Run: `check`
7. Run: `run`
8. Verify that the selected payload succeeds and created artifacts are removed.
```

Do not promise a shell for targets/actions that do not create sessions.

### Options

Document module-specific options, changed inherited defaults, conditional behavior, and operationally important advanced options. Match spelling and case exactly:

```markdown
## Options

### TARGETURI

The Acme App base path. The default is `/`.

### VerifyTimeout

The number of seconds to wait for the asynchronous job to finish. The default
is 60 seconds.
```

Cross-check every registered option, default, validation constraint, and conditional behavior against the document. Remove stale options when code changes.

### Scenarios

Scenarios must be supplied and refreshed by a human from real target output. An agent may format user-provided output but must not invent:

- console lines or success messages;
- session numbers, timestamps, payload sizes, ports, or credentials;
- product/OS versions or architecture;
- cleanup/reporting output.

Each scenario heading identifies product version, OS/container image, architecture, target, and meaningful payload type. Include `check`, exploitation/gathering, proof of effect, cleanup, and an immediate rerun where repeatability matters.

State whether the captured output used the document's exact setup recipe or a different lab. Do not attribute output to a recipe when materially relevant versions, dependencies, configuration, or topology differ; record those differences.

```markdown
## Scenarios

### Acme App 4.2.1 on Ubuntu 24.04 x64 - Unix/Linux Command target

<!-- Replace this comment with human-recorded console output before submission. -->
```

A placeholder is not acceptable in a submitted pull request; pause for human test output.

Redact credentials, API tokens, hashes, cookies, and other secrets from the captured output, including values generated for a disposable test account. Preserve the fact that a credential was created with placeholders such as `<generated-password>`.

## Keep documentation diff-aware

Refresh the document whenever code changes:

- option names, casing, requiredness, conditions, or defaults;
- target/action names or selection behavior;
- default/compatible payload behavior;
- printed status/error/cleanup lines;
- exploit prerequisites, version range, or lab setup;
- created artifacts, restoration, or cleanup order;
- verification evidence and expected result.

Documentation for an existing module must be updated too; docs are not only a new-module requirement.

## Safety and validation

- Use placeholders such as `<username>`, `<password>`, and `<api-token>` rather than real or real-looking secrets.
- Do not include private IPs, credentials, keys, or hashes taken from an actual environment. Local/private lab addresses are allowed in genuine Scenarios when safe; unit/spec examples use TEST-NET-1 (`192.0.2.0/24`).
- Ensure commands are copy/pasteable and fenced with the correct language.
- Verify referenced files/URLs and pin versions where practical.
- Run:

  ```bash
  ruby tools/dev/msftidy_docs.rb documentation/modules/<doc-type>/<path>/<name>.md
  ```

- Compare the final human Scenario with the final code after the last rebase/change.

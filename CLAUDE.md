# CLAUDE.md - Project memory for Claude Code
<!-- This file is automatically loaded at the start of every Claude Code session. -->

## What is this project?
Ansible project for automated installation and configuration of IIS on
Windows Server 2022, including .NET runtimes and the URL Rewrite Module.
Started in collaboration with Claude (claude.ai), maintained going forward
via Claude Code.

## Language and communication
- Write comments, variable names, documentation, and this CLAUDE.md file in **English**
- Communicate with the user in **Dutch**
- Do **not** add Claude references to commit messages (no Co-Authored-By or other AI attribution)

## Architectural decisions (do not change without good reason)

### Disk layout
- OS lives on `C:\` — no IIS data goes there
- All IIS data on `D:\`: wwwroot, logs, temp, compression, backups
- Temporary installer files on `C:\Temp\ansible_downloads` (cleaned up after use)

### Collections
- `microsoft.iis` for website and app pool management — **not** `community.windows`
  win_iis_* modules, which are deprecated and will be removed in v4.0
- `ansible.windows` for general Windows tasks
- `community.windows` for win_dotnet_ngen (firewall is handled by verifying
  the default IIS firewall rules in `iis_configure.yml`)
- `ansible.posix` for the `timer` and `profile_tasks` callback plugins
  (`callbacks_enabled = ansible.posix.timer, ansible.posix.profile_tasks` in `ansible.cfg`)

### Package source: `iis_package_source`

The role supports two strategies for fetching installer files.
The choice is controlled by `iis_package_source` in `defaults/main.yml`:

```yaml
iis_package_source: url    # Default: download directly from Microsoft CDN
iis_package_source: share  # Copy from an internal UNC share
```

**Why two strategies?**

| Situation | Strategy | Reason |
|---|---|---|
| GitHub Actions CI | `url` | Runner has no access to internal shares |
| Production / OTAP | `share` | Servers without internet access; controlled distribution process |

**Task order — identical after the source choice:**

```
[url]   → win_get_url (Microsoft CDN) → temp → verification → install → cleanup
[share] → win_copy (UNC path)         → temp →               install → cleanup
```

The install and cleanup tasks are **identical** for both strategies. Only
the copy step differs. Implement this with `when: iis_package_source == 'url'`
and `when: iis_package_source == 'share'` on the respective copy tasks,
**not** by creating separate task files per strategy.

**Verification per strategy:**
- `url`: checksum verification via `win_get_url` (SHA512 for .NET, Authenticode for URL Rewrite)
- `share`: no additional verification — the organisation is responsible for the integrity
  of files on the internal share

**Share configuration belongs in `group_vars`, never in the role:**
```yaml
# inventory/group_vars/windows/main.yml
iis_package_share: '\\fileserver\software\iis'
```

`iis_package_share` has no default in `defaults/main.yml` and is only used
when `iis_package_source: share`. Claude Code must generate a clear error
if `iis_package_source: share` is set but `iis_package_share` is not defined:

```yaml
- name: Verify share path is defined when source is share
  ansible.builtin.assert:
    that: iis_package_share is defined and iis_package_share | length > 0
    fail_msg: "'iis_package_share' must be set in group_vars when iis_package_source is 'share'"
  when: iis_package_source == 'share'
```

**CI override in `ci/inventory/`:**
```yaml
iis_package_source: url  # Explicit override — CI always downloads directly
```

### .NET installation strategy
- Hosting Bundle per version (installs x64 and x86 runtime + ASP.NET + IIS module)
- Separate x86 runtime optionally alongside the bundle (`iis_dotnet_install_individual_x86`)
- Versions defined as a list in `defaults/main.yml` → `iis_dotnet_versions`
- Which versions are active per environment: `iis_dotnet_enabled_versions` in group_vars

#### Download URLs and checksums via the Microsoft .NET Releases API

Download URLs and SHA512 checksums are **not** hardcoded. Microsoft publishes
a fully machine-readable JSON API that is updated on every Patch Tuesday.
The role fetches the URL and checksum dynamically via two layers.

**Layer 1 — Release index (all channels):**
```
https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json
```

Key fields per channel entry:

| Field | Description |
|---|---|
| `channel-version` | Version string, e.g. `"8.0"` |
| `support-phase` | `"active"`, `"maintenance"` or `"eol"` |
| `release-type` | `"lts"` or `"sts"` |
| `eol-date` | End-of-support date (ISO 8601) |
| `releases.json` | URL to Layer 2 |

**Layer 2 — Per-channel releases.json:**
Follow the `releases.json` URL from Layer 1. Contains a `releases` array with
per-patch-release sub-objects for `runtime`, `aspnetcore-runtime` and `sdk`.
Each sub-object has a `files` array with:

| Field | Description |
|---|---|
| `name` | Filename, e.g. `dotnet-hosting-win.exe` |
| `rid` | Runtime identifier, e.g. `win`, `win-x64` |
| `url` | Direct download URL |
| `hash` | SHA512 checksum (hex) |

**Component mapping:**

| Component | Filename | Sub-object in releases.json |
|---|---|---|
| Hosting Bundle | `dotnet-hosting-win.exe` | `aspnetcore-runtime.files` |
| ASP.NET Core Runtime | `aspnetcore-runtime-win-x64.exe` | `aspnetcore-runtime.files` |
| .NET Runtime x64 | `dotnet-runtime-win-x64.exe` | `runtime.files` |
| .NET Runtime x86 | `dotnet-runtime-win-x86.exe` | `runtime.files` |

The Hosting Bundle is a superset: it installs the .NET Runtime, ASP.NET Core
Runtime and the IIS integration module (ANCM). Separate components are
generally not needed.

**Ansible implementation pattern:**
```yaml
- name: Fetch .NET release index
  ansible.builtin.uri:
    url: "https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json"
    return_content: true
  register: dotnet_release_index
  delegate_to: localhost
  run_once: true

- name: Set per-channel releases.json URL
  ansible.builtin.set_fact:
    dotnet_releases_url: >-
      {{ (dotnet_release_index.json['releases-index'] |
          selectattr('channel-version', 'equalto', dotnet_channel_version) |
          first)['releases.json'] }}

- name: Fetch per-channel release metadata
  ansible.builtin.uri:
    url: "{{ dotnet_releases_url }}"
    return_content: true
  register: dotnet_releases
  delegate_to: localhost
  run_once: true

- name: Set hosting bundle download facts
  ansible.builtin.set_fact:
    dotnet_hosting_url: >-
      {{ (_dotnet_latest_release['aspnetcore-runtime'].files |
          selectattr('name', 'equalto', 'dotnet-hosting-win.exe') |
          first).url }}
    dotnet_hosting_checksum: >-
      {{ (_dotnet_latest_release['aspnetcore-runtime'].files |
          selectattr('name', 'equalto', 'dotnet-hosting-win.exe') |
          first).hash }}
```

`delegate_to: localhost` with `run_once: true` keeps the HTTP calls on the
control node and avoids redundant requests when targeting multiple Windows hosts.

The only version-related variable the caller sets is:
```yaml
dotnet_channel_version: "8.0"  # Or "9.0", "10.0", etc.
```

**Exception — pinning a specific patch version:**
If a user forks the role and wants a fixed patch version, `url` and `hash`
can be overridden per version entry in `defaults/main.yml`. This is
intentionally the exception, not the norm.

### URL Rewrite Module: no JSON API, but Authenticode verification

The IIS URL Rewrite Module 2.1 — unlike .NET — has **no** machine-readable
metadata API and Microsoft publishes no official checksums for the installer.
The download URL has been stable and unchanged for years (frozen product,
last DLL update September 2018):

```
# x64 (use this)
https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi

# x86 (only if explicitly required)
https://download.microsoft.com/download/D/8/1/D81E5DD6-1ABB-46B0-9B4B-21894E18B77F/rewrite_x86_en-US.msi
```

These URLs may be hardcoded in `defaults/main.yml` — there is no alternative
and they do not change.

**Verification when `iis_package_source: url`:**
Use Authenticode signature verification via PowerShell after downloading.
This is more robust than a manually maintained hash:

```yaml
- name: Verify Authenticode signature of URL Rewrite installer
  ansible.windows.win_powershell:
    script: |
      $sig = Get-AuthenticodeSignature -FilePath "{{ iis_temp_path }}\rewrite_amd64_en-US.msi"
      if ($sig.Status -ne 'Valid') {
        Write-Error "Authenticode verification failed: $($sig.Status)"
        exit 1
      }
      if ($sig.SignerCertificate.Subject -notmatch 'O=Microsoft Corporation') {
        Write-Error "Unexpected signer: $($sig.SignerCertificate.Subject)"
        exit 1
      }
  when: iis_package_source == 'url'
```

**Verification when `iis_package_source: share`:** none — the organisation
is responsible for the integrity of files on the internal share.

### DRY principle
- `dotnet_version_install.yml` is a generic helper task file (underscore = not
  standalone). Called 3× from `tasks/main.yml` via `include_tasks`.
- **Why no loop?** `include_tasks` with a loop breaks per-version tags
  (`--tags dotnet_8`). Three separate includes with their own tags is the only correct solution.
- Three related log settings are combined in a single PowerShell call in `iis_configure.yml`.

### Connection: PSRP instead of WinRM

Ansible connects to Windows via `ansible_connection: psrp` (PyPSRP library).
PSRP is chosen over `winrm` (PyWinRM) because:

| Property | PSRP | WinRM |
|---|---|---|
| Pipelining | ✓ Yes | ✗ No |
| Per-task overhead | Low (reuses connection) | High (new connection per task) |
| Authentication | basic, ntlm, kerberos, negotiate | same |
| Transport | same HTTP/HTTPS ports (5985/5986) | same |

**Variable names (PSRP):**
```yaml
ansible_psrp_auth: ntlm               # instead of ansible_winrm_transport
ansible_psrp_protocol: https          # instead of ansible_winrm_scheme
ansible_psrp_cert_validation: ignore  # instead of ansible_winrm_server_cert_validation
ansible_psrp_connection_timeout: 120  # instead of ansible_winrm_operation_timeout_sec
ansible_psrp_read_timeout: 130        # instead of ansible_winrm_read_timeout_sec
ansible_pipelining: true              # new — not available with winrm
```

**CI requirement:** `Enable-PSRemoting -Force -SkipNetworkProfileCheck` runs before
`winrm quickconfig` in the workflow — PSRP requires PowerShell Remoting endpoints.

**Python dependency:** `pypsrp>=0.7.0` (replaces `pywinrm` + `requests-ntlm`
+ `xmltodict` in requirements.txt). The `[ntlm]` extra is omitted: pypsrp 0.9.x
no longer declares it, and basic auth over HTTP does not require NTLM.

### PowerShell ngen optimisation

`os_requirements.yml` contains a task that pre-compiles PowerShell assemblies
via ngen (native image generator). This speeds up PowerShell startup by ~10×
and reduces overhead on every Ansible module call.

- Source: [Ansible Windows performance guide](https://docs.ansible.com/projects/ansible/latest/os_guide/windows_performance.html)
- Location: first `pre_task` in `playbooks/site.yml` — runs before all role tasks
- Idempotent: re-running when native images already exist is safe and fast
- Tag: `ps_optimize` (only with `--tags ps_optimize` or as part of a full run)
- Always reports `changed: false` — it is optimisation, not configuration

### Security
- No credentials or usernames in the role or in git
- `ansible_user` and `ansible_password` always come from `vault.yml` (in .gitignore)
- `vault.yml.example` shows the format without real values — is committed to git
- `vars/main.yml` contains only public Microsoft constants (URLs, registry paths)
- `defaults/main.yml` contains configurable default values — no secrets

### Ansible Lint and code quality
- Always use Fully Qualified Collection Names (FQCN), e.g. `ansible.windows.win_get_url`
- Every task has a `name` with a meaningful description (sentence case)
- Handlers are notified by name only, never by module outcome
- Do not use deprecated modules or bare variable syntax (`{{ var }}` without quotes as the sole value of a YAML key)
- Inventory files always in YAML format (`.yml`), never INI
- `callbacks_enabled` in `ansible.cfg` uses FQCN: `ansible.posix.timer, ansible.posix.profile_tasks`
- `ansible_managed` is **not** in `ansible.cfg` (deprecated via `DEFAULT_MANAGED_STR`);
  it lives in `inventory/group_vars/all/main.yml`

## OTAP structure
```
all → windows → otap_o (Development)
                otap_t (Test)
                otap_a (Acceptance)
                otap_p (Production)
```
- Production: `ansible_psrp_cert_validation: validate` (CA certificate)
- Other:      `ignore` (self-signed certificate permitted)
- HTTP permitted in Development and Test; HTTPS only in Acceptance and Production

## .NET version lifecycle (update when changed)
| Version | Type | EOL        | Action |
|---------|------|------------|--------|
| 8       | LTS  | 2026-11-10 | Fine |
| 9       | STS  | 2026-11-10 | Plan migration to 10 |
| 10      | LTS  | 2028-11-14 | Recommended |

## Available tags
| Tag                   | What it does |
|-----------------------|--------------|
| `os_requirements`     | Timezone, directory structure, TLS, firewall, pagefile |
| `ps_optimize`         | PowerShell ngen optimisation (pre_task in site.yml, not in the role) |
| `iis_features`        | Install Windows features |
| `iis_configure`       | IIS paths, logging, app pools, security |
| `iis_rewrite`         | URL Rewrite Module 2.1 |
| `dotnet`              | All .NET versions |
| `dotnet_8`            | .NET 8 only |
| `dotnet_9`            | .NET 9 only |
| `dotnet_10`           | .NET 10 only |
| `dotnet_ngen`         | .NET ngen recompilation (skipped in CI) |
| `dotnet_cleanup`      | Remove temporary installer files |
| `iis_rewrite_example` | Example HTTPS redirect rule |

## Common customisations

### Adding a new .NET version
1. Add an entry to `iis_dotnet_versions` in `defaults/main.yml`
2. Add an `include_tasks` block to `tasks/main.yml` with the appropriate tag
3. Add the major version to `iis_dotnet_enabled_versions` in the relevant group_vars

### Updating download URLs and patch versions
The role fetches the URL and checksum automatically via the Microsoft .NET Releases API
(see above). Manual updates are **not needed**. Only verify that `dotnet_channel_version`
per version entry in `defaults/main.yml` still matches the desired lifecycle phase.

To inspect a specific patch version:
- API (machine-readable): `https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json`
- Browser: `https://dotnet.microsoft.com/download/dotnet`

### Adding a new environment
1. Add hosts to `inventory/hosts.yml` under a new group
2. Create `inventory/group_vars/<groupname>/main.yml`
3. Create `inventory/group_vars/<groupname>/vault.yml` (not committed to git)

## Local development environment

### Python virtual environments
Two venvs available under `~/.virtualenvs/`:

| Venv        | ansible-lint version |
|-------------|----------------------|
| `ansible13` | 26.1.1 (latest)      |
| `ansible12` | 25.9.0               |

Activate and run ansible-lint:
```bash
source ~/.virtualenvs/ansible13/bin/activate
ansible-lint --nocolor
```

## CI/CD pipeline (GitHub Actions)

### Architecture
- **lint + syntax** jobs: Ubuntu runner, triggered on every push
- **integration** job: `windows-2022` GitHub-hosted runner, runs after lint and syntax
- Pushes that change **only `.md` files** do not trigger CI
  (`paths-ignore: ["*.md", "**/*.md"]` — two patterns because `**.md` does not
  reliably match root-level files in GitHub Actions)

### How the integration job works
The Windows runner configures WinRM + PSRemoting on itself (`127.0.0.1:5985`,
HTTP basic auth, temporary local admin). Ansible runs from WSL (Ubuntu 22.04) on
the same machine and connects back via PSRP. After the first run an idempotency
check follows (the second run must report `changed=0`).

**WSL configuration before the first wsl-bash step (PowerShell):**
- DrvFs metadata: via `wsl --exec sh -c "printf ... > /etc/wsl.conf"` followed by
  `wsl --shutdown`. Without `metadata`, files on NTFS mounts appear as `0777` and
  Ansible ignores `ansible.cfg`.
- Timezone: `iana_timezone` from `ci/inventory/group_vars/all.yml` is read with
  `Select-String`, written as `TZ=…` to `$GITHUB_ENV`, and forwarded to all
  wsl-bash steps via `WSLENV`. This gives local timestamps in the `profile_tasks`
  and `timer` callbacks.

**Other CI settings:**
- `ANSIBLE_CACHE_PLUGIN=memory` per playbook run (environment variable): prevents
  warnings from the jsonfile cache when moving files on DrvFs mounts.

### CI-specific overrides (`ci/inventory/`)
| Override | CI value | Reason |
|---|---|---|
| `iis_data_drive` | `C:` | Windows runners have no D: drive |
| All `iis_*_path` | `C:\IIS\...` | Follows from the above |
| `iis_dotnet_enabled_versions` | `["8"]` | Shorter pipeline runtime |
| `iis_package_source` | `url` | CI has no access to internal shares |
| `iana_timezone` | `Europe/Amsterdam` | Control node timezone (WSL); CI-only variable |

### GitHub Actions log analysis

When investigating CI failures, always fetch the **full log**, not just the tail.
GitHub Actions truncates long lines and `--log-failed` sometimes shows only the
task header without the result when the step aborts immediately after that task.

```bash
# Full log — use this as the starting point
gh run view <run-id> --repo <owner>/<repo> --log 2>&1 | grep -E "TASK|fatal|FAILED|changed|ERROR" | head -100

# Failed step output only (useful for large runs)
gh run view <run-id> --repo <owner>/<repo> --log-failed 2>&1 | tail -100

# Context around a specific task
gh run view <run-id> --repo <owner>/<repo> --log 2>&1 | grep -A 10 "task name"
```

If a task header appears but no result follows, and the next step immediately
starts with `Post Run`, the Ansible step aborted at that point. In that case
view the full step output (not just the tail) to find whether a `fatal:` line
precedes it.

### Local testing
```bash
# Lint and syntax (Linux/Mac)
source ~/.virtualenvs/ansible13/bin/activate
ansible-lint --nocolor
ansible-playbook playbooks/site.yml --syntax-check -i ci/inventory/hosts.yml
```

## .gitignore — required exclusions
The `.gitignore` in the repository root must contain at minimum:

```gitignore
# Ansible secrets
*.vault
vault_pass.txt
.vault_pass
group_vars/*/vault.yml
host_vars/*/vault.yml

# Local inventory overrides
inventory/local/
inventory/*.local.yml

# Editor and OS noise
.idea/
.vscode/
*.swp
.DS_Store
Thumbs.db

# Python virtual environments
.venv/
__pycache__/
*.pyc
```

Use `ansible-vault encrypt_string` for secrets that must live in version control.
Never commit `ansible.cfg` entries containing passwords, tokens, or internal hostnames.

## Not yet built (possible extensions)
- HTTPS binding configuration (attaching a certificate to a site)
- Multiple websites and app pools per server
- Automated Windows Updates installation
- Monitoring/alerting integration

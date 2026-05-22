# CLAUDE.md - Projectgeheugen voor Claude Code
<!-- Dit bestand wordt automatisch ingeladen bij elke Claude Code sessie. -->

## Wat is dit project?
Ansible project voor geautomatiseerde installatie en configuratie van IIS op
Windows Server 2022, inclusief .NET runtimes en URL Rewrite Module.
Ontwikkeld in samenwerking met Claude (claude.ai) als startpunt, verder te
beheren via Claude Code.

## Taal en communicatie
- Schrijf comments, variabelenamen en documentatie in het **Engels**
- Communiceer met de gebruiker in het **Nederlands**
- Voeg **geen** Claude-verwijzingen toe aan commit messages (geen Co-Authored-By of andere AI-attributie)

## Architectuurkeuzes (niet zomaar wijzigen)

### Schijfindeling
- OS staat op `C:\` — daar komt niets van IIS bij
- Alle IIS data op `D:\`: wwwroot, logs, temp, compressie, backups
- Tijdelijke installatiebestanden op `C:\Temp\ansible_downloads` (worden opgeruimd)

### Collections
- `microsoft.iis` voor website en app pool beheer — **niet** `community.windows`
  win_iis_* modules, die zijn deprecated en worden verwijderd in v4.0
- `ansible.windows` voor algemene Windows taken
- `community.windows` voor firewall en win_dotnet_ngen

### .NET installatiestrategie
- Hosting Bundle per versie (installeert x64 én x86 runtime + ASP.NET + IIS module)
- Losse x86 runtime optioneel naast de bundle (`iis_dotnet_install_individual_x86`)
- Versies gedefinieerd als lijst in `defaults/main.yml` → `iis_dotnet_versions`
- Welke versies actief zijn per omgeving: `iis_dotnet_enabled_versions` in group_vars

#### Download-URL's en checksums via de Microsoft .NET Releases API

Download-URL's en SHA512-checksums worden **niet** hardgecodeerd. Microsoft
publiceert een volledig machine-leesbare JSON API die bij elke Patch Tuesday
wordt bijgewerkt. De rol haalt URL en checksum dynamisch op via twee lagen.

**Laag 1 — Release-index (alle kanalen):**
```
https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json
```

Sleutelvelden per kanaal-entry:

| Veld | Beschrijving |
|---|---|
| `channel-version` | Versiestring, bv. `"8.0"` |
| `support-phase` | `"active"`, `"maintenance"` of `"eol"` |
| `release-type` | `"lts"` of `"sts"` |
| `eol-date` | Einddatum ondersteuning (ISO 8601) |
| `releases.json` | URL naar Laag 2 |

**Laag 2 — Per-kanaal releases.json:**
Volg de `releases.json`-URL uit Laag 1. Bevat een `releases`-array met per
patch-release sub-objecten voor `runtime`, `aspnetcore-runtime` en `sdk`.
Elk sub-object heeft een `files`-array met:

| Veld | Beschrijving |
|---|---|
| `name` | Bestandsnaam, bv. `dotnet-hosting-win.exe` |
| `rid` | Runtime identifier, bv. `win`, `win-x64` |
| `url` | Directe download-URL |
| `hash` | SHA512-checksum (hex) |

**Componentmapping:**

| Component | Bestandsnaam | Sub-object in releases.json |
|---|---|---|
| Hosting Bundle | `dotnet-hosting-win.exe` | `runtime.files` |
| ASP.NET Core Runtime | `aspnetcore-runtime-win-x64.exe` | `aspnetcore-runtime.files` |
| .NET Runtime | `dotnet-runtime-win-x64.exe` | `runtime.files` |

De Hosting Bundle is een superset: hij installeert de .NET Runtime, ASP.NET Core
Runtime én de IIS-integratiemodule (ANCM). Losse componenten zijn doorgaans
niet nodig.

**Ansible-implementatiepatroon:**
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
      {{ (dotnet_releases.json.releases |
          selectattr('release-version', 'equalto', dotnet_releases.json['latest-release']) |
          first).runtime.files |
          selectattr('name', 'equalto', 'dotnet-hosting-win.exe') |
          first | community.general.json_query('url') }}
    dotnet_hosting_checksum: >-
      sha512:{{ (dotnet_releases.json.releases |
          selectattr('release-version', 'equalto', dotnet_releases.json['latest-release']) |
          first).runtime.files |
          selectattr('name', 'equalto', 'dotnet-hosting-win.exe') |
          first | community.general.json_query('hash') }}
```

`delegate_to: localhost` met `run_once: true` houdt de HTTP-aanroepen op de
control node en voorkomt onnodige requests bij meerdere Windows-targets.

De enige versiegerelateerde variabele die de aanroeper instelt is:
```yaml
dotnet_channel_version: "8.0"  # Of "9.0", "10.0", etc.
```

**Uitzondering — specifieke patch-versie pinnen:**
Wil een gebruiker de rol forken en een vaste patch-versie gebruiken, dan kan
`url` en `hash` per versie worden overschreven in `defaults/main.yml`. Dit is
bewust de uitzondering, niet de norm.

### DRY-principe
- `_dotnet_version_install.yml` is een generiek helper-taakbestand (underscore = niet
  standalone). Wordt 3× aangeroepen vanuit `tasks/main.yml` via `include_tasks`
- **Waarom geen loop?** `include_tasks` met een loop breekt per-versie tags
  (`--tags dotnet_8`). Drie aparte includes met eigen tags is de enige correcte oplossing.
- Drie gerelateerde log-instellingen zitten in één PowerShell-aanroep in `iis_configure.yml`

### Beveiliging
- Geen credentials of gebruikersnamen in de rol of in git
- `ansible_user` en `ansible_password` komen altijd uit `vault.yml` (staat in .gitignore)
- `vault.yml.example` toont het formaat zonder echte waarden — wél in git
- `vars/main.yml` bevat alleen publieke Microsoft-constanten (URLs, registry-paden)
- `defaults/main.yml` bevat configureerbare standaardwaarden — geen secrets

### Ansible Lint en codekwaliteit
- Gebruik altijd Fully Qualified Collection Names (FQCN), bv. `ansible.windows.win_get_url`
- Elke taak heeft een `name` met een betekenisvolle omschrijving (sentence case)
- Handlers worden alleen bij naam genotificeerd, nooit door module-uitkomst
- Gebruik geen deprecated modules of kale variabele-syntax (`{{ var }}` zonder aanhalingstekens als enige waarde van een YAML-sleutel)
- Inventory bestanden altijd in YAML-formaat (`.yml`), nooit INI

## OTAP-structuur
```
all → windows → otap_o (Ontwikkeling)
                otap_t (Test)
                otap_a (Acceptatie)
                otap_p (Productie)
```
- Productie: `ansible_winrm_server_cert_validation: validate` (CA-certificaat)
- Overig:    `ignore` (zelfondertekend certificaat toegestaan)
- HTTP toegestaan in O en T, alleen HTTPS in A en P

## .NET versie lifecycle (bijwerken indien gewijzigd)
| Versie | Type | EOL        | Actie |
|--------|------|------------|-------|
| 8      | LTS  | 2026-11-10 | Prima |
| 9      | STS  | 2026-11-10 | Plan migratie naar 10 |
| 10     | LTS  | 2028-11-14 | Aanbevolen |

## Beschikbare tags
| Tag                   | Wat het doet |
|-----------------------|--------------|
| `os_requirements`     | Tijdzone, mapstructuur, TLS, firewall, pagefile |
| `iis_features`        | Windows features installeren |
| `iis_configure`       | IIS paden, logging, app pools, beveiliging |
| `iis_rewrite`         | URL Rewrite Module 2.1 |
| `dotnet`              | Alle .NET versies |
| `dotnet_8`            | Alleen .NET 8 |
| `dotnet_9`            | Alleen .NET 9 |
| `dotnet_10`           | Alleen .NET 10 |
| `dotnet_cleanup`      | Tijdelijke installatiebestanden verwijderen |
| `iis_rewrite_example` | Voorbeeld HTTPS redirect regel |

## Veelgestelde aanpassingen

### Nieuwe .NET versie toevoegen
1. Voeg een item toe aan `iis_dotnet_versions` in `defaults/main.yml`
2. Voeg een `include_tasks`-blok toe in `tasks/main.yml` met de juiste tag
3. Voeg de major versie toe aan `iis_dotnet_enabled_versions` in de gewenste group_vars

### Download-URL's en patch-versies bijwerken
De rol haalt URL en checksum automatisch op via de Microsoft .NET Releases API
(zie hierboven). Handmatig bijwerken is **niet nodig**. Controleer alleen of
`dotnet_channel_version` per versie-entry in `defaults/main.yml` nog klopt
met de gewenste lifecycle-fase.

Wil je toch een specifieke patch-versie inzien, raadpleeg dan:
- API (machine): `https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json`
- Browser: `https://dotnet.microsoft.com/download/dotnet`

### Nieuwe omgeving toevoegen
1. Voeg hosts toe in `inventory/hosts.yml` onder een nieuwe groep
2. Maak `inventory/group_vars/<groepnaam>/main.yml` aan
3. Maak `inventory/group_vars/<groepnaam>/vault.yml` aan (niet in git)

## Lokale ontwikkelomgeving

### Python virtual environments
Twee venvs beschikbaar onder `~/.virtualenvs/`:

| Venv        | ansible-lint versie |
|-------------|---------------------|
| `ansible13` | 26.1.1 (nieuwst)    |
| `ansible12` | 25.9.0              |

Activeren en ansible-lint uitvoeren:
```bash
source ~/.virtualenvs/ansible13/bin/activate
ansible-lint --nocolor
```

## CI/CD pipeline (GitHub Actions)

### Architectuur
- **lint + syntax** jobs: Ubuntu runner, draaien op elke push
- **integration** job: `windows-2022` GitHub-hosted runner, draait na lint en syntax

### Hoe de integration job werkt
De Windows runner configureert WinRM op zichzelf (`127.0.0.1:5985`, HTTP basic,
tijdelijke lokale admin). Ansible draait via Python op dezelfde machine en
verbindt terug via WinRM. Na de eerste run volgt een idempotency check (tweede
run moet `changed=0` rapporteren).

### CI-specifieke overrides (`ci/inventory/`)
| Override | CI-waarde | Reden |
|---|---|---|
| `iis_data_drive` | `C:` | Windows runners hebben geen D:-schijf |
| Alle `iis_*_path` | `C:\IIS\...` | Volgt uit bovenstaande |
| `iis_dotnet_enabled_versions` | `["8"]` | Kortere pipeline runtime |

### Lokaal testen
```bash
# Lint en syntax (Linux/Mac)
source ~/.virtualenvs/ansible13/bin/activate
ansible-lint --nocolor
ansible-playbook playbooks/site.yml --syntax-check -i ci/inventory/hosts.yml
```

## .gitignore — verplichte uitsluitingen
Het `.gitignore` in de repository root moet minimaal bevatten:

```gitignore
# Ansible secrets
*.vault
vault_pass.txt
.vault_pass
group_vars/*/vault.yml
host_vars/*/vault.yml

# Lokale inventory-overrides
inventory/local/
inventory/*.local.yml

# Editor en OS-ruis
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

Gebruik `ansible-vault encrypt_string` voor secrets die per se in versiebeheer
moeten leven. Commit nooit `ansible.cfg`-entries met wachtwoorden, tokens of
interne hostnamen.

## Wat nog niet gebouwd is (mogelijke uitbreidingen)
- HTTPS binding configuratie (certificaat koppelen aan site)
- Meerdere websites en app pools per server
- Windows Updates automatisch installeren
- Monitoring/alerting integratie
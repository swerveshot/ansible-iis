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
- `defaults/main.yml` bevat download-URLs — daar horen ze thuis zodat gebruikers
  ze kunnen overschrijven zonder de rol te forken

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
| Tag                  | Wat het doet |
|----------------------|-------------|
| `os_requirements`    | Tijdzone, mapstructuur, TLS, firewall, pagefile |
| `iis_features`       | Windows features installeren |
| `iis_configure`      | IIS paden, logging, app pools, beveiliging |
| `iis_rewrite`        | URL Rewrite Module 2.1 |
| `dotnet`             | Alle .NET versies |
| `dotnet_8`           | Alleen .NET 8 |
| `dotnet_9`           | Alleen .NET 9 |
| `dotnet_10`          | Alleen .NET 10 |
| `dotnet_cleanup`     | Tijdelijke installatiebestanden verwijderen |
| `iis_rewrite_example`| Voorbeeld HTTPS redirect regel |

## Veelgestelde aanpassingen

### Nieuwe .NET versie toevoegen
1. Voeg een item toe aan `iis_dotnet_versions` in `defaults/main.yml`
2. Voeg een `include_tasks` blok toe in `tasks/main.yml` met de juiste tag
3. Voeg de major versie toe aan `iis_dotnet_enabled_versions` in de gewenste group_vars

### Download-URL's bijwerken (bij nieuwe patch releases)
Pas `patch` en `hosting_url` aan in `defaults/main.yml` → `dotnet_versions`.
Controleer actuele URLs op: https://dotnet.microsoft.com/download/dotnet

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
De Windows runner configureert WinRM op zichzelf (`127.0.0.1:5985`, HTTP basic, tijdelijke lokale admin). Ansible draait via Python op dezelfde machine en verbindt terug via WinRM. Na de eerste run volgt een idempotency check (tweede run moet `changed=0` rapporteren).

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

## Wat nog niet gebouwd is (mogelijke uitbreidingen)
- HTTPS binding configuratie (certificaat koppelen aan site)
- Meerdere websites en app pools per server
- Windows Updates automatisch installeren
- Monitoring/alerting integratie
- CI/CD pipeline (GitLab CI / GitHub Actions)
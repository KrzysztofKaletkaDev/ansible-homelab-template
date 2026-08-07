# Kontekst projektu dla Claude Code

## Co to za projekt
Ansible IaC dla jednowęzłowego homelabu na AlmaLinux 9: DNS z ad-blockingiem
(Blocky), reverse proxy z auto-TLS przez Cloudflare DNS-01 (Caddy, budowany
custom obrazem xcaddy), self-hosted menedżer haseł (Vaultwarden) i dashboard
usług (Homepage) — wszystko na Dockerze, spięte wspólną siecią `caddy-ingress`.
Docelowo jeden host (`core_nodes`).

## Twarde zasady

- **Nigdy nie commituj realnych sekretów ani plików spoza `*.example`.**
  `.gitignore` już to wymusza dla `inventory/hosts.yml` i `group_vars/**/*.yml`
  — nie omijaj tego i nie proponuj commitowania odszyfrowanego `vault.yml`.
- **Pliki zawierające zmienne z Vault (np. wygenerowane docker-compose.yml
  z tokenem Cloudflare) muszą mieć `mode: '0600'`, nie `'0644'`.** To był
  realny błąd w tym repo — pilnuj tego przy każdej zmianie w rolach.
- **Testuj zmiany w rolach na VM (Vagrantfile) lub przez Molecule, nigdy
  bezpośrednio na produkcyjnym hoście.** Szczególnie dotyczy to zadań
  z `become: yes`, operacji na firewalld i budowania obrazów Docker —
  część z nich jest trudna do bezpiecznego cofnięcia.
- Przed commitem uruchom lokalnie `pre-commit run --all-files` (ansible-lint
  + yamllint). Nie proponuj wyłączania hooków ani reguł lintera bez wyraźnej
  prośby — jeśli lint coś łapie, napraw kod, a nie regułę.

## Struktura, którą warto znać

- `site.yml` — główny playbook, kolejność ról ma znaczenie: `docker` musi
  iść przed resztą (wszystkie zależą od zainstalowanego Dockera), a `caddy`
  musi iść przed `vaultwarden` i `homepage` — to caddy tworzy sieć Docker
  `caddy-ingress` (`driver: bridge`), do której `vaultwarden`/`homepage`
  dołączają się jako `external: true`. Zmiana tej kolejności wywali wdrożenie.
- `roles/*/tasks/main.yml` — logika; `roles/*/templates/*.j2` — konfiguracja
  generowana Jinja2; `roles/*/handlers/main.yml` — restart/reload po zmianie.
- Zmienne domyślne (`domain_name`, `ansible_user`, IP urządzeń,
  `vaultwarden_dir`, `vaultwarden_version`, `homepage_dir`) żyją
  w `group_vars/all/vars.yml` (nieobecny w repo, tylko `.example`).
  Sekrety — w `group_vars/core_nodes/vault.yml` (zaszyfrowany Ansible Vault).
- Testy Molecule (`roles/*/molecule/default/`) NIE są jeszcze skonfigurowane
  dla żadnej roli — jedyna realna weryfikacja przed produkcją to
  `vagrant up`/`vagrant provision` (patrz `Vagrantfile`). Molecule pozostaje
  zalecanym, ale niewdrożonym sposobem testowania — nie zakładaj, że istnieje.

## Konwencje kodu

- Nazwy tasków po polsku (tak jak reszta repo) — trzymaj się tego stylu
  przy nowych taskach, nie przechodź samowolnie na angielski w połowie roli.
- Wartości domyślne przez `| default(...)` w szablonach i taskach zamiast
  twardego kodowania (patrz istniejące `blocky_dir | default('/opt/blocky')`).
- Każda zmiana w plikach generujących sekrety lub otwierających porty
  (firewalld) wymaga wyjaśnienia w opisie commita, dlaczego jest bezpieczna.

### Konwencje językowe

- Nazwy tasków Ansible (`name:`) — po polsku, zgodnie z resztą repo.
- Komentarze w kodzie (`#` w plikach `.yml`, `.yml.j2`, `.cfg`, `.gitignore`,
  `.ansible-lint` itp.) — po angielsku.
- Komunikaty commitów — po angielsku, w formacie Conventional Commits
  (`feat:`, `fix:`, `docs:`, `chore:` itd.).
- Nie dodawaj trailera `Co-Authored-By: Claude ...` (ani żadnego innego
  AI) do commitów — sposób pracy z Claude Code nad tym repo jest już
  opisany w tym pliku (CLAUDE.md), więc taki trailer jest zbędny i nie
  powinien pojawiać się w historii.
- `README.md` — po angielsku.
- `CLAUDE.md` — po polsku.

## Czego NIE rób bez pytania

- Nie zmieniaj kolejności ról w `site.yml` bez wyjaśnienia zależności.
- Nie usuwaj wpisów z `.gitignore` dotyczących `vault.yml` / `hosts.yml`.
- Nie dodawaj nowych zależności (kolekcji, ról) bez wpisania ich do
  `collections/requirements.yml`.

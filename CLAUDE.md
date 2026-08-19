# Kontekst projektu dla Claude Code

## Co to za projekt
Ansible IaC dla jednowęzłowego homelabu na AlmaLinux 9: DNS z ad-blockingiem
(Blocky), reverse proxy z auto-TLS przez Cloudflare DNS-01 (Caddy, budowany
custom obrazem xcaddy), self-hosted menedżer haseł (Vaultwarden), dashboard
usług (Homepage), statyczna strona wizytówkowa (kaletkadev.com, serwowana
bezpośrednio przez file_server Caddy), stos monitoringu (Prometheus,
node_exporter, cAdvisor, Grafana z dashboardami provisionowanymi jako kod)
i zero-trust ingress bez otwartych portów przez Cloudflare Tunnel
(`cloudflared`, locally-managed, `config.yml` jako kod) — wszystko na
Dockerze, spięte wspólną siecią `caddy-ingress`. Docelowo jeden host
(`core_nodes`).

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
  musi iść przed `monitoring`, `vaultwarden` i `homepage` — to caddy tworzy
  sieć Docker `caddy-ingress` (`driver: bridge`), do której `monitoring`
  (kontener Grafany), `vaultwarden` i `homepage` dołączają się jako
  `external: true`. Rola `monitoring` idzie zaraz po `caddy`, przed
  `vaultwarden`/`homepage` — trzymaj tę kolejność, zmiana wywali wdrożenie.
  `portfolio` idzie PRZED `caddy` — z innego powodu niż siatka
  `caddy-ingress`: to rola `caddy` montuje `{{ portfolio_dir }}/site` jako
  bind mount `:ro` do kontenera, więc katalog z treścią musi już istnieć
  (i być sklonowany) zanim Compose spróbuje go zamontować. `cloudflared`
  idzie NA KOŃCU listy, po `homepage` — potrzebuje istniejącej sieci
  `caddy-ingress` (dołącza się jako `external: true`, tak samo jak
  `monitoring`/`vaultwarden`/`homepage`), a kolejność względem pozostałych
  usług poza `caddy` nie ma znaczenia.
- `roles/*/tasks/main.yml` — logika; `roles/*/templates/*.j2` — konfiguracja
  generowana Jinja2; `roles/*/handlers/main.yml` — restart/reload po zmianie.
- Zmienne domyślne (`domain_name`, `ansible_user`, IP urządzeń,
  `vaultwarden_dir`, `vaultwarden_version`, `homepage_dir`,
  `homepage_version`, `portfolio_dir`, `portfolio_repo_url`,
  `blocky_version`, `monitoring_dir`, `prometheus_version`,
  `node_exporter_version`, `cadvisor_version`, `grafana_version`,
  `cloudflared_dir`, `cloudflared_version`) żyją w `group_vars/all/vars.yml`
  (nieobecny w repo, tylko `.example`). Sekrety — w
  `group_vars/core_nodes/vault.yml` (zaszyfrowany Ansible Vault).
- Rola `portfolio` ma scenariusz Molecule
  (`roles/portfolio/molecule/default/`: create/converge/idempotence/verify/
  destroy), uruchamiany lokalnie przez `cd roles/portfolio && molecule test`
  oraz automatycznie w CI (job `molecule` w `.github/workflows/lint.yml`).
  Pozostałych siedem ról NIE jest pokrytych Molecule — wymagałoby to
  docker-in-docker (`community.docker.docker_compose_v2` potrzebuje demona
  Dockera wewnątrz kontenera testowego) albo obrazu z działającym systemd
  i D-Bus (dla firewalld w rolach `docker`, `blocky`, `caddy`). Dla tych ról
  jedyna realna weryfikacja przed produkcją to nadal `vagrant up`/
  `vagrant provision` (patrz `Vagrantfile`). Nie zakładaj, że pokrycie
  Molecule jest szersze niż opisane tutaj.

## Pułapki tego repo

- **`vars.yml` vs `vars.yml.example` się rozjeżdżają.** Aktualizacja
  `group_vars/all/vars.yml.example` NIE propaguje się do realnego,
  ignorowanego przez git `group_vars/all/vars.yml` na kontrolerze. Jawnie
  wpisana tam stara wartość przykrywa nowy `| default(...)` w szablonie.
  Przy każdej zmianie domyślnej wersji obrazu sprawdź też realny plik na
  kontrolerze, nie tylko `.example`. To realnie zablokowało wdrożenie —
  próba pobrania nieistniejącego `ghcr.io/google/cadvisor:v0.49.1`.
- **cAdvisor: wersja i rejestr są przypięte celowo, nie "porządkuj" ich.**
  `cadvisor_version` domyślnie `v0.60.5`, obraz z `ghcr.io/google/cadvisor`
  (nie `gcr.io`). Docker na hoście używa containerd snapshottera, a
  cAdvisor sprzed `v0.54.0` nie obsługuje tego układu przechowywania warstw
  (upstream issue #3643, fix w PR #3709). Rejestr zmienił się z `gcr.io` na
  `ghcr.io` przy `v0.53.0`. Nie cofaj wersji ani rejestru bez sprawdzenia
  tych numerów.
- **node_exporter bez `network_mode: host` kłamie o sieci.** Raportuje
  interfejsy własnego namespace'u kontenera, nie hosta — potwierdzone
  empirycznie porównaniem z `/proc/net/dev` na hoście. Montowanie `/proc`
  z `--path.procfs` tego NIE naprawia, bo `/proc/net` jest per-namespace,
  nie per-mount. Dlatego panel sieciowy hosta został usunięty z dashboardu
  Grafany (`node-exporter.json`). Metryki CPU/RAM/dysk pozostają dokładne.
- **Przebudowa obrazu Caddy pod tym samym tagiem osieroca kontener.** Stary
  kontener wciąż wskazuje na osierocony sha, więc `docker compose` wywala
  się przy `images`/`up` błędem `No such image`. Naprawa:
  `docker compose down && docker compose up -d` w `/opt/caddy`.
- **`cloudflared`: `localhost` w ingress wskazuje na kontener cloudflared,
  nie na hosta.** `cloudflared` żyje we własnym kontenerze podpiętym do
  sieci `caddy-ingress`, więc `service:` w `config.yml` MUSI wskazywać na
  Caddy po nazwie kontenera (`http://caddy:80`), nigdy `localhost`/`127.0.0.1`
  — to by wskazywało na sam kontener cloudflared, który niczego nie
  serwuje. Ta sama pułapka co gdyby ktoś próbował `network_mode: host`
  dla node_exportera, tylko w drugą stronę.
- **Rola `cloudflared` wymaga jednorazowego ręcznego bootstrapu poza
  Ansible.** Tunel po stronie Cloudflare (`cloudflared tunnel login` →
  `tunnel create` → `tunnel route dns`) trzeba założyć ręcznie z CLI, zanim
  playbook pierwszy raz uruchomi tę rolę — `vault_cloudflared_tunnel_id` i
  `vault_cloudflared_credentials_json` w `group_vars/core_nodes/vault.yml`
  to dane z tego kroku, rola ich nie generuje ani nie tworzy tunelu.
- **Molecule dla roli leżącej w `roles/` obok innych ról nie znajdzie jej
  bez jawnego `ANSIBLE_ROLES_PATH`.** Domyślna ścieżka przeszukiwania ról
  Ansible nie obejmuje katalogu nadrzędnego scenariusza testowego, więc
  `include_role: name: portfolio` w `converge.yml` wywala się błędem
  "role not found", mimo że rola fizycznie istnieje obok. Naprawa: w
  `provisioner.env` w `molecule.yml` ustawić
  `ANSIBLE_ROLES_PATH: ${MOLECULE_PROJECT_DIRECTORY}/..` (patrz
  `roles/portfolio/molecule/default/molecule.yml`).

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

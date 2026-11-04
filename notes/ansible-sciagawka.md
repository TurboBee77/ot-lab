# Ściągawka Ansible

> Komendy i pojęcia pogrupowane tematycznie + osobna sekcja z tym, czego
> faktycznie użyliśmy w tym projekcie (i dlaczego). Kontekst: control node
> to zawsze VM Linux (nigdy Windows — patrz `CLAUDE.md`), projekt w
> `fastapi-devops-infra/ansible/`.

---

## 1. Uruchamianie

```
ansible-playbook site.yml
```
Odpal z wnętrza katalogu `ansible/` (tam gdzie `ansible.cfg`) — ścieżki
do inventory i ról w `ansible.cfg` są względne od bieżącego katalogu.

```
ansible-playbook --syntax-check site.yml
```
Waliduje YAML i nazwy modułów **w całym drzewie** — nie tylko `site.yml`,
ale też wszystkie taski ról, do których się on odwołuje (`roles: [docker,
ufw]` → wchodzi w `roles/docker/tasks/main.yml` i `roles/ufw/tasks/main.yml`).
Nie łączy się po SSH — nie złapie błędów połączenia ani nieistniejących
zmiennych w `{{ }}` (te wychodzą dopiero przy realnym uruchomieniu).

```
ansible <hosts> -m <moduł> -a "<argumenty>"
```
Komenda **ad-hoc** — jedno zadanie bez pisania playbooka. Przykład użyty
w projekcie:
```
ansible all -m ansible.builtin.command -a "docker ps"
ansible all -m ansible.builtin.command -a "ufw status verbose"
```
- `all` — wzorzec hostów z inventory (wbudowana grupa = wszystkie hosty;
  można też podać konkretną grupę: `ci`, `app`, `monitoring`)
- `-m` — który moduł wykonać (ten sam koncept co moduł w tasku playbooka)
- `-a` — argumenty dla modułu; dla `command`/`shell` to samo polecenie

**`ansible.builtin.command` vs `ansible.builtin.shell`:** `command`
uruchamia proces bezpośrednio, bez `/bin/sh` — nie rozumie pipe'ów (`|`),
przekierowań (`>`), zmiennych środowiskowych (`$VAR`) ani globbingu (`*`).
Bezpieczniejszy domyślny wybór; `shell` dopiero gdy faktycznie potrzebujesz
tych funkcji.

```
ansible-lint roles/docker roles/ufw
```
Lintuje role **bezpośrednio**, bez potrzeby playbooka, inventory ani
hostów — dobry pierwszy, szybki filtr przed czymkolwiek innym. Wymaga
osobnej instalacji (`pip install ansible-lint`), nie jest częścią
`ansible-core`.

---

## 2. Struktura projektu i auto-discovery

Ansible **sam wykrywa** poniższe pliki/katalogi po nazwie i lokalizacji —
nie trzeba ich nigdzie jawnie importować (inaczej niż np. `vars_files`).

```
ansible.cfg
```
Konfiguruje **zachowanie narzędzia CLI** (gdzie szukać inventory/ról, czy
tworzyć pliki `.retry`) — niezależnie od tego, jakie hosty się ma. Ścieżki
w środku (`inventory = inventory/hosts.yml`, `roles_path = roles`) są
względne od katalogu, z którego odpalasz komendę.

```
inventory/hosts.yml
```
Lista hostów pogrupowana logicznie. **W tym projekcie generowany
skryptem** (`scripts/generate_inventory.py`), nie edytowany ręcznie —
Elastic IP dostaje nową alokację przy każdym `terraform destroy`→`apply`,
więc statyczny, ręcznie wpisany plik dezaktualizowałby się po każdej
sesji. Plik wynikowy w `.gitignore` (artefakt generowany, nie źródło
prawdy — `terraform output` jest źródłem prawdy).

```
group_vars/<nazwa_grupy>.yml
group_vars/all.yml
host_vars/<nazwa_hosta>.yml
```
Zmienne automatycznie doklejane do hostów należących do danej grupy
(albo do konkretnego hosta). `all` to specjalna, niejawna grupa — zawsze
istnieje, zawsze obejmuje każdy host z inventory. Precedencja: `host_vars`
> `group_vars/<konkretna_grupa>` > `group_vars/all`.

Zmienne z prefiksem `ansible_*` (np. `ansible_user`, `ansible_become`,
`ansible_ssh_common_args`) są **zarezerwowane** — silnik Ansible czyta je
bezpośrednio, żeby zdecydować jak połączyć się z hostem, w odróżnieniu od
zwykłych zmiennych użytkownika (jak `ufw_allowed_ports`), które mają
znaczenie tylko w Twoich własnych taskach/szablonach.

```
roles/<rola>/tasks/main.yml
roles/<rola>/defaults/main.yml
roles/<rola>/handlers/main.yml
```
Ustandaryzowana struktura roli. `defaults/main.yml` to punkt rozszerzenia —
wartości domyślne, które grupy/hosty mogą nadpisać bez ruszania kodu roli
(przykład z projektu: `ufw_allowed_ports: []`, nadpisywane per-rola w
kolejnych etapach).

---

## 3. Moduły i idempotencja

Pełna nazwa modułu (**FQCN**) zamiast skrótu: `ansible.builtin.apt` zamiast
`apt`, `community.general.ufw` zamiast `ufw`. Jednoznacznie wiadomo z
jakiej kolekcji pochodzi moduł — istotne bo `community.general` **nie**
jest częścią `ansible-core`, trzeba ją doinstalować osobno
(`ansible-galaxy collection install community.general`).

**Idempotencja = drugie uruchomienie tego samego playbooka na już
skonfigurowanym hoście nie zgłasza `changed`.** Dedykowane moduły
(`apt`, `service`, `ufw`, `user`, `file`) same sprawdzają aktualny stan
przed działaniem. Kontrast: `shell`/`command` wykonują się bezwarunkowo za
każdym razem — Ansible nie ma pojęcia, czy coś faktycznie się zmieniło,
chyba że ręcznie dodasz `creates:`/`changed_when:`. Dlatego w tym projekcie
konsekwentnie unikamy `shell`/`command` w rolach na rzecz modułów
dedykowanych.

---

## 4. Pułapki YAML

Niecytowane wartości, które *wyglądają* jak co innego niż string, są
parsowane jako co innego niż string:

```yaml
mode: "0755"        # bez cudzysłowu: niejednoznaczna liczba ósemkowa
logging: "on"        # bez cudzysłowu: boolean True, nie string "on"
```
YAML (a konkretnie parser którego używa Ansible) interpretuje niecytowane
`on`/`off`/`yes`/`no`/`true`/`false` jako **boolean**. Jeśli moduł
oczekuje konkretnego stringa z listy wyboru (jak `logging` w module
`ufw`: `on`/`off`/`low`/`medium`/`high`/`full`), dostanie zamiast tego
`True`/`False` i się wywali albo zachowa nieprzewidywalnie. Zasada: gdy
wartość mogłaby zostać źle zinterpretowana — cytuj ją jawnie.

---

## 5. `ufw` (moduł `community.general.ufw`) — specyfika

Dwa różne mechanizmy samego `ufw`, odzwierciedlone jako dwie różne
kombinacje parametrów modułu:

```yaml
# polityka domyślna — dla całego kierunku ruchu, brak wyjątków
- community.general.ufw:
    direction: incoming
    policy: deny

# konkretna reguła-wyjątek — nadpisuje politykę domyślną dla dopasowanego ruchu
- community.general.ufw:
    rule: allow
    port: "22"
    proto: tcp
```
`ufw` sprawdza najpierw konkretne reguły, dopiero gdy żadna nie pasuje —
politykę domyślną. Stąd `policy: deny` na `incoming` + osobna
`rule: allow, port: 22` = "domyślnie blokuj wszystko, ale zrób wyjątek dla
portu 22".

**Kolejność w pliku ma znaczenie tylko dla jednego taska — `enable`.**
`ufw enable` to jedyny moment, w którym firewall faktycznie zaczyna
egzekwować reguły — wszystkie wcześniejsze taski tylko układają
konfigurację w tle. `enable` musi więc być **ostatni** w roli: gdyby był
pierwszy, domyślna polityka `deny incoming` zablokowałaby ruch (w tym
Twoje własne SSH) zanim reguła zezwalająca na port 22 zdążyłaby powstać.

`name: OpenSSH` zamiast `port: "22", proto: tcp` też działa (to profil
aplikacji z `/etc/ufw/applications.d/`, instalowany razem z
`openssh-server`) — ale jest to reguła *pośrednia*, zależna od istnienia
tego profilu. W tym projekcie świadomie zostaliśmy przy jawnym porcie,
spójnie z resztą configu (nic innego też nie polega na niejawnej
zawartości AMI).

---

## 6. `ansible-vault` — sekrety w repo

Szyfruje plik tak, by bezpiecznie trzymać go w gicie (bezużyteczny bez
hasła do Vaulta, które nigdy nie trafia do repo). W tym projekcie: jeden
wspólny Vault password na cały projekt, po jednym `vars/main.yml` per
rola `deploy-*` (`deploy-app`, `deploy-jenkins`).

```
ansible-vault view <plik>
```
Odszyfrowuje i wypisuje w terminalu, **bez zapisu na dysk** — bezpieczny
wybór do samego podglądu wartości.

```
ansible-vault edit <plik>
```
Otwiera odszyfrowaną treść w `$EDITOR`, po zapisaniu z powrotem szyfruje
plik na dysku. Użyj, gdy chcesz zmienić wartość.

```
ansible-vault encrypt <plik>
```
Szyfruje plik pierwszy raz (np. świeżo skopiowany z `vars/main.yml.example`).

**Pułapka: `ansible-vault decrypt <plik>` bez `--output=-` trwale
nadpisuje plik na dysku odszyfrowaną treścią.** Dla pliku trzymanego
zaszyfrowanego w repo to realne ryzyko — przypadkowy `git add`/`commit`
po takim `decrypt` wrzuciłby plaintext sekretów do historii gita. `view`
nie ma tego ryzyka, bo nic nie zapisuje — domyślny wybór do podglądu.

Hasło do Vaulta: interaktywnie przez `--ask-vault-pass`, albo
`--vault-password-file <plik>` (plik z samym hasłem, spoza repo) — to
samo hasło działa dla wszystkich zaszyfrowanych plików w projekcie,
niezależnie od tego, którą rolę/etap dotyczą.

**Gdzie żyją wartości sekretów — tylko w `roles/<rola>/vars/main.yml`
(zaszyfrowany), nigdy w `group_vars/*.yml`.** `group_vars` jest zwykłym,
niezaszyfrowanym, śledzonym przez gita plikiem — ma wyłącznie
**referencjonować** zmienne Vaulta przez `{{ vault_xxx }}` (jak w
`jenkins_secrets:` tego projektu), nigdy przypisywać im wartość wprost.
Złapane na żywo: przypadkowe dopisanie `vault_jenkins_admin_password: "..."`
bezpośrednio do `group_vars/ci.yml` (na VM, więc bez ryzyka trafienia do
historii gita — ale gdyby to był plik commitowany, byłby to plaintext
sekret w repo).

**Pułapka przy debugowaniu: `no_log: true` na tasku maskuje też komunikat
błędu, nie tylko normalny output.** Task z `no_log: true` (świadomie
ustawione w tym projekcie na kopiowaniu sekretów, żeby wartości nie
leciały do logów Jenkinsa/Ansible) przy failu pokazuje tylko
`"censored": "the output has been hidden..."` — zero informacji o
przyczynie. Trzeba wtedy rozumować z zewnątrz: sprawdzić czy plik Vault
istnieje, czy playbook dostał `--ask-vault-pass`, czy nazwy zmiennych w
`group_vars`/`vars/main.yml` się zgadzają — zamiast liczyć na treść błędu.

---

## 7. `delegate_to` i widoczność zmiennych między hostami

```yaml
- name: Zapisz coś na innym hoście niż ten, dla którego leci play
  ansible.builtin.copy:
    content: "{{ ansible_host }}"
    dest: /gdzies/na/innym/hoscie
  delegate_to: "{{ groups['ci'][0] }}"
```
`delegate_to` przekierowuje **wykonanie pojedynczego taska** na inny host
niż ten, po którym iteruje bieżący `hosts:` w play — ale zmienne bieżącego
hosta (jak `ansible_host`) zostają dostępne. Różni się od zmiany całego
`hosts:` play — tu zmienia się tylko jeden task, w środku pętli po hostach
z oryginalnego play.

`groups['<nazwa_grupy>']` — wbudowana zmienna, lista nazw hostów należących
do danej grupy z inventory. W tym projekcie hosty nazywają się
`<rola>-VM` (`ci-VM`, `app-VM`, generowane przez
`scripts/generate_inventory.py`), nie tak samo jak grupa (`ci`, `app`) —
`delegate_to: ci` nie zadziała (nie ma hosta dosłownie nazwanego `ci`),
`groups['ci'][0]` dynamicznie rozwiązuje się na `ci-VM`, odporne na
ewentualną zmianę konwencji nazewnictwa w przyszłości.

**Pułapka: zmienne z `defaults/main.yml` roli nie zawsze są widoczne przez
`hostvars` innego hosta.** Trafiają do `hostvars` danego hosta dopiero gdy
rola faktycznie wykona się dla niego **w bieżącym przebiegu** playbooka.
Jeśli odpalasz `ansible-playbook site.yml --limit app`, a play
`hosts: ci` (z rolą, która definiuje potrzebną zmienną w swoich
`defaults/`) w ogóle się nie wykonuje — `hostvars[groups['ci'][0]]['ta_zmienna']`
będzie niezdefiniowane, mimo że host `ci-VM` istnieje w inventory.
`group_vars/<grupa>.yml` nie ma tego problemu — ładowane są dla wszystkich
hostów zawsze, niezależnie od `--limit` i od tego, które playe faktycznie
się wykonują. Zastosowane w tym projekcie: `jenkins_home` przeniesione z
`roles/deploy-jenkins/defaults/main.yml` do `group_vars/ci.yml`, żeby
`deploy-app` (`delegate_to` na `ci`) mogło się do niego odwołać nawet przy
`--limit app`.

---

## 8. `hostvars` bez `delegate_to` — kiedy wystarczy samo inventory

```yaml
targets: ["{{ hostvars[groups['app'][0]]['ansible_host'] }}:9100"]
```

Sekcja 7 opisuje pułapkę: zmienne z `defaults/main.yml` roli trafiają do
`hostvars` innego hosta dopiero gdy jego play faktycznie się wykonał w
danym przebiegu. **`ansible_host` tej pułapki nie ma** — pochodzi wprost
z pliku inventory (wygenerowanego przez `scripts/generate_inventory.py`),
czyli jest znane Ansible od razu przy starcie playbooka, zanim jakikolwiek
play w ogóle zacznie się wykonywać. Dlatego szablon `prometheus.yml.j2`
(rola `deploy-monitoring`, `hosts: monitoring`) może swobodnie odczytać
`ansible_host` hosta z grupy `ci`/`app`, mimo że nie ma dla nich żadnego
`delegate_to` ani wspólnego play — w odróżnieniu od zmiennej typu
`jenkins_home` (zdefiniowanej w `defaults/` roli, nie w inventory), która
tego wymagała.

Zasada ogólna: **dane z inventory** (`ansible_host`, grupy, `host_vars`/
`group_vars`) są w `hostvars` zawsze i od razu; **fakty zbierane w
trakcie play** (`ansible_facts`, zmienne z `defaults/` roli, które
"aktywują się" dopiero gdy rola faktycznie wykona się dla danego hosta)
zależą od tego, co się realnie wykonało w bieżącym przebiegu.

---

## 9. `docker_container` z `network_mode: host`

```yaml
- community.docker.docker_container:
    name: node-exporter
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
```

W trybie `host` kontener dzieli stos sieciowy hosta wprost — port, na
którym nasłuchuje proces w środku, **jest** tym samym portem na hoście,
bez żadnego mapowania. Dlatego w tasku **nie ma** `ports:` — Docker i tak
by je zignorował w tym trybie. Używane tam, gdzie kontener ma czytać/
udostępniać dane samego hosta (tu: node_exporter i jego metryki
CPU/RAM), nie żyć w odizolowanej sieci Dockera.

Bind mounty `:ro` obok tego to osobna sprawa — `network_mode: host`
daje dostęp do sieci hosta, ale kontener wciąż ma **swój własny**,
odizolowany `/proc`/`/sys` (dotyczący tylko procesów/zasobów widocznych
z jego namespace'u). Żeby zobaczyć prawdziwe dane hosta, trzeba je
jawnie podmontować pod inną ścieżką (`/host/proc`) i wskazać to
programowi wewnątrz (`--path.procfs=/host/proc`).

---

## 10. Historia komend/decyzji użytych w tym projekcie (skrót)

| Sytuacja | Komenda / decyzja |
|---|---|
| Sprawdzenie zależności `community.general` | `ansible-galaxy collection list \| grep community.general` |
| Wygenerowanie inventory przed każdym uruchomieniem | `python3 scripts/generate_inventory.py` |
| Walidacja składni przed realnym uruchomieniem | `ansible-playbook --syntax-check site.yml` |
| Pierwsze uruchomienie (instalacja + konfiguracja) | `ansible-playbook site.yml` |
| Test idempotencji (warunek "gotowe" Etapu 2) | `ansible-playbook site.yml` (drugi raz — oczekiwane `changed=0`) |
| Weryfikacja Dockera bez playbooka | `ansible all -m ansible.builtin.command -a "docker ps"` |
| Weryfikacja ufw bez playbooka | `ansible all -m ansible.builtin.command -a "ufw status verbose"` |
| Literówka złapana przez `--syntax-check` | `community.genereal.ufw` → `community.general.ufw` (brakująca litera) |
| Literówka złapana dopiero przy realnym uruchomieniu | `tulle: deny` → `policy: deny` (`--syntax-check` tego nie łapie — to poprawny YAML, tylko nieistniejący parametr modułu, wykryty dopiero przy walidacji argumentów modułu na hoście) |
| Kolekcje instalowane jawnie (bez `requirements.yml`) w obrazie Docker, poza control node | `ansible-galaxy collection install community.docker ansible.posix community.general` — musi się zgadzać z `requirements.yml`, inaczej gap ujawni się dopiero przy realnym użyciu brakującego modułu |
| Zapis pliku na innym hoście niż bieżący w play (`hosts: app`) | `delegate_to: "{{ groups['ci'][0] }}"` (sekcja 7 wyżej) |
| Test idempotencji po zmianie `pull` w `docker_compose_v2` | dwa przebiegi `--limit app` z rzędu — drugi musi dać `changed=0`, `pull: always` psuje to niezależnie od realnej zmiany treści obrazu |

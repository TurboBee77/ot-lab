# Przewodnik promptowania — Projekt DevOps

> Dokument pomocniczy do pracy z Claude nad kolejnymi etapami projektu.
> Trzymaj razem z `DevOpsProj.md`. Cel: każdy nowy wątek zaczyna się
> szybko, trzyma ustalony zakres i faktycznie buduje zrozumienie,
> a nie tylko generuje gotowy kod do wklejenia.

---

## 0. Zanim zaczniesz nowy wątek — checklist

- [ ] Wiem, który **Etap** z harmonogramu (`DevOpsProj.md`) realizuję teraz
- [ ] Mam załączony `DevOpsProj.md` (zawsze, w każdym nowym wątku)
- [ ] Jeśli poprzedni etap coś zmienił w założeniach — zaktualizowałem plik
      PRZED otwarciem nowego wątku, nie w jego trakcie
- [ ] Wiem, czy to praca **planistyczna** (decyzje, wybory) czy
      **produkcyjna** (kod, konfiguracja) — różne szablony poniżej

**Zasada:** jeden wątek = jeden etap (lub ściśle powiązana para etapów,
np. 5+6 bo współdzielą Jenkinsfile). Nie ciągnij jednego wątku przez
cały projekt — kontekst rośnie, a Ty tracisz kontrolę nad tym, co Claude
"pamięta" i zakłada.

---

## 1. Szablon promptu startowego (nowy etap, praca produkcyjna)

Użyj przy przechodzeniu do kodu/konfiguracji dla konkretnego etapu.

```
Załączam ustalone założenia projektu (DevOpsProj.md) - punkt odniesienia,
nie do renegocjacji w tym wątku.

Realizuję Etap [NUMER]: [nazwa etapu z harmonogramu].
Deliverable z pliku: [wklej z sekcji "Deliverable"]
Warunek "gotowe": [wklej z sekcji "Gotowe, gdy"]

Mój poziom: znam teoretycznie narzędzia, podstawowa praktyczna styczność.
Zależy mi na zrozumieniu, nie na gotowcu do wklejenia bez czytania.

Zasady pracy w tym wątku:
- Jeśli coś wymaga mojej decyzji (nazwa, konwencja, wybór między
  równoważnymi opcjami) - zapytaj, nie zgaduj
- Wyjaśniaj DLACZEGO dany fragment kodu/konfiguracji wygląda tak, a nie
  inaczej - szczególnie tam, gdzie teoria rozjeżdża się z praktyką
- Buduj krok po kroku, nie wrzucaj całości na raz - chcę rozumieć
  każdy fragment zanim przejdziemy dalej
- Jeśli coś odbiega od założeń w DevOpsProj.md, zatrzymaj się i zapytaj,
  zanim to zaimplementujesz

Zacznij od: [pierwszy konkretny krok, np. "struktura katalogów dla tego etapu"]
```

---

## 2. Szablon promptu planistycznego (decyzja/wybór do podjęcia)

Użyj, gdy trzeba podjąć decyzję architektoniczną w trakcie etapu
(tak jak przy Kubernetesie/SSL wcześniej).

```
W ramach Etapu [NUMER] pojawiła się decyzja do podjęcia: [opisz w 1 zdaniu].

Kontekst z DevOpsProj.md: [co jest istotne z założeń]

Nie rekomenduj jeszcze konkretnej opcji. Pokaż mi:
1. Realne opcje (nie więcej niż 3)
2. Trade-offy każdej pod kątem: (a) zgodność z kryteriami oceny/wagami,
   (b) nakład czasu przy moim poziomie, (c) ryzyko dla harmonogramu
3. Czy któraś opcja wymaga zmiany czegoś, co już ustaliliśmy

Ja decyduję na końcu.
```

---

## 3. Szablon zamknięcia etapu (przed przejściem dalej)

Warto zrobić na koniec każdego etapu — buduje historię i wymusza
faktyczne zrozumienie, zanim pójdziesz dalej.

```
Zamykamy Etap [NUMER]. Zanim przejdę dalej:

1. Zadaj mi 2-3 pytania sprawdzające rozumienie kluczowych decyzji
   z tego etapu (np. dlaczego coś jest skonfigurowane w ten sposób)
2. Podsumuj w 3-5 punktach co powstało i czy spełnia warunek "gotowe"
   z DevOpsProj.md
3. Jeśli coś w trakcie pracy odbiegło od pierwotnych założeń - wypisz
   to, żebym zaktualizował DevOpsProj.md przed nowym wątkiem
```

---

## 4. Zasady stałe (obowiązują w każdym wątku)

- **Nie upraszczaj** wyjaśnień do poziomu podstaw terminologii — zakładaj
  znajomość teorii, skupiaj się na praktyce i pułapkach
- **Priorytet wag punktowych** (z kryteriów oceny) > estetyka/"ładne
  rozwiązanie" — jeśli coś ma wagę 1, nie inwestuj w to ponad podstawę
- **Historia commitów ma znaczenie** — proś o commitowanie krokowo
  i opisowo, nie o gotowe bloki do jednego zbiorczego commita
- **Dokumentacja MD na bieżąco**, nie hurtowo na końcu etapu (dotyczy również plików w projektach) zwłaszcza infrastruktury

---

## 5. Log zmian założeń

> Aktualizuj przy każdej zmianie w `DevOpsProj.md`, żeby wiedzieć,
> kiedy i dlaczego coś się zmieniło.

| Data | Etap | Co się zmieniło | Dlaczego |
|---|---|---|---|
| 2026-07-22 | 0 / 5 / 6 | Sekcja 5: dopisano, że fork appki ma gałąź główną `master`, repo infra `main`. Etap 6: warunek `when` w Jenkinsfile zmieniony z `branch main` na `branch 'master'` | Ujawnione przy zamknięciu Etapu 0 (patrz `etap-0-podsumowanie.md`) — fork odziedziczył `master` z oryginalnego repo i świadomie nie był przemianowywany; bez tej poprawki Etap 6 użyłby błędnej nazwy gałęzi |
| 2026-07-23 | 1 / 2 (architektura) | Sekcja 2: VirtualBox → AWS EC2 (domyślne VPC, Elastic IP, `t3.micro`/`t2.micro` Free Tier). Sekcja 4: nowa 4-kolumnowa tabela zakresu, Ansible/Prometheus/Grafana przeniesione z "obowiązkowe" do "priorytetowe (opcjonalne)". Etap 1: przepisany na EC2, usunięty Plan B (Vagrant). Etap 9: dopisane zdanie o nietwardości checkpointu dla Ansible/monitoringu | Przy projektowaniu struktury Terraform dla Etapu 1 okazało się, że `network_adapter.ipv4_address` providera `terra-farm/virtualbox` jest computed-only — nie da się ustawić statycznego IP. Przy okazji ponowna lektura `Punktacja.txt` ujawniła, że Ansible/Prometheus/Grafana są tam opcjonalne, nie obowiązkowe, jak wcześniej założono w `DevOpsProj.md` |
| 2026-07-26 | 3 | Sekcja 2: dopisano, że idempotencja to wymóg ogólny całego IaC (nie tylko warunek "gotowe" Etapu 2), i że role `deploy-*` są rozdzielone per VM, nie jedną wspólną rolą z `when` | Przy budowie roli `deploy-app` przyjąłem błędnie, że idempotencja obowiązuje tylko tam, gdzie `DevOpsProj.md` explicite ją wymienia (Etap 2) — użytkownik to zakwestionował, a `Wymagania Projektu Dyplomowego.md` (nadrzędne źródło) mówi to wprost i ogólnie. Naprawiono rolę: klon appki do podkatalogu `src/` (zamiast nadpisywać trackowany plik appki) + `community.docker.docker_compose_v2` (`build: policy`) zamiast `command` bez wykrywania stanu |
| 2026-07-28 | 4 → 5/6 | Deliverable Etapu 4 ("skonfigurowane credentiale: SSH do VM2, login Docker Hub") świadomie **nie** zrealizowany w Etapie 4 — przesunięty do Etapu 5 (Docker Hub, potrzebny od startu pipeline'u CI) / Etapu 6 (SSH do VM2, potrzebny dopiero w stage `deploy`). Etap 4 mimo to oznaczony jako zamknięty | Warunek "gotowe" Etapu 4 (test "Hello World") nie testuje credentiali, więc traktuję je jako pracę do zrobienia dopiero gdy faktycznie potrzebna, nie jako blokadę zamknięcia etapu. Uwaga proceduralna: pierwsza wersja `etap-4-podsumowanie.md` błędnie oznaczyła etap jako w pełni "ZAMKNIĘTY" mimo tej luki, bez jawnej decyzji użytkownika — poprawione dopiero po tym, jak użytkownik to wychwycił ("przekłamujesz dokumentację"). Wniosek na przyszłość: nie domykać etapu z niekompletnym deliverable bez wyraźnej, udokumentowanej zgody użytkownika na przesunięcie zakresu |
| 2026-07-29 | 5 (architektura) | Sekcja 2: typ instancji EC2 zmieniony z `t3.micro`/`t2.micro` na **`t3.small`** dla wszystkich trzech VM (nie tylko `ci`) — wspólna zmienna `var.instance_type`, osobna zmienna per-VM rozważona, ale niewdrożona | `ci` (Jenkins) dwukrotnie stał się nieresponsywny pod obciążeniem pipeline'u CI (JVM Jenkinsa + demon Dockera + kontenery testowe + build obrazu jednocześnie, na 1GB RAM `t3.micro`, bez domyślnego swapu na Ubuntu/EC2) — prawdopodobny OOM, potwierdzony przez failujący "Instance status check" w konsoli AWS i `sshd` nieodpowiadający na poziomie protokołu mimo działającego TCP. `t3.small` (2GB RAM) sprawdzone w konsoli AWS jako "Free tier eligible" dla tego konta — inny/szerszy model Free Tier niż klasyczny "750h/mies. tylko t2/t3.micro" założony pierwotnie w tym dokumencie; wymaga samodzielnej weryfikacji przez Billing → Free Tier, nie tylko etykiety w konsoli |
| 2026-07-29 | 5 (testy) | Doprecyzowanie założenia z sekcji 4 ("unit testy pytest już w forku appki — uruchamiane w ramach CI, bez dodatkowego nakładu") — nakład jednak był, tylko innego rodzaju | `tests/conftest.py`'s `FakeAsyncPGPool` okazał się owijać **prawdziwe** połączenie `asyncpg.create_pool()` (izolacja per-test przez rollback transakcji, nie mock/stub) — testy tej appki wymagają realnej, żywej bazy Postgres z odpaloną migracją Alembic, nie da się ich uruchomić jako gołego `RUN pytest` wewnątrz `docker build` (brak dostępu do sąsiedniego kontenera bazy w trakcie budowania obrazu). Wymagało to osobnej orkiestracji (`docker-compose.test.yml` + `docker compose up` w Jenkinsfile), nie tylko zbudowania obrazu z pytest w środku |
| 2026-08-02 | 6 (SSH CD) | Klucz SSH Jenkins→VM2: **dedykowany, nie reużyty** klucz projektowy z Etapu 1 (`fastapi-devops-key`) | Świadoma decyzja o zawężeniu blast radius — kompromitacja Jenkinsa (kontener z zamontowanym `docker.sock`, już zaakceptowane ryzyko z Etapu 4) nie powinna dawać dostępu tym samym kluczem, którym operator ręcznie zarządza wszystkimi trzema VM. Publiczna część dopisywana automatycznie do `authorized_keys` na VM2 przez rolę `deploy-app` (`ansible.posix.authorized_key`, idempotentnie) |
| 2026-08-02 | 6 (pojemność) | Sekcja 2: root EBS `ci`/`app` sparametryzowany (`ci_root_volume_size`, `app_root_volume_size`), domyślnie 8GB — za mało dla obu pod pełnym obciążeniem | Trzy oddzielne incydenty `no space left on device` na `ci` w trakcie tego etapu (pełny pakiet `ansible` zamiast `ansible-core`, nieczyszczony Docker build cache BuildKita, sam rozmiar wolumenu). `app` miał to już wcześniej zahardkodowane na `12` (nieudokumentowane w tym pliku do tej pory), `ci` nie miał parametru wcale — szczegóły: `etap-6-podsumowanie.md`, `TODO.md` (cron do sprzątania build cache, niewdrożony) |
| 2026-08-05 | 7 (sieć) | Sekcja 2: druga pula portów SG ograniczona do CIDR domyślnego VPC (`internal_ingress_ports`/`internal_cidr`, nowa `data "aws_vpc" "default"`), nie tylko `extra_ingress_ports`/`ssh_allowed_cidr` publiczne jak dotąd. Ruch wewnętrzny (Prometheus → node_exporter) musi iść po **prywatnym** IP instancji (`private_ip`, nowy output), nie po publicznym Elastic IP (`ansible_host`) | Node_exporter (9100) i UI Prometheusa (9090) nie mają auth — świadoma decyzja, żeby nie wystawiać ich do całego internetu, w odróżnieniu od dotychczasowego (Etap 4/6) wzorca "wszystkie dodatkowe porty na `ssh_allowed_cidr`". Przy pierwszej próbie scrape'owania po publicznym IP: reguła SG z CIDR VPC nie dopasowywała się do ruchu między dwiema EC2 w tym samym VPC, jeśli szedł przez publiczny adres celu (`context deadline exceeded`, cichy drop) — naprawione przez dodanie `private_ip` i przełączenie `prometheus.yml.j2` na ten hostvar dla node_exporterów |

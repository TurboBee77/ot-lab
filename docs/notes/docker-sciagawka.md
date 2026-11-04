# Ściągawka Docker

> Komendy i pojęcia pogrupowane tematycznie + osobna sekcja z tym, czego
> faktycznie użyliśmy w tym projekcie (i dlaczego). Zaczęta dopiero w
> Etapie 6, mimo że Docker jest w projekcie od Etapu 1 (waga 6 w
> `Punktacja.txt`) — nadrabiane od teraz, nie backfillowane wstecz.

---

## 1. Diagnostyka miejsca na dysku

```
docker system df
```
Skrót zajętości: `Images`, `Containers`, `Local Volumes`, `Build Cache` —
z kolumną `RECLAIMABLE` (ile realnie odzyskasz sprzątając). W tym
projekcie kluczowe odkrycie: `Build Cache` bywa największą, w 100%
odzyskiwalną pozycją, mimo że nic go jawnie nie tworzy — dokłada go każdy
`docker build`/`docker compose ... up --build`.

```
docker system prune -af
```
Usuwa **wszystkie nieużywane** obrazy (nie tylko dangling), zatrzymane
kontenery, nieużywane sieci i **cały build cache**. `-a` = agresywnie
(bez tej flagi usuwa tylko dangling); bez `--volumes` (domyślnie) **nie
rusza wolumenów** — bezpieczne względem named volumes typu `jenkins_home`.

```
docker builder prune -af
```
Węższa wersja — tylko build cache, zostawia obrazy/kontenery w spokoju.
Lepsza, gdy chcesz odzyskać miejsce, ale świadomie zachować już zbudowane,
otagowane obrazy.

**Pułapka: `docker compose down -v --rmi local` (już używane w tym
projekcie jako `post always` po teście CI) nie czyści build cache.**
Usuwa tylko: kontenery i wolumeny **danego compose projektu**, oraz obraz
z `--rmi local` (czyli zbudowany lokalnie, otagowany dla tego projektu).
Build cache BuildKita żyje osobno, celowo przetrwa między buildami (żeby
kolejne budowanie appki było szybsze — widać to jako `CACHED` w logu
buildu) i rośnie bezustannie, jeśli nic go nie sprząta. To był główny,
najmniej oczywisty z trzech powodów powtarzających się `no space left on
device` w Etapie 6 — szczegóły `etap-6-podsumowanie.md`.

---

## 2. Multi-stage build i `--target`

```dockerfile
FROM python:3.9-slim as builder
# ... pełne zależności, do testów

FROM python:3.9-slim
# ... zależności produkcyjne (bez dev)
```
Bez jawnego `--target` przy buildzie, Docker buduje **ostatni** stage w
pliku. W tym projekcie to działa poprawnie "przez przypadek" — stage
produkcyjny jest ostatni, więc `docker.build()` w Jenkinsfile (bez
`--target`) trafia we właściwy obraz. Kruche: gdyby ktoś dopisał trzeci
stage albo zmienił kolejność, build zacząłby cicho produkować coś innego,
bez błędu kompilacji sygnalizującego problem. Jawny `--target <nazwa>`
byłby bezpieczniejszy, ale wymaga nazwania obu stage'y — świadomie
zostawione bez zmian w tym projekcie (fork appki, minimalna ingerencja).

`docker-compose.test.yml` w tym projekcie buduje wyłącznie
`--target builder` (jawnie), właśnie żeby uniknąć tej niejednoznaczności
dla obrazu **testowego**.

---

## 3. `pull` w `community.docker.docker_compose_v2` (Ansible)

```yaml
- community.docker.docker_compose_v2:
    project_src: "{{ app_dest }}"
    pull: "{{ 'always' if app_image_tag == 'latest' else 'missing' }}"
    state: present
```
Trzy relewantne wartości: `missing` (pull tylko jeśli obrazu nie ma
lokalnie — domyślne, idempotentne), `always` (pull za każdym razem,
niezależnie czy coś się zmieniło). **`pull: always` łamie idempotencję
Ansible** — moduł zgłasza `changed` przy każdym przebiegu, nawet gdy
treść obrazu pod danym tagiem jest identyczna z poprzednim uruchomieniem
(nie ma jak porównać, skoro i tak zawsze każe Dockerowi sprawdzić/pobrać).

Rozwiązanie użyte w tym projekcie: `pull` warunkowy, zależny od tego czy
tag jest mutowalny (`latest` — zawsze świeży, `always`) czy unikalny per
commit z realnego CD (`missing` — idempotentne, bo nowy tag i tak wymusi
pobranie, a ten sam tag drugi raz nic nie robi).

---

## 4. `ansible-core` vs pełny pakiet `ansible`

```
apt-get install ansible-core   # silnik, bez dołączonych collections (~1.4MB deb)
apt-get install ansible        # silnik + setki community collections (~18.6MB deb, +281MB po rozpakowaniu)
```
Jeśli i tak instalujesz potrzebne kolekcje jawnie przez `ansible-galaxy
collection install`, `ansible-core` wystarcza i jest drastycznie lżejszy.
W tym projekcie pełny `ansible` (dociągający m.in. `infinidat.infinibox`,
kompletnie niepotrzebną kolekcję) był bezpośrednią przyczyną jednego z
trzech incydentów `no space left on device` na małym (6.8GB) EBS —
szczegóły `etap-6-podsumowanie.md`.

---

## 5. `No such container` tuż po `Created` (containerd image store)

Sporadyczny, udokumentowany w ekosystemie Dockera race: `docker compose
up --build` tworzy kontener z właśnie zbudowanego obrazu, ale containerd
nie zdążył jeszcze w pełni zarejestrować referencji do tego obrazu, zanim
compose próbuje go wystartować. Objaw: `Container ... Created` → `...
Starting` → `Error response from daemon: No such container: <hash>`.

Zaobserwowane raz w tym projekcie, po świeżym `terraform destroy`→`apply`
(nowy, "zimny" demon Dockera na świeżej instancji). Obejście: retry builda
(w Jenkinsie: push pusty commit / "Build Now"). Jeśli się powtarza —
`docker info | grep -i containerd` i ewentualne wyłączenie containerd
snapshotter w `/etc/docker/daemon.json`:
```json
{ "features": { "containerd-snapshotter": false } }
```

---

## 6. Resize EBS a system plików w środku (nie stricte Docker, ale odkryte
przy debugowaniu miejsca dla Dockera)

Powiększenie `root_block_device.volume_size` w Terraform zmienia sam
wolumen EBS w AWS, ale **nie rozciąga automatycznie** partycji/systemu
plików Linuksa wewnątrz, jeśli AWS zmodyfikował wolumen in-place (bez
wymiany instancji):
```bash
lsblk                        # sprawdź układ partycji
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
```
Przy pełnym `terraform destroy`→`apply` (nowa instancja od zera) problem
nie występuje — cloud-init na świeżym boocie formatuje partycję od razu
pod docelowy rozmiar wolumenu.

---

## 7. Historia komend/decyzji użytych w tym projekcie (skrót)

| Sytuacja | Komenda / decyzja |
|---|---|
| Diagnoza `no space left on device` | `df -h /`, `docker system df` |
| Doraźne odzyskanie miejsca (bezpieczne dla named volumes) | `docker system prune -af` (bez `--volumes`) |
| Sprawdzenie czy containerd snapshotter jest włączony | `docker info \| grep -i containerd` |
| Ciężki pakiet `ansible` zamiast `ansible-core` jako przyczyna braku miejsca | `apt-get install ansible-core` + `ansible-galaxy collection install <konkretne kolekcje>` |
| `docker_compose_v2` zawsze `changed` mimo braku realnej zmiany | `pull: always` → `pull` warunkowy zależny od tagu (sekcja 3 wyżej) |
| Weryfikacja, który stage builduje się domyślnie | sprawdzenie kolejności `FROM` w Dockerfile — ostatni wygrywa bez `--target` |

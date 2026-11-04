# Ściągawka Git

> Komendy pogrupowane tematycznie + osobna sekcja z tym, czego faktycznie
> użyliśmy w tym projekcie (i dlaczego). Kontekst: trunk-based, `main`/`master`
> jako jedyny długożyjący branch, `feature/<opis>`/`fix/<opis>` jako robocze
> (patrz `etap-0-podsumowanie.md`).

---

## 1. Podstawy — status, historia, staging

```
git status
```
Co jest zmodyfikowane / zastage'owane / untracked. Odpalaj przed każdą
komendą, która coś zmienia (checkout, reset, clean) — wymóg z zasad
bezpieczeństwa tego projektu.

```
git status -sb
```
Skrócona wersja: `-s` daje jedną linię na plik z dwuznakowym kodem
(`??` untracked, ` M` zmodyfikowany niezastage'owany, `M ` zastage'owany,
`A ` nowy zastage'owany), `-b` dokleja na górze branch + stan względem
remote. Wygodniejsze niż pełny `git status`, gdy sprawdzasz stan często
w trakcie pracy, np. przed każdym pushem.

```
git log --oneline --graph --decorate
```
Zwarty widok historii z gałęziami i tagami. Dobre pierwsze spojrzenie
przed jakąkolwiek operacją na branchach.

```
git log -1 --format='%B' <commit>
```
Pełna treść (subject + body) konkretnego commita — przydatne do
sprawdzenia, czy merge commit ma poprawny format (patrz sekcja 5).

```
git add <plik>
git restore --staged <plik>
```
Dodanie do stage'a / cofnięcie stage'a (bez ruszania plików na dysku).
Działa tylko przed commitem.

```
git rm --cached <plik>
```
Przestań śledzić plik (np. bo trafia do `.gitignore`), zostaw go na dysku.
Wymaga commita, żeby zmiana weszła do historii.

---

## 2. Branche

```
git branch -a
```
Lista branchy lokalnych + zdalnych (`remotes/origin/...`).

```
git checkout -b feature/<opis>
git checkout -b fix/<opis>
```
Nowy branch roboczy zgodnie z konwencją z Etapu 0. Nie commituj drobnych
poprawek bezpośrednio na `main` — nawet małe fixy (np. literówka) idą
przez `fix/<opis>`.

```
git fetch origin
git checkout <nazwa-brancha>
```
Pobranie brancha, którego jeszcze nie masz lokalnie (np. kolega/inna
maszyna go wypchnął). `git fetch` najpierw pobiera info o nim — lokalnie
git o nim nie wie, dopóki nie zapyta remote. Sam `checkout` wygląda jak
zwykłe przełączenie brancha, ale to "DWIM" (do-what-I-mean): jeśli nie ma
lokalnego brancha o tej nazwie, a istnieje dokładnie jeden
`origin/<nazwa-brancha>`, git sam tworzy lokalny branch śledzący zdalny
i na niego przełącza — nie trzeba `-b`.

Jawna wersja (gdy nazwa lokalna ma się różnić, albo wolisz widzieć co
się dzieje):
```
git checkout -b <nazwa-lokalna> origin/<nazwa-brancha>
```
Nowsza składnia (Git ≥ 2.23), robi to samo co DWIM-owy `checkout`:
```
git switch <nazwa-brancha>
```

```
git checkout main
```
Powrót na główny branch.

```
git branch -d <branch>
```
Usuń branch lokalnie — tylko jeśli jest już zmergowany (małe `-d`
odmawia usunięcia niezmergowanego, duże `-D` usunie i tak — nie używać
`-D` bez pewności).

```
git push origin --delete <branch>
```
Usuń zmergowany branch na remote (GitHub). Sprzątanie po merge'u —
zmergowany branch nie daje nic więcej, tylko zaśmieca listę.

```
git ls-remote --heads origin
```
Co faktycznie wisi na remote, bez pobierania (`fetch`). Przydatne, żeby
sprawdzić czy branch na GitHubie już zniknął po `push --delete`.

```
git branch -vv
```
Wszystkie lokalne branche naraz, każdy z tym co śledzi i różnicą
commitów: `main 1a2b3c [origin/main: ahead 2, behind 1] ...`. `ahead` =
masz commity, których nie ma na originie (do `push`); `behind` = origin
ma coś, czego nie masz (do `pull`); oba naraz = rozjazd wymagający
mergea/rebase. Wygodniejsze niż `status -sb` gdy chcesz sprawdzić **kilka**
branchy jednym rzutem oka, nie tylko bieżący.

```
git push origin main
```
**Kolejność argumentów: `git push <remote> <branch>`, nie odwrotnie.**
`git push main origin` (branch i remote zamienione miejscami) kończy się
`fatal: 'main' does not appear to be a git repository` — git próbuje
potraktować `main` jako nazwę zdalnego repozytorium.

```
git branch -m <stara-nazwa> <nowa-nazwa>
git push origin -u <nowa-nazwa>
git push origin --delete <stara-nazwa>
```
Rename brancha, który **już jest wypchnięty** na remote. Samo `git branch
-m` zmienia nazwę tylko lokalnie — bez kolejnych dwóch komend `origin`
dalej ma starą nazwę, i lokalny branch przestaje ją poprawnie śledzić.
Kolejność ważna: najpierw wypchnij nową nazwę (`-u` ustawia tracking),
dopiero potem skasuj starą — inaczej na chwilę branch istnieje tylko
lokalnie, bez żadnej kopii na remote.

---

## 3. Merge

```
git merge --no-ff <branch>
```
Merge z jawnym commitem scalającym (nawet jeśli fast-forward byłby
możliwy) — zachowuje w historii informację "to był osobny branch roboczy".
Bez `--no-ff` git czasem po cichu robi fast-forward i merge commit
w ogóle nie powstaje.

```
git merge --no-ff <branch> -m "Merge branch 'x'" -m "Opis co i dlaczego."
```
Dwa `-m` w merge = subject + **pusta linia** + body. Ważne: jeśli wpiszesz
całą treść w jednym `-m` z ręcznym enterem, ale bez pustej linii między
zdaniami, git i tak potraktuje wszystko jako jeden "subject" — narzędzia
pokazujące tylko pierwszą linię (`git log --oneline`, lista commitów na
GitHubie) wyświetlą wtedy sklejony, nieczytelny tekst. Zdarzyło się nam
to przy merge'u `fix/repo-hygiene` — poprawione właśnie przez dwa `-m`.

---

## 4. Cofanie zmian (przed pushem)

```
git commit --amend -m "nowa treść"
```
Podmienia treść **ostatniego** commita (jeśli jeszcze nie wypchnięty na
remote). Zmienia hash commita — nigdy nie rób tego na commicie, który
ktoś już pobrał / który jest na współdzielonym branchu bez ostrzeżenia.

```
git reset --soft <commit>
```
Cofa branch do wskazanego commita, ale **nie rusza** plików na dysku ani
indeksu (staging area) — wszystko, co było w cofniętych commitach, wraca
jako zastage'owane ("Changes to be committed"). Używamy tego, żeby
rozpakować jeden zbyt duży/zbiorczy commit z powrotem do stage'a: potem
`git restore --staged .` odstagowuje wszystko, a `git add <plik>` +
`git commit` osobno dla każdej logicznej części rozbija go na kilka
mniejszych, opisowych commitów. Bezpieczne, jeśli cofany commit nie jest
wypchnięty na remote.

```
git reset --hard <commit>
```
Cofa branch do wskazanego commita, odrzucając **też** zmiany w plikach
i w indeksie — w przeciwieństwie do `--soft` (patrz wyżej) nic z tamtych
commitów nie zostaje nawet jako niezastage'owana zmiana.
**Destrukcyjne** — ale bezpieczne, jeśli cofane commity nie są
wypchnięte na remote (nic nie ginie z historii innych, a lokalnie i tak
zostają w `git reflog` na wypadek pomyłki). Użyliśmy tego, żeby cofnąć
źle sformatowany merge commit przed jego poprawieniem.

```
git reflog
```
Historia przesunięć HEAD — siatka bezpieczeństwa po `reset --hard`,
`commit --amend` itp. Jeśli coś poszło nie tak, commit prawie zawsze
da się tu odzyskać.

---

## 5. Zdalne repozytorium

```
git push
git push origin <branch>
```
Wypchnięcie commitów. Rób to świadomie, na końcu — po sprawdzeniu
`git log` i `git status`, nie automatycznie po każdym commicie.

```
git fetch
```
Pobiera stan z remote bez mergowania (bezpieczne, nic nie zmienia
lokalnie) — dobre do sprawdzenia "czy coś się zmieniło na GitHubie"
przed `pull`/`merge`.

```
git fetch origin
git status -sb
```
Porównanie lokalnego brancha z origin. `git status` sam z siebie
porównuje tylko względem **ostatniego znanego** stanu `origin`
zapisanego lokalnie (`refs/remotes/origin/...`) — to się nie odświeża
samo, stąd najpierw `fetch`. Dopiero potem `status -sb` pokaże realny
`ahead`/`behind`, np. `## feature/ansible...origin/feature/ansible
[ahead 2]` = masz 2 commity, których nie ma jeszcze na origin.

```
git log --oneline origin/<branch>..<branch>
```
Konkretnie **które** commity to są (nie tylko liczba) — commity, które
masz lokalnie, a nie ma ich na origin ("co bym wypchnął"). Odwrotnie
(`origin/<branch>..<branch>` zamienione miejscami) pokazuje co jest na
remote, a czego nie masz lokalnie ("co bym ściągnął").

```
git log --oneline <A>..<B>
```
Ogólny wzorzec — `<A>` i `<B>` to **dowolne** referencje, nie tylko
`origin/<branch>` i `<branch>`: nazwa brancha, hash commita, `HEAD`, cokolwiek.
Pokazuje commity osiągalne z `<B>`, których nie ma w `<A>` — czyli "co ma
`<B>`, czego nie ma `<A>`". **Kolejność ma znaczenie** — zamiana miejscami
odwraca pytanie. Dwa konkretne podstawienia użyte w tym projekcie:

| Chcę sprawdzić | Komenda |
|---|---|
| Co bym wypchnął (lokalne commity, których nie ma na remote) | `origin/<branch>..<branch>` |
| Czy branch ma coś, czego `main` jeszcze nie wchłonął (przed/po merge'u powinno wyjść puste) | `main..<branch>` |

Pułapka: `origin/<branch>..main` (branch feature'owy zbudowany liniowo na
`main`) **zawsze** wyjdzie puste, niezależnie czy merge się już wydarzył —
bo `main` z definicji nie ma niczego, czego nie miałby też branch zbudowany
na jego szczycie. To nie jest test na to, czy merge się odbył — do tego
służy `main..<branch>` (kierunek odwrotny), patrz tabela wyżej.

---

## 6. Diagnostyka pojedynczego pliku (working tree vs commit)

```
git restore <plik>
```
Cofa **niezastage'owane** zmiany w pliku z powrotem do stanu z ostatniego
commita (HEAD) — w odróżnieniu od `git restore --staged <plik>` (sekcja 1),
który tylko odstage'owuje, nic nie zmieniając na dysku. Użyj, gdy lokalna
edycja przypadkiem cofnęła/zepsuła coś, co było już poprawnie
zacommitowane. **Destrukcyjne dla niezastage'owanych zmian w tym pliku**
— jeśli edycja miała coś wartościowego, przepada.

```
git diff HEAD -- <plik>
```
Różnica między tym, co jest teraz na dysku, a ostatnim commitem —
dla jednego, konkretnego pliku. Pierwsze narzędzie, gdy podejrzewasz,
że plik "wygląda inaczej niż powinien po fixie" — pokazuje dokładnie
które linie się różnią, zamiast zgadywać.

```
git show HEAD:<plik>
```
Treść pliku dokładnie taka, jak w ostatnim commicie — bez ruszania
working tree. Przydatne do porównania "co jest w gicie" z tym, co
ktoś wkleja z innej maszyny (np. z VM), gdy nie wiadomo które źródło
jest aktualne.

```
git ls-files --error-unmatch <plik>
```
Sprawdza, czy plik jest w ogóle śledzony przez gita (zwraca błąd, jeśli
nie). Ważne przed usunięciem pliku "poprawnie" — jeśli nieśledzony,
wystarczy zwykłe `rm`; jeśli śledzony, usunięcie/nadpisanie ma
konsekwencje w historii.

---

## 7. Rename z zachowaniem historii

```
git mv stary_plik.md nowy_plik.md
```
Rename + `git add` w jednym kroku. Git i tak wykrywa rename po
podobieństwie treści nawet przy `rm`+`add` osobno, ale `git mv` jest
jawne i czytelniejsze w `git status`. Użyte przy poprawce literówki
`CONTRIGUTING.md` → `CONTRIBUTING.md`.

---

## 8. Historia komend użytych w tym projekcie (skrót)

| Sytuacja | Komenda |
|---|---|
| Rename pliku z literówką | `git mv CONTRIGUTING.md CONTRIBUTING.md` |
| Sprawdzenie brancha z fixem AMI | `git log --oneline -20`, `git branch -a` |
| Poprawa treści ostatniego (niewypchniętego) commita | `git commit --amend -m "..."` |
| Cofnięcie przypadkowej lokalnej edycji, która cofnęła już zacommitowany fix | `git diff HEAD -- <plik>` (diagnoza), `git restore <plik>` |
| Sprawdzenie skąd VM ma stary kod mimo naprawy na Windowsie | `git status -sb` (ahead/behind origin), `git push` brakujący |
| Cofnięcie źle sformatowanego merge commita (niewypchniętego) | `git reset --hard 9dfb6bb` |
| Ponowny merge z poprawnym formatem wiadomości | `git merge --no-ff fix/repo-hygiene -m "..." -m "..."` |
| Sprzątanie zmergowanego brancha lokalnie | `git branch -d fix/repo-hygiene` |
| Sprzątanie zmergowanego brancha na GitHubie | `git push origin --delete fix/ami-filter-ubuntu-noble` |
| Weryfikacja co wisi na remote | `git ls-remote --heads origin` |
| Testowy push bez ruszania kodu appki (żeby odpalić webhook/pipeline) | `git commit --allow-empty -m "test: ..."` — pusty commit, brak zmian w plikach, zgodne z zasadą minimalnej ingerencji w fork appki |
| `main` lokalnie 8 commitów przed `origin/main` (zmergowane lokalnie, nigdy niewypchnięte) | `git branch -vv` do zdiagnozowania, `git push origin main` (fast-forward, bez konfliktu) do naprawy |
| Sprawdzenie czy `git checkout <inny-branch>` skasował niescommitowaną robotę (untracked pliki) | `git status` od razu po checkout — untracked pliki przetrwały, bo nie kolidowały z niczym trackowanym na docelowym branchu |
| Branch nazwany po pierwszym sub-tasku etapu, mimo że urósł o więcej | rename z zachowaniem historii na remote (sekcja 2 wyżej) — jedna nazwa = cały etap, nie pierwszy fragment pracy |

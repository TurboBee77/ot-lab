# Notatki do README repo `fastapi-devops-infra`

> Przygotowane pod przyszły wątek, który będzie pisał/uzupełniał README tego
> repo (Etap 8, waga 2, obowiązkowe). Ten wątek nie będzie miał pamięci
> rozmowy, w której powstały Etapy 1–2 — poniżej skondensowane fakty, żeby
> nie trzeba było ich wymyślać/wyciągać z historii commitów od nowa.
>
> **Nie mylić z `Wymagania Projektu Dyplomowego.md`** — to oryginalna treść
> zadania od promotora, nie notatki robocze. Ten plik to praktyczne fakty
> o tym, co repo faktycznie zawiera i jak to odtworzyć od zera.
>
> Warunek "gotowe" Etapu 8 (`DevOpsProj.md`): *"osoba z zewnątrz, czytając
> tylko README, potrafi powtórzyć `terraform apply` → `ansible-playbook` →
> działający pipeline"*. README musi więc być samowystarczalne — nie
> odwoływać się do tego pliku ani do `etap-*-podsumowanie.md` jako
> wymaganej lektury, tylko streścić z nich to, co faktycznie potrzebne.

---

## 1. Wymagania wstępne (setup przed pierwszym uruchomieniem)

Z Etapu 1 (`etap-1-podsumowanie.md`):
- **Terraform** (`>= 1.5`)
- **AWS CLI**, skonfigurowane (`aws configure` lub zmienne środowiskowe)
  z uprawnieniami do EC2/VPC w koncie docelowym
- **Istniejąca para kluczy EC2** o nazwie zgodnej z `key_name` w
  `terraform.tfvars` (Terraform jej nie tworzy — trzeba założyć wcześniej
  ręcznie, `aws ec2 create-key-pair` albo `aws ec2 import-key-pair`)

Doszło w Etapie 2 (`etap-2-podsumowanie.md`):
- **Ansible** (`ansible-core`, plus **`community.general`** collection —
  moduł `ufw` nie jest częścią `ansible-core`: `ansible-galaxy collection
  install community.general`)
- **Python 3 + PyYAML** na control node (do `scripts/generate_inventory.py`
  — PyYAML jest i tak zależnością samego Ansible, więc zwykle nic dodatkowo
  nie trzeba instalować)
- Klucz SSH (ten sam co dla Terraform/EC2) dodany do **`ssh-agent`**
  (`ssh-add`) — Ansible łączy się bez jawnej ścieżki do klucza w configu

**Twarde wymaganie środowiskowe (nie do pominięcia w README):** `terraform`
i `ansible-playbook` muszą być odpalane z **Linuksa** (control node) — nie
z Windows. Ansible nie działa jako control node natywnie na Windows.

## 2. Pełna sekwencja komend od zera

```bash
# 1. Infrastruktura (control node, katalog terraform/)
cd terraform
terraform init
terraform apply

# 2. Wygenerowanie inventory Ansible z outputu Terraform (katalog ansible/)
cd ../ansible
python3 scripts/generate_inventory.py

# 3. Konfiguracja bazowa hostów (docker + ufw)
ansible-playbook --syntax-check site.yml   # opcjonalny, szybki filtr
ansible-playbook site.yml
```

Po Etapie 3+ dojdzie tu jeszcze rola `deploy` (aplikacja na `app`, Jenkins
na `ci`, monitoring na `monitoring`) — na razie `site.yml` obejmuje tylko
`docker`+`ufw`.

**Sprzątanie po sesji pracy (Free Tier — przypomnieć w README wprost):**
```bash
cd terraform
terraform destroy
```

## 3. Gdzie szukać szczegółów (dla README, nie do wklejania 1:1)

README ma być **zwięzłe i samowystarczalne** — poniższe pliki mają dużo
więcej kontekstu/uzasadnień niż potrzeba w README, ale są źródłem, z
którego README powinno streszczać fakty, zamiast wymyślać je na nowo:

| Plik | Co tam jest |
|---|---|
| `etap-1-podsumowanie.md` | Terraform: moduł EC2, decyzje o AMI/kluczu SSH/Elastic IP/security groups |
| `etap-2-podsumowanie.md` | Ansible: struktura ról, decyzje (inventory generowane skryptem, ssh-agent, kolejność tasków ufw itd.) |
| `docs/network-architecture.md` (w repo infra) | Diagram sieci, porty otwarte per rola, plan rozszerzeń |
| `ansible-sciagawka.md` | Wyjaśnienia koncepcji Ansible — **nie** dla README (to materiał edukacyjny dla usera), ale dobre źródło do zrozumienia "dlaczego" przy pisaniu opisów |

## 4. Znane luki do domknięcia (przed albo przy pisaniu README)

- **Brak `.gitattributes`** w repo, mimo że `CLAUDE.md` zakłada jego
  istnienie (wymuszenie LF dla `*.sh`/`*.yml`/`Jenkinsfile`). Obecnie
  działa przez przypadek (ustawienia edytora), nie przez wymuszenie.
- **`terraform/.terraform.lock.hcl` nieotrackowany w git** — `.gitignore`
  celowo go nie wyklucza (ma być commitowany dla spójnych wersji
  providera), ale nigdy nie trafił do `git add`. Trzeba to zrobić zanim
  README zacznie opisywać "czysty" stan repo.
- Warning Ansible "group i host o tej samej nazwie" w wygenerowanym
  inventory — sprawdzić czy fix (sufiks nazwy hosta) już wszedł, zanim
  README opisze strukturę inventory jako przykład.

## 5. Struktura repo (stan na koniec Etapu 2)

```
fastapi-devops-infra/
├── terraform/
│   ├── main.tf, variables.tf, outputs.tf
│   ├── modules/ec2-instance/   (reużywalny, wywołany 3×: ci/app/monitoring)
│   └── terraform.tfvars.example
├── ansible/
│   ├── ansible.cfg
│   ├── group_vars/all.yml
│   ├── inventory/hosts.yml     (generowany, gitignored)
│   ├── scripts/generate_inventory.py
│   ├── roles/{docker,ufw}/
│   └── site.yml
├── docs/network-architecture.md
└── .gitignore, CLAUDE.md, CONTRIBUTING.md
```

Docker-compose dla `app`/`monitoring` i rola `deploy` jeszcze nie istnieją
(Etap 3+) — nie opisywać ich w README jako gotowe, tylko ewentualnie jako
"w toku"/backlog, zgodnie z tym co faktycznie jest w repo w danym momencie.

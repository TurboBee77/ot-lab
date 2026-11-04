# Ściągawka Terraform

> Komendy i pojęcia pogrupowane tematycznie + osobna sekcja z tym, czego
> faktycznie użyliśmy w tym projekcie (i dlaczego). Kontekst: `terraform`
> odpalany zawsze z VM Linux (control node), nigdy z Windows — patrz
> `CLAUDE.md`. Jeden reużywalny moduł `ec2-instance`, wywoływany 3× z
> root module (`fastapi-devops-infra/terraform/`).

---

## 1. Podstawowy cykl

```
terraform init
terraform plan
terraform apply
terraform destroy
```

`init` — pobiera providery (`hashicorp/aws`) i konfiguruje backend (u nas
lokalny plik `.tfstate`, S3 to świadomie odłożony backlog). Trzeba
uruchomić po `git clone`/dodaniu nowego providera, nie przy każdej zmianie.

`plan` — **tylko odczyt** realnej infrastruktury (`Refreshing state...`)
+ wyliczenie różnicy między stanem (`.tfstate`) a kodem. Nic nie zmienia.
Zawsze przed `apply`, żeby wiedzieć dokładnie co się wydarzy — twarda
zasada bezpieczeństwa tego projektu.

**Czytanie planu:** `~` = update in-place, `+`/`-` wewnątrz `~` bloku =
konkretne dodane/usunięte elementy zagnieżdżonej listy (np. nowa reguła
`ingress` w security group), `# (N unchanged attributes hidden)` = reszta
zasobu bez zmian, nieistotna dla tej zmiany. `Plan: X to add, Y to
change, Z to destroy` na końcu — licznik **zasobów**, nie pojedynczych
atrybutów w środku (jeden zasób z 3 nowymi regułami `ingress` to wciąż
"1 to change").

---

## 2. `resource` vs `data`

```hcl
resource "aws_security_group" "this" { ... }   # Terraform TWORZY i zarządza tym zasobem

data "aws_vpc" "default" {                     # Terraform tylko CZYTA coś, co już istnieje
  default = true
}
```

`data` nie tworzy, nie zmienia, nie usuwa niczego w AWS — to zapytanie
"co tam już jest". Użyte w tym projekcie: instancje EC2 od zawsze lądują
w domyślnym VPC konta (bez jawnego `vpc_id`), więc nie było potrzeby
znać jego CIDR-u. Gdy zaszła potrzeba ograniczenia reguły `ingress` do
"tylko wewnątrz VPC" (nie z internetu), trzeba było ten CIDR poznać —
`data "aws_vpc" "default" { default = true }` czyta go bez tworzenia
nowego VPC ani hardkodowania CIDR-u na sztywno w kodzie.

Odwołanie: `data.<typ>.<nazwa>.<atrybut>`, np.
`data.aws_vpc.default.cidr_block` — ta sama składnia co przy `resource`,
tylko z `data.` na początku.

---

## 3. `dynamic` blocks — powtarzalne zagnieżdżone bloki z listy

```hcl
variable "extra_ingress_ports" {
  type    = list(number)
  default = []
}

dynamic "ingress" {
  for_each = var.extra_ingress_ports
  content {
    description = "extra port ${ingress.value}"
    from_port   = ingress.value
    to_port     = ingress.value
    protocol    = "tcp"
    cidr_blocks = [var.ssh_allowed_cidr]
  }
}
```

Problem, który to rozwiązuje: blok `ingress { }` wewnątrz
`aws_security_group` normalnie trzeba pisać ręcznie, raz na port. `dynamic
"ingress"` generuje **jeden taki blok per element listy** — zmiana listy
w `terraform.tfvars` (np. `[8000]` → `[8000, 9100]`) automatycznie daje
drugą regułę, bez dotykania kodu `.tf`.

`ingress.value` wewnątrz `content { }` — bieżący element iteracji
(nazwa `ingress` tu to nazwa bloku z zewnętrznego `dynamic "ingress"`,
nie coś magicznego — dla `dynamic "foo"` byłoby `foo.value`).

**Można mieć więcej niż jeden `dynamic` blok tego samego typu w tym
samym zasobie** — w tym projekcie `aws_security_group.this` ma **dwa**
`dynamic "ingress"` obok siebie: jeden dla `extra_ingress_ports`
(publiczne, `ssh_allowed_cidr`), drugi dla `internal_ingress_ports`
(tylko wewnątrz VPC, `internal_cidr`) — różnią się tylko źródłem listy
i `cidr_blocks`, oba dokładają reguły do tej samej, jednej listy
`ingress` na zasobie.

---

## 4. Moduł reużywalny — wywołanie z różnymi parametrami

```hcl
module "ci" {
  source = "./modules/ec2-instance"
  role   = "ci"
  # ...
}

module "app" {
  source = "./modules/ec2-instance"
  role   = "app"
  # ...
}
```

Ten sam kod modułu (`modules/ec2-instance/`), trzy niezależne wywołania
z różnymi wartościami zmiennych — nie trzy kopie tego samego `.tf`.
Osobno punktowane w `Punktacja.txt` jako "Reużywalność". Zmienna
niewymagana w danym wywołaniu (bez `default` w `variables.tf` modułu)
**musi** być podana — pominięcie kończy się błędem `terraform plan` w
stylu "no value for required variable", nie cichym pominięciem.

**Pułapka złapana na żywo w tym projekcie:** dodanie nowej zmiennej do
modułu (`internal_ingress_ports`, `internal_cidr`) i do root
`variables.tf` **nie wystarczy** — trzeba jeszcze jawnie przekazać ją w
każdym z 3 wywołań `module "x" { ... }` w root `main.tf`. Zapomniana
linijka w jednym wywołaniu (`module "monitoring"` bez
`extra_ingress_ports = var.monitoring_extra_ingress_ports`) nie daje
żadnego błędu — zmienna po prostu nigdzie się nie używa, `terraform
plan` pokazuje `0 to add` tam, gdzie się jej spodziewasz, bez żadnej
wskazówki dlaczego.

---

## 5. `terraform.tfvars` vs `terraform.tfvars.example`

`terraform.tfvars` — **prawdziwe** wartości (region, nazwa klucza SSH,
listy portów). W `.gitignore`, nigdy niecommitowany, żyje **tylko** na
VM Linux (control node) — nigdy nie trafia na Windows, zgodnie z
podziałem środowisk tego projektu. `terraform plan`/`apply` czytają go
automatycznie (Terraform sam podłącza plik o tej dokładnej nazwie w
katalogu roboczym, bez `-var-file` flagi).

`terraform.tfvars.example` — commitowany szablon z przykładowymi
wartościami i komentarzami, punkt startowy do skopiowania (`cp
terraform.tfvars.example terraform.tfvars`) na nowej maszynie/po
sklonowaniu repo od zera.

**Pułapka złapana na żywo:** edycja `.example` zamiast prawdziwego
pliku — zmiana wygląda na zrobioną (widoczna w edytorze, w gicie), ale
`terraform plan` jej nie widzi, bo czyta wyłącznie plik bez `.example`.

---

## 6. Resize EBS (`root_block_device.volume_size`) — pułapka z systemem plików

```hcl
root_block_device {
  volume_size = var.root_volume_size
  volume_type = "gp3"
}
```

Zmiana tej wartości i `terraform apply` **powiększa wolumen EBS**, ale
**nie rozciąga** automatycznie partycji ani systemu plików w środku przy
modyfikacji in-place na już istniejącej instancji — trzeba by ręcznie
`growpart` + `resize2fs` na VM. Przy pełnym `terraform destroy` →
`apply` problem nie występuje (świeży boot formatuje od razu pod
docelowy rozmiar) — to jedna z przyczyn, dla której ten projekt trzyma
się dyscypliny pełnego recreate zamiast łatania infrastruktury w miejscu
między etapami.

---

## 7. Historia komend/decyzji użytych w tym projekcie (skrót)

| Sytuacja | Komenda / decyzja |
|---|---|
| AMI Ubuntu nie znajdowane przez `data "aws_ami"` | filtr `hvm-ssd` nie pasował do aktualnych AMI Noble 24.04 — poprawiony wzorzec `name` (Etap 1) |
| Ograniczenie portu do ruchu tylko wewnątrz VPC (node_exporter, Prometheus UI) | `data "aws_vpc" "default" { default = true }` + drugi `dynamic "ingress"` z `cidr_blocks = [data.aws_vpc.default.cidr_block]` zamiast `ssh_allowed_cidr` (Etap 7) |
| Zapomniana zmienna nieprzekazana do jednego z 3 wywołań modułu | `terraform plan` pokazujący za mało zmian niż oczekiwane — diagnoza przez ręczne czytanie `main.tf` linia po linii, nie przez komunikat błędu (bo błędu nie ma) |
| Root volume `ci` za mały pod Docker + Jenkins | `ci_root_volume_size` sparametryzowane w `terraform.tfvars` (Etap 6) |

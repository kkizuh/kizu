## 0) Что это вообще

Terraform (TF) — декларативно описываешь инфраструктуру в `.tf` файлах → `plan` показывает что изменится → `apply` делает то же самое в облаке. State-файл хранит текущее «состояние» ресурсов.

---

## 1) Минимальный скелет проекта

```
infra/
├─ main.tf          # что создаём (ресурсы, модули)
├─ variables.tf     # входные переменные (типы, валидация)
├─ outputs.tf       # что показывать после apply (ипшник и т.п.)
├─ terraform.tfvars # значения по умолчанию (локально)
├─ versions.tf      # версии TF/провайдеров
└─ .gitignore       # .terraform/, *.tfstate*, terraform.tfvars
```

**`versions.tf` (пин версий):**

```hcl
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
```

**`providers` + backend (S3 пример) в `main.tf`:**

```hcl
terraform {
  backend "s3" {
    bucket = "my-tf-state-bucket"
    key    = "envs/dev/terraform.tfstate"
    region = "eu-central-1"
    dynamodb_table = "tf-locks"   # для блокировок state
    encrypt = true
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = { Project = "demo", Owner = "kizu" }
  }
}
```

**`variables.tf` (политика безопасных дефолтов):**

```hcl
variable "aws_region" {
  type        = string
  description = "AWS region"
  default     = "eu-central-1"
  validation {
    condition     = contains(["eu-central-1","eu-west-1"], var.aws_region)
    error_message = "Allowed regions: eu-central-1, eu-west-1."
  }
}

variable "env" {
  type        = string
  description = "Environment name"
  default     = "dev"
  validation {
    condition     = can(regex("^(dev|stg|prod)$", var.env))
    error_message = "env must be dev|stg|prod."
  }
}

variable "db_password" {
  type        = string
  description = "DB password"
  sensitive   = true         # скрывать в планах/выводе
}
```

**`terraform.tfvars` (удобно, но не коммить секреты):**

```hcl
env         = "dev"
aws_region  = "eu-central-1"
# db_password = "ХРАНИ ВНЕ ГИТА!"  # лучше через ENV: TF_VAR_db_password
```

**`outputs.tf`:**

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

**Простейший ресурс (пример):**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.10.0.0/16"
}
```

---

## 2) Команды на память

```bash
terraform init          # инициализация провайдера/бэкенда
terraform fmt -recursive# автоформат
terraform validate      # базовая проверка синтаксиса
terraform plan          # покажи, что изменится
terraform apply         # сделай (подтверждение Y)
terraform destroy       # снести всё (осторожно)
```

State/окружения:

```bash
terraform workspace new dev
terraform workspace select dev
terraform state list
terraform state show aws_vpc.main
```

Переменные:

```bash
export TF_VAR_db_password='super-secret'    # безопаснее, чем tfvars в гите
terraform apply -var="env=stg" -var-file=stg.tfvars
```

---

## 3) Remote state + backend (зачем)

- **Зачем:** один state для команды, блокировки, бэкапы.
- **Как:** см. блок `backend "s3"` выше + создайте заранее:
    - S3 bucket для state,
    - DynamoDB таблицу `tf-locks` с ключом `LockID` (для lock).

> В других облаках аналог: GCS backend, AzureRM backend и т.д.

---

## 4) Шаблон модуля (повторное использование)

**Структура:**

```
modules/vpc/
  main.tf        # ресурсы
  variables.tf   # входы
  outputs.tf     # выходы
```

**`modules/vpc/variables.tf`:**

```hcl
variable "cidr" {
  type        = string
  description = "VPC CIDR"
}

variable "tags" {
  type        = map(string)
  default     = {}
}
```

**`modules/vpc/main.tf`:**

```hcl
resource "aws_vpc" "this" {
  cidr_block = var.cidr
  tags       = merge({ Name = "vpc" }, var.tags)
}
```

**`modules/vpc/outputs.tf`:**

```hcl
output "id" { value = aws_vpc.this.id }
```

**Подключение модуля в корне проекта (`main.tf`):**

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.20.0.0/16"
  tags   = { Env = var.env }
}
```

---

## 5) Качество кода: `tflint` и `tfsec`

Установка (пример для Linux/macOS, см. их доки под вашу ОС):

```bash
# tflint
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
tflint --init
tflint             # найдёт ошибки/несоответствия best practices

# tfsec
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install.sh | bash
tfsec .            # статический анализ безопасности (открытые SG, etc.)
```

Мини-пайплайн локально:

```bash
terraform fmt -recursive
terraform validate
tflint
tfsec .
terraform plan
```

---

## 6) Политика переменных (безопасные дефолты)

- **Типизируй всё:** `string`, `number`, `bool`, `list(string)`, `map(string)` — меньше сюрпризов.
- **Валидация:** через `validation { condition = ... }`.
- **`sensitive = true`** для секретов (скрывает в output/plan).
- **Не храни секреты в гите:** используй ENV `TF_VAR_*` или внешние секрет-менеджеры (AWS SSM/Secrets Manager, Vault).
- **Значения по умолчанию** делай консервативными (минимальные размеры, закрытые SG, выключенные публичные IP), а «расшивать» явно через tfvars.

---

## 7) Логи/артефакты TF

- **`.terraform/`** — кеш провайдеров (в гит не коммитим).
- **`.terraform.lock.hcl`** — lock-файл версий провайдеров (как `pip freeze`) — **коммить**, чтобы сборки были детерминированными.
- **`*.tfstate`** — локальный state (в гит **нельзя**). При backend — хранится удалённо.
- **`crash.log`** — появляется при крэше TF (редкость, но полезно в баг-репортах).

`.gitignore` (минимум):

```
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
terraform.rc
override.tf
```

---

## 8) Частые грабли (и быстрые ответы)

- **«AccessDenied»** — проверь AWS креды: `aws sts get-caller-identity`, профиль/роль, env vars.
- **«Error locking state»** — висит lock в DynamoDB → сними вручную через консоль (убедись, что никто не применяет план).
- **Drift (ресурс меняли руками)** — `terraform plan` покажет рассинхрон; решай: принять как есть (import) или вернуть «как в коде».
- **Разделяй окружения** — через workspaces (`dev/stg/prod`) или отдельные каталоги/репы (ещё лучше).

---

## 9) TL;DR — командами

```bash
# Инициализация
terraform init

# Приведи стиль, проверь
terraform fmt -recursive
terraform validate
tflint
tfsec .

# План и применить
terraform plan -out tf.plan
terraform apply tf.plan

# Рабочие пространства (окружения)
terraform workspace new dev
terraform workspace select dev
```

Ты это осилишь. TF — про «код = истина», а не «тыкал мышкой и забыл». Начни с одного VPC и одной подсети — дальше пойдёт как по маслу.
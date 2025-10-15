# Шаблоны скриптов: аргументы, логи, `set -euo pipefail` (c комментами)

> задача: быстро писать безопасные bash-скрипты, понимать _почему_ так, и не ловить мин.

---

## 0) Мини-скелет (безопасный дефолт)

```bash
#!/usr/bin/env bash
# shellcheck disable=SC2155,SC2086  # подсказка для shellcheck: знаем, что делаем
set -Eeuo pipefail                  # E: ловим ERR в функциях/подпроцессах, e: падать на ошибке, u: незаданная переменная = ошибка, pipefail: ошибка в любом звене пайпа = ошибка
IFS=$'\n\t'                         # безопасный разделитель (не рвёмся на пробелах)

# --- traps & cleanup ---
_tmpdir="$(mktemp -d)"              # создаём временную папку и держим её путь
cleanup(){ rm -rf -- "$_tmpdir"; }  # функция очистки времёнки
trap 'rc=$?; echo "[ERR] $0:$LINENO exit $rc" >&2; cleanup; exit $rc' ERR  # если любая команда/функция упала — лог + очистка + выход
trap 'cleanup' INT TERM EXIT        # при Ctrl+C/kill/обычном выходе — тоже чистим

# --- logging ---
log(){ printf '%s %s\n' "[$(date -Iseconds)]" "$*" >&2; }  # лог в stderr с ISO-временем
die(){ log "FATAL: $*"; exit 1; }                          # аварийный выход

# --- usage (помощь) ---
usage(){ cat <<'EOF'
Usage: script.sh [-n] --src /path --dst /path [--mode fast|safe]
  -n, --dry-run     не выполнять, только показать
      --src PATH    исходный путь (обязателен)
      --dst PATH    целевой путь (обязателен)
      --mode MODE   режим (fast|safe), по умолчанию safe
      -h, --help    помощь
EOF
}

# --- парсер аргументов (простые long-флаги) ---
dry_run=false; src=""; dst=""; mode="safe"           # значения по умолчанию
while (( "$#" )); do
  case "${1:-}" in
    -n|--dry-run) dry_run=true; shift ;;             # флаг без значения
    --src)  src="${2:-}"; shift 2 ;;                 # флаг со значением
    --dst)  dst="${2:-}"; shift 2 ;;
    --mode) mode="${2:-}"; shift 2 ;;
    -h|--help) usage; exit 0 ;;                      # показать помощь и выйти
    --) shift; break ;;                               # конец флагов → позиционные
    -*) die "неизвестный флаг: $1" ;;                # ловим опечатки
    *)  set -- "$@" "$1"; shift ;;                   # можно копить позиционные
  esac
done

# --- валидация входа ---
[[ -n "$src" && -n "$dst" ]] || { usage; die "нужны --src и --dst"; }
[[ "$mode" =~ ^(fast|safe)$ ]] || die "--mode fast|safe"

# --- main (пример полезной команды с dry-run) ---
log "start: src=$src dst=$dst mode=$mode dry=$dry_run"
rsync_flags=(-aHAX --delete --numeric-ids)           # флаги rsync: архивный режим, сохраняем атрибуты/ссылки, чистим удалённые на dst
$dry_run && rsync_flags+=(--dry-run)                 # при dry-run добавляем симуляцию
rsync "${rsync_flags[@]}" "$src/" "$dst/"            # кавычки и массив = безопасно для пробелов

log "done"
```

---

## 1) Аргументы: два способа и когда какой

### A) Ручной парсер (как в скелете)

- Читаемо, легко расширять, поддержка `--flag value`.  
    − Нет автоматических коротких флагов.
    

### B) `getopts` для коротких флагов

```bash
from=""; out=""; dry=false
while getopts ":f:o:nh" opt; do        # двоеточие после буквы = ожидается значение
  case "$opt" in
    f) from="$OPTARG" ;;
    o) out="$OPTARG"  ;;
    n) dry=true       ;;
    h) usage; exit 0  ;;
    \?) die "неизвестный флаг: -$OPTARG" ;;
    :)  die "флаг -$OPTARG требует аргумент" ;;
  esac
done
shift $((OPTIND-1))                    # сдвигаем позиционные после флагов
```

> Микс? делай ручной парсер — он проще для long-опций.

---

## 2) Логи: файл, journald, уровни

```bash
LOG_FILE="${LOG_FILE:-/var/log/mytool.log}"       # можно переопределить окружением
logf(){ printf '%s %s\n' "[$(date -Iseconds)]" "$*" | tee -a "$LOG_FILE" >&2; }
info(){ logf "[INFO] $*"; }
warn(){ logf "[WARN] $*"; }
err(){  logf "[ERR ] $*"; }

# В systemd-юните лучше писать в journald:
# [Service]
# StandardOutput=journal
# StandardError=journal
# SyslogIdentifier=mytool
```

---

## 3) Таймауты, ретраи, эксклюзивный запуск

```bash
# Повтор команды с экспоненциальной задержкой (1,2,4,8,16 сек)
retry(){
  local tries=${1:-5}; shift
  local d=1
  for ((i=1;i<=tries;i++)); do
    "$@" && return 0 || { sleep "$d"; d=$((d*2)); }
  done
  return 1
}
# пример: ждём, пока API оживёт
retry 5 curl -fsS http://127.0.0.1/health || die "healthcheck fail"

# Таймаут (если команда может зависнуть)
timeout 20s some_long_cmd || die "some_long_cmd timed out"

# Лок-файл → запрет параллельных запусков
lockfile="/run/mytool.lock"
exec 9>"$lockfile"                 # открываем дескриптор 9 на файл
flock -n 9 || die "already running" # берём эксклюзивную блокировку, иначе выходим
```

---

## 4) Времёнка и безопасный `rm -rf`

```bash
_tmpdir="$(mktemp -d)"             # создаем /tmp/tmp.XXXXXX
trap 'rm -rf -- "$_tmpdir"' EXIT   # гарантированно удалим

safe_rm(){                         # удаляем только если путь реально существует
  local target="$1"
  [[ -n "$target" && -e "$target" ]] || return 0
  if ${dry_run:-false}; then
    echo "DRY: rm -rf -- $target"
  else
    rm -rf -- "$target"
  fi
}
```

---

## 5) Конфиг и секреты (`.env`)

```bash
# Переменные окружения из файла (ключ=значение), безопасно
envfile="/etc/mytool.env"          # права 600, владелец root/mytool
if [[ -f "$envfile" ]]; then
  set -a                           # экспортировать всё, что будет загружено
  # shellcheck source=/dev/null
  . "$envfile"                     # загрузить
  set +a
fi
# теперь VAR из .env доступны как обычные переменные/окружение
```

**Не клади секреты в git.** Для прод — менеджер секретов (Vault/SSM/…).

---

## 6) JSON/YAML: `jq`/`yq` на пальцах

```bash
command -v jq >/dev/null || die "jq не установлен"

name="$(jq -r '.name' cfg.json)"                    # поле name как строка
mapfile -t ports < <(jq -r '.ports[]' cfg.json)     # массив портов

# YAML:
# yq -p=auto сам определит формат; -r = raw строки без кавычек
images="$(yq -r '.spec.template.spec.containers[].image' deploy.yaml)"
```

---

## 7) Параллелизм: быстро и без боли

```bash
# xargs: 4 параллельных процесса, по одному файлу на запуск
find data -type f -print0 | xargs -0 -n1 -P4 bash -c 'process_one "$0"'

# где process_one — твоя функция (объяви её экспортируемо, если нужно для subshell):
process_one(){ gzip -9 -- "$1"; }
export -f process_one
```

> Есть GNU `parallel` — шикарно, но не везде предустановлен.

---

## 8) Цвета/прогресс (для UX)

```bash
c_red=$'\e[31m'; c_grn=$'\e[32m'; c_yel=$'\e[33m'; c_off=$'\e[0m'
ok(){   printf '%s[OK]%s %s\n'   "$c_grn" "$c_off" "$*"; }
warn(){ printf '%s[WARN]%s %s\n' "$c_yel" "$c_off" "$*"; }
fail(){ printf '%s[FAIL]%s %s\n' "$c_red" "$c_off" "$*" >&2; }

# минимальный прогресс: крутилка точками
progress(){ while :; do printf '.'; sleep 1; done; }
# usage: progress & pid=$!; some_cmd; kill "$pid" 2>/dev/null || true; echo
```

---

## 9) Идиомы “не убейся об пробелы”

```bash
# безопасно обойти файлы (без for + glob)
find . -type f -name '*.log' -print0 |
  while IFS= read -r -d '' f; do
    gzip -9 -- "$f"
  done

# проверка наличия бинарей (не падаем на set -e → используем || die)
command -v jq >/dev/null || die "нет jq"

# значения по умолчанию
port="${PORT:-8080}"

# читаем вывод в массив без потери пробелов
mapfile -t lines < <(some_cmd_producing_lines)

# когда ожидаешь, что grep может ничего не найти — подавить ошибку
grep -q 'pattern' file || true
```

---

## 10) Шаблон “подкоманды как у git”

```bash
#!/usr/bin/env bash
set -Eeuo pipefail; IFS=$'\n\t'

usage(){ echo "Usage: $0 <init|run|status> [...]"; }
init_cmd(){ echo "init..."; }
run_cmd(){  echo "run...";  }
status_cmd(){ echo "status..."; }

main(){
  case "${1:-}" in
    init) shift; init_cmd "$@";;
    run)  shift; run_cmd  "$@";;
    status) shift; status_cmd "$@";;
    -h|--help|"") usage; exit 0;;
    *) usage; exit 2;;
  esac
}
main "$@"
```

---

## 11) Скрипт под systemd (юнит с комментами)

```ini
# /etc/systemd/system/mytool.service
[Unit]
Description=MyTool job
After=network-online.target              # ждём сеть
Wants=network-online.target

[Service]
Type=oneshot                             # задача, которая завершится
User=mytool                              # НЕ root
EnvironmentFile=-/etc/mytool.env         # окружение из файла (необязательно "-")
WorkingDirectory=/opt/mytool             # где запускать
ExecStart=/usr/local/bin/mytool.sh --mode safe
StandardOutput=journal                   # логи в journald
StandardError=journal
SyslogIdentifier=mytool                  # метка в journald
# Санбокс и лимиты:
NoNewPrivileges=yes
PrivateTmp=true
ProtectSystem=full                       # /usr,/etc ro
ProtectHome=true
ReadWritePaths=/var/log/mytool /backup   # что можно писать
CPUQuota=50%
MemoryMax=500M

[Install]
WantedBy=multi-user.target
```

> После сохранения: `systemctl daemon-reload && systemctl enable --now mytool`.

---

## 12) Подводные камни `set -e` (и как жить)

- В `if …; then` сбои **не** валят скрипт: это нормально.
```bash
    if grep -q 'ready' file; then ...; fi    # ок
```
- В пайпе **без** `pipefail` падает только последняя команда → ставим `pipefail`.
- Если ожидаешь ненулевой код — подавляй явно:
```bash
    grep -q 'maybe' file || true
```
- Проверяй `shellcheck` — реально спасает:
```bash
    sudo apt/dnf install shellcheck
    shellcheck script.sh
```

---

## 13) Готовые куски (с комментариями)

**Сжатие логов c датой:**

```bash
ts="$(date +%F-%H%M)"                                  # 2025-10-12-1530
tar -C /var/log -czf "/backup/logs-$ts.tgz" syslog auth.log \
  || die "tar fail"                                    # если tar упал — умираем
```

**Своя ротация логов (простая, без logrotate):**

```bash
max=5; base="/var/log/mytool.log"                      # храним 5 файлов
for i in $(seq $((max-1)) -1 1); do                    # 4→3→2→1
  [[ -f "$base.$i" ]] && mv -f "$base.$i" "$base.$((i+1))"
done
[[ -f "$base" ]] && mv -f "$base" "$base.1"
: > "$base"                                            # truncate
```

**Проверка места (и инодов) перед работой:**

```bash
# df -P: POSIX-формат (надёжнее парсить)
mapfile -t df_out < <(df -P / | awk 'NR==2{print $5,$4}')
use_pct="${df_out[0]%\%}"                              # 95 (без %)
free_kb="${df_out[1]}"
(( use_pct < 95 && free_kb > 102400 )) || die "мало места на /"
```

**Безопасная массовая замена конфигов:**

```bash
# сначала показать, ЧТО заменим (без -i)
sed -n -E 's/(listen\s+)[0-9]+;/\18080;/p' /etc/nginx/nginx.conf | head

# если ок — с бэкапом .bak
sed -i.bak -E 's/(listen\s+)[0-9]+;/\18080;/' /etc/nginx/nginx.conf
nginx -t || die "nginx config bad"                     # проверка синтаксиса
systemctl reload nginx
```

**Список крупных директорий (диагностика места):**

```bash
du -h --max-depth=2 /var 2>/dev/null | sort -h | tail -n 20   # top-20 крупных
```

**Кто слушает порт и каким PID (удобный вывод):**

```bash
ss -ltnp | awk 'NR==1{print;next} {printf "%-22s %-8s %s\n",$4,$2,$NF}'
```

---

## 14) Makefile как драйвер (команды по кнопке)

```makefile
SHELL := /usr/bin/env bash
.DEFAULT_GOAL := help

lint: ## shellcheck
	shellcheck -x mytool.sh

run: ## запуск локально (пример)
	bash mytool.sh --src /data --dst /backup --mode safe

help: ## эта помощь
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' Makefile | awk 'BEGIN{FS=":.*?## "}{printf "  \033[36m%-12s\033[0m %s\n", $$1, $$2}'
```

---

## 15) Чеклист перед продом (прям по пунктам)

-  `shellcheck` чисто
-  `set -Eeuo pipefail`, `IFS=$'\n\t'`, traps + cleanup
-  валидация входа, `--dry-run`, понятный `usage`
-  абсолютные пути (особенно для cron/timer) или `Environment=PATH=…` в юните
-  ретраи/таймауты для сетевых/IO
-  файловый лок (`flock`) — не запускаемся параллельно
-  секреты в `EnvironmentFile`, права 600
-  логи в journald/файл + метка `SyslogIdentifier`
-  юнит с sandbox (NoNewPrivileges/ProtectSystem/ReadWritePaths)

---

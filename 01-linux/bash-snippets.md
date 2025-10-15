## 0) База: флаги безопасности

- `set -euo pipefail` — падать на ошибках/незаданых переменных/пайпах.
- `IFS=$'\n\t'` — адекватный разделитель (против пробелов в путях).
- **Никаких** `for f in $(ls)` — пользуй `find … -print0 | xargs -0 …`.

---

## 1) `find` — искать файлы и делать с ними

```bash
# найти по имени/маске (регистр нечувствителен)
find . -type f -iname "*.log"

# старше 30 дней и >100МБ
find /var/log -type f -size +100M -mtime +30

# удалить пустые каталоги (с конца)
find /data -type d -empty -delete

# удалить файлы по маске (сухой прогон → убрать echo)
find . -type f -name "*.tmp" -print -exec echo rm -f {} \;

# заменить текст во всех *.conf (безопасно, с бэкапом .bak)
find /etc -type f -name "*.conf" -exec sed -i.bak 's/foo/bar/g' {} +

# chmod +x всем *.sh
find . -type f -name "*.sh" -exec chmod +x {} +

# безопасная передача имён с пробелами
find . -type f -print0 | xargs -0 du -sh | sort -h
```

**Паттерны времени:**

- `-mtime +N` — старше N суток; `-mmin +N` — минут.
- `-newermt "2025-10-01"` — новее даты (круто для окон времени).

---

## 2) `grep` — фильтровать строки

```bash
# базовое: найти "ERROR" рекурсивно, показать номера строк
grep -Rin --color=always 'ERROR' /var/log

# по слову (границы), регистр нечувств.
grep -Riw 'failed' .

# исключить папки node_modules и .git
grep -Rin 'TODO' . --exclude-dir={node_modules,.git}

# Только имя файлов, где совпало
grep -Rl 'listen 80' /etc/nginx

# показать 3 строки контекста до/после
grep -RinC3 'panic' /var/log

# несколько паттернов
grep -E 'ERROR|WARN' app.log

# инверсия (всё, где нет 'DEBUG')
grep -v 'DEBUG' app.log
```

> На больших логах привыкай к `ripgrep (rg)` — быстрее. Но `grep` везде есть.

---

## 3) `sed` — быстрые правки

```bash
# замена в одном файле (все вхождения)
sed -i 's/old/new/g' file.txt

# замена только целого слова (границы \b)
sed -i 's/\<prod\>/stage/g' config.ini

# удалить строки, начинающиеся с #
sed -i '/^#/d' file.conf

# вставить строку после совпадения
sed -i '/\[Service\]/a Restart=always\nRestartSec=3' /etc/systemd/system/app.service

# заменить порт в nginx.conf (цифры после "listen ")
sed -i -E 's/(listen\s+)[0-9]+;/\18080;/' /etc/nginx/sites-enabled/site.conf

# комментировать строки, содержащие "PermitRootLogin"
sed -i '/^PermitRootLogin/s/^/# /' /etc/ssh/sshd_config
```

> На macOS: `sed -i '' 's/x/y/' file` (пустой арг. бэкапа).

---

## 4) `awk` — поля, агрегаты, мини-SQL

```bash
# вывести 1-е и 3-е поле, разделитель по умолчанию — пробелы
awk '{print $1, $3}' file.txt

# указать разделитель: двоеточие
awk -F: '{print $1, $3}' /etc/passwd

# сумма значений 2-го столбца
awk '{sum+=$2} END{print sum}' data.txt

# фильтр: поле 3 > 100
awk '$3 > 100 {print $0}' data.txt

# CSV: первый столбец уникально-отсортированный
awk -F, 'NR>1 {print $1}' file.csv | sort -u

# форматированный вывод (printf)
ss -tan | awk 'BEGIN{printf "%-22s %-12s\n","LADDR","STATE"} NR>1{printf "%-22s %-12s\n",$4,$2}'
```

**Мнемоника:** `awk 'условие {действие}'`. Если условие пустое — действие для каждой строки.

---

## 5) `xargs` — безопасно прокинуть список в команды

```bash
# удалить найденные файлы пачками (быстро и безопасно)
find . -type f -name "*.pyc" -print0 | xargs -0 -r rm -f

# проверить sha256 всех *.iso
find . -type f -name "*.iso" -print0 | xargs -0 -I{} sh -c 'sha256sum "{}"'

# параллельно (-P): сжать логи
ls *.log | xargs -r -n1 -P4 gzip
```

- `-0` / `-print0` — против пробелов/переводов строк в именах
- `-n1` — по одному аргументу на запуск
- `-P4` — 4 параллельных процесса
    

---

## 6) `jq` — JSON как взрослые

```bash
# красиво распечатать
jq . file.json

# вытащить поле
jq -r '.name' file.json

# отфильтровать массив объектов по условию
jq -r '.items[] | select(.status=="READY") | .id' data.json

# взять уникальные значения
jq -r '.items[].namespace' data.json | sort -u

# изменить значение и сохранить
jq '.config.port=8080' cfg.json > cfg.new.json && mv cfg.new.json cfg.json

# из curl → сразу фильтровать
curl -s https://api.example.com/v1/users | jq -r '.users[] | [.id,.email] | @tsv'
```

**Полезно:**

- `@csv`, `@tsv` — готовый вывод.
- `try … catch` — не падать на `null`.

---

## 7) Частые рецепты (copy-paste)

### Диски/место

```bash
# топ-20 тяжёлых директорий в /var (до 2 уровней)
du -h --max-depth=2 /var 2>/dev/null | sort -h | tail -n 20

# найти файлы >1ГБ, показать размер
find / -xdev -type f -size +1G -print0 | xargs -0 du -h | sort -h
```

### Логи

```bash
# последние ошибки за час (journald)
journalctl --since "1 hour ago" -p err..alert

# показать файлы, где встречается "OutOfMemory"
grep -Ril 'OutOfMemory' /var/log
```

### Тексто-обработка

```bash
# уникальные строки с подсчётом
sort file.txt | uniq -c | sort -nr | head

# оставить только 1-й столбец (CSV) уникально
awk -F, '{print $1}' data.csv | sort -u
```

### Сеть

```bash
# кто слушает порты
ss -ltnp | awk 'NR>1 {print $4, $6}' | sort -u

# вытащить IP из текста
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' file.txt | sort -u
```

### Массовые правки конфигов

```bash
# заменить "debug=true" → "debug=false" во всех *.conf под /etc/app
grep -RIl 'debug=true' /etc/app --include='*.conf' | xargs -r sed -i 's/debug=true/debug=false/g'
```

---

## 8) Производительность и “не стреляем себе в ногу”

- `grep -R` vs `ripgrep (rg)`: на гигантских деревьях `rg` быстрее, но `grep` везде.
- `-exec … {} +` у `find` обычно быстрее, чем `| xargs` (меньше процессов).
- Всегда делай **сухой прогон**:
    - вместо `rm` — `echo rm …`
    - `sed -i.bak` (с бэкапом) и/или на копии
- **Пробелы в путях:** всегда используем `-print0 | xargs -0` или `-exec … {} +`.
- **Границы слов:** в `grep -w`, в `sed` — `\<word\>`, в `awk` — `($1=="word")`.

---

## 9) Шаблоны для скриптов

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

log() { printf '%s %s\n' "[$(date -Iseconds)]" "$*" >&2; }

# пример: пройти по файлам безопасно
find "${1:-.}" -type f -name '*.log' -print0 |
  while IFS= read -r -d '' f; do
    log "processing: $f"
    gzip -9 "$f"
  done
```

---

## 10) Мини-шпаргалка по регэкспам

- Начало/конец строки: `^` / `$`
- Любой символ: `.`
- Множества: `[0-9]`, отрицание `[^A-Z]`
- Кванторы: `*` (0+), `+` (1+), `?` (0/1), `{m,n}`
- Группы и “или”: `(foo|bar)`
- В `sed -E` включай расширенный синтаксис (как у `egrep`).

---

## 11) Быстрый “тулсет” (если можно поставить)

- `ripgrep (rg)` — turbo-grep
- `fd` — turbo-find
- `sd` — человекочитаемый sed
- `yq` — YAML-версия jq

---

## 12) Проверка перед запуском (one-liner safety)

```bash
# посмотрим, КАКИЕ файлы затронем
find . -type f -name "*.tmp" -print

# покажем КАКУЮ замену сделаем (без -i)
sed -E 's/(listen\s+)[0-9]+;/\18080;/' /etc/nginx/nginx.conf | diff -u /etc/nginx/nginx.conf -

# что реально будет передано в xargs
find . -type f -name "*.log" -print0 | xargs -0 -n1 -I{} echo gzip "{}"
```

---

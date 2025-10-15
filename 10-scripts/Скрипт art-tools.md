# Скрипт art-tools (artcheck and artdelete) для быстрой проверки и удаления артефактов в OpenStack по списку ID
Мини-набор для массовой чистки артефактов в OpenStack.  
`artcheck` валидирует ID и готовит «чистый» список. `artdelete` удаляет — только после явного подтверждения.

---

## Что делает

- **`artcheck`** — проверяет существование ресурсов (`openstack <res> show <ID>`) и формирует `FILE.ok` с живыми ID. Показывает стату: всего / найдено / не найдено|timeout.
- **`artdelete`** — удаляет ресурсы из файла. Для `volume` при фейлах пробует `cinder reset-state` и повтор.

Поддерживаемые типы: `server`, `volume`, `port`, `image`.

---

## Требования

- Рабочий `openstack` CLI с валидными кредами и ролями, которые позволяют **читать** и **удалять** нужные ресурсы.
- Доступ к `cinder` для операций `reset-state` (только для volumes).

---

## Установка (локально в `$HOME`)

```bash
mkdir -p ~/.local/bin
# Сохраните скрипты под именами:
#   ~/.local/bin/artcheck
#   ~/.local/bin/artdelete

chmod +x ~/.local/bin/artcheck ~/.local/bin/artdelete

# (опционально) автодополнение bash
cat >> ~/.bashrc <<'BASH'
export PATH="$HOME/.local/bin:$PATH"

_art_completion() {
  local cur opts
  COMPREPLY=()
  cur="${COMP_WORDS[COMP_CWORD]}"
  opts="volume port server image --help -h"

  if [[ ${COMP_CWORD} -eq 1 ]]; then
    COMPREPLY=( $(compgen -W "$opts" -- "$cur") )
  elif [[ ${COMP_CWORD} -eq 2 ]]; then
    compopt -o filenames
    COMPREPLY=( $(compgen -f -- "$cur") )
  fi
}

complete -F _art_completion artdelete
complete -F _art_completion artcheck
BASH

# Применить
source ~/.bashrc
```

---

## Быстрый старт: базовый флоу из 2 шагов

1. **Проверка и подготовка списка**
```bash
# Пример: собрать все серверы проекта в файл
openstack server list -f value -c ID > servers.txt

# Прогнать валидацию (ожидает один ID на строку)
artcheck server servers.txt
# Результат: servers.txt.ok — только валидные ID
```

2. **Удаление**
```bash
# Удаление только после явного подтверждения 'yes'
artdelete server servers.txt.ok
```

---

## Использование

### `artcheck`

```bash
artcheck {port|server|volume|image} FILE
# Пример:
artcheck volume volumes.txt
```

**Выводит:**
- построчный статус (`Найден` / `Не найден или timeout`);
- сводку: `Всего / Найдено / Не найдено|timeout`;
- файл `FILE.ok` с живыми ID.

**Заметки:**

- Пустые строки пропускаются. Комменты `# ...` лучше не класть (могут улететь в stderr от `openstack`).
    
- Таймаут проверки — 15 секунд на ID.
    

### `artdelete`

```bash
artdelete {volume|port|server|image} FILE
# Примеры:
artdelete port ports.ok
artdelete image images.ok
```

**Поведение:**

- Перед стартом спросит подтверждение: напишите **`yes`**.
- Для каждого ID: пробует `openstack <res> delete <ID>`.
- Если `volume` не удаляется — делает:
    - `cinder reset-state --state available`
    - `cinder reset-state --reset-migration-status`
    - `cinder reset-state --attach-status detached`
    - повторяет удаление.

**Таймаут удаления:** 30 секунд на ID.

---

## Формат входного файла

- Один ID на строку.
- Без запятых, лишних пробелов и комментариев.

```text
1a2b3c4d-1111-2222-3333-abcdef123456
7e8f9a0b-4444-5555-6666-fedcba654321
9d8c7b6a-7777-8888-9999-112233445566
```

---

## Примеры получения списков

```bash
# Серверы по проекту
openstack server list -f value -c ID > servers.txt

# Все тома проекта
openstack volume list -f value -c ID > volumes.txt

# Порты по фильтру
openstack port list --project <project_id> -f value -c ID > ports.txt

# Образы пользователя
openstack image list --private -f value -c ID > images.txt
```

---

## Траблы и решения

- **`Файл '…' не найден.`** — проверь путь/имя файла.
- **`Неверный тип ресурса` / `Неизвестный ресурс`** — доступные: `server|volume|port|image`.
- **Много `timeout` в `artcheck`.**  
    Сеть/endpoint тупит. Запусти позже или сузь список.
- **`openstack ... delete` падает с 403/409.**  
    Ролей не хватает / ресурс занят. Для volume см. авто-reset в скрипте; для server — проверь attached volumes/port binding.
- **Не те креды/проект.**  
    Убедись в актуальном контексте (`OS_CLOUD`, `openstack token issue`, `openstack configuration show`).

---

## Подсказки по безопасности (aka не взорвать прод)

- Всегда запускай **`artcheck` → смотри `*.ok` → только потом `artdelete`**.
- Делай бэкап списков ID и логи консоли.
- Для образов/томов сначала проверь, что нет зависимостей (snapshots, attachments).
- Если сомневаешься — режь список на батчи и удаляй постепенно.
    

---

## Справка (как должно отображаться)

> У тебя в `artcheck` и `artdelete` help-тексты норм, но есть опечатки в примерах. Вот эталон:

### `artcheck --help`

```text
Usage:
  artcheck {port|server|volume|image} FILE

Examples:
  artcheck server ids.txt
```

### `artdelete --help`

```text
Usage:
  artdelete {volume|port|server|image} FILE

Examples:
  artdelete server ids.txt
```

---

## Всё, что надо для копипасты

**Валидация → удаление серверов:**

```bash
openstack server list -f value -c ID > servers.txt
artcheck server servers.txt
artdelete server servers.txt.ok
```

**Валидация → удаление томов:**

```bash
openstack volume list -f value -c ID > volumes.txt
artcheck volume volumes.txt
artdelete volume volumes.txt.ok
```
####  Скрипт artdelete
```bash
#!/usr/bin/env bash

if [[ "$1" == "--help" || "$1" == "-h" ]]; then
  cat <<'EOF'
Usage:
  delate.sh {volume|port|server|image} FILE

Examples:
  delate.sh server ids.txt
EOF
  exit 0
fi

RESOURCE="$1"
FILE="$2"

usage() {
  echo "Usage: $0 {volume|port|server|image} FILE"
  exit 1
}

[[ -z "$RESOURCE" || -z "$FILE" ]] && usage
[[ ! -f "$FILE" ]] && { echo "❌ Файл '$FILE' не найден."; exit 1; }

case "$RESOURCE" in
  volume|port|server|image) ;;
  *) echo "❌ Неизвестный ресурс '$RESOURCE'"; usage ;;
esac

echo "Удаление ресурсов типа: $RESOURCE"
echo "Файл: $FILE"
echo "-------------------------------"

read -r -p "Ты уверен, что хочешь удалить все объекты из файла? (напиши 'yes' для подтверждения): " CONFIRM
if [[ "$CONFIRM" != "yes" ]]; then
  echo "Отмена операции. Ничего не удалено."
  exit 0
fi

echo
echo "Подтверждено. Начинаем удаление..."
echo "-------------------------------"

while IFS= read -r ID; do
  [[ -z "$ID" ]] && continue
  echo "⏳ Удаляю $RESOURCE $ID"

  if timeout 30 openstack "$RESOURCE" delete "$ID" &>/dev/null; then
    echo "✅ Удалено: $ID"
  else
    echo "⚠️  Ошибка удаления: $ID"

    if [[ "$RESOURCE" == "volume" ]]; then
      echo "🔁 Пробую сбросить состояние для $ID..."
      cinder reset-state --state available "$ID"
      cinder reset-state --reset-migration-status "$ID"
      cinder reset-state --attach-status detached "$ID"

      sleep 2

      if timeout 30 openstack volume delete "$ID" &>/dev/null; then
        echo "✅ Удалено после сброса состояния: $ID"
      else
        echo "❌ Всё ещё не удаётся удалить: $ID"
      fi
    fi
  fi
done < "$FILE"

echo "-------------------------------"
echo "✅ Удаление завершено."
```

#### Скрипт artcheck 
```bash
#!/bin/bash

if [[ "$1" == "--help" || "$1" == "-h" ]]; then
  cat <<'EOF'
Usage:
  artcheck {port|server|volume} FILE

Examples:
  artcheck server ids.txt
EOF
  exit 0
fi

RES="$1"
INPUT_FILE="$2"
TMP_FILE=$(mktemp)
trap 'rm -f "$TMP_FILE"' EXIT

usage() {
  echo "Usage: $0 {port|server|volume|image} FILE"
  exit 1
}

[[ -z "$RES" || -z "$INPUT_FILE" ]] && usage
[[ ! -f "$INPUT_FILE" ]] && { echo "Файл '$INPUT_FILE' не найден."; exit 1; }

case "$RES" in
  port|server|volume|image) ;;
  *) echo "Неверный тип ресурса: $RES"; usage ;;
esac

TOTAL=0
FOUND=0
REMOVED=0

echo "🔍 Проверка $RES по ID (через 'openstack $RES show')"
echo "Файл: $INPUT_FILE"
echo "-------------------------------"

while IFS= read -r ID; do
    [[ -z "$ID" ]] && continue
    ((TOTAL++))
    if timeout 15 openstack "$RES" show "$ID" &>/dev/null; then
        echo "Найден: $ID"
        echo "$ID" >> "$TMP_FILE"
        ((FOUND++))
    else
        echo "Не найден или timeout: $ID"
        ((REMOVED++))
    fi
done < "$INPUT_FILE"

# не трогаем исходник — пишем рядом
mv "$TMP_FILE" "${INPUT_FILE}.ok"

echo "-------------------------------"
echo "Готово. Всего: $TOTAL | Найдено: $FOUND | Не найдено/timeout: $REMOVED"
echo "Живые ID записаны в: ${INPUT_FILE}.ok"
  ```
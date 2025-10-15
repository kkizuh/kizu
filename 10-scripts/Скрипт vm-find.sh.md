# vm-find.sh — быстрый поиск ВМ / FIP / портов / дисков в OpenStack

Инструмент для дежурных и SRE: на вход — имя/ID ВМ, FIP, список или файл; на выход — проект, сервер, адреса, FIP(+ID), порт(ы), группы безопасности и примонтированные тома. Работает в рамках указанного проекта или по всем проектам (`all`). Для FIP сам находит проект, порт и связанную ВМ.

---

## Требования

- Установлен и сконфигурирован `openstack` CLI с валидными cred’ами.
- Права на чтение ресурсов проекта(ов) (как минимум `member`/`reader`).

## Установка

```bash
vim vm-find.sh         # вставьте код
chmod +x vm-find.sh
# (опционально) добавить в PATH:
# sudo mv vm-find.sh /usr/local/bin/vm-find && vm-find ...
```

## Синтаксис

```bash
./vm-find.sh <file|name|id|comma,list|floating_ip> [project_id|project_name|all]
```

- 1-й аргумент — что искать: файл, одиночное значение, список через запятую, FIP.
- 2-й аргумент — область поиска:
    - `all` — по всем доступным проектам;
    - `<project_id>` или `<project_name>` — в конкретном проекте;
    - не указан — используется «текущий» контекст CLI.

## Форматы входа (1-й аргумент)

- Имя ВМ: `ceph-node-1`
- ID ВМ: `f175683e-3335-4859-b752-27c23bef4b25`
- FIP: `193.141.231.114`
- Список через запятую: `vm1,vm2,193.141.231.114`
- Файл (по одной записи в строке): `vms.txt`

## Примеры

1. По имени везде:
```bash
./vm-find.sh ceph-node-1 all
```

2. По ID в конкретном проекте:
```bash
./vm-find.sh f175683e-3335-4859-b752-27c23bef4b25 proj-ozo4v4ttrs
```

3. По FIP (сам найдёт проект, порт и ВМ):
```bash
./vm-find.sh 193.141.231.114 all
```

4. Файл со смешанными типами (имена/ID/FIP):
```bash
./vm-find.sh vms.txt all
```

5. Список через запятую в проекте:
```bash
./vm-find.sh ceph-node-3,ceph-node-2 proj-ozo4v4ttrs
```

## Что выводит (по каждой найденной сущности)

- `PROJECT`: `<name> (<project_id>)`
- `SERVER_ID`
- `NAME`
- `ADDRESSES`
- `FIP` и `FIP_ID` (если есть)
- `ACCESS_IPV4` (менеджмент/фиксированный IP из `server show`)
- `ACCESS_IPV4_PORT_ID` и все `PORT_ID`, найденные по фиксированному IP
- `SECURITY_GROUPS`
- `VOLUME_ID` (все примонтированные к ВМ тома)

## Поведение

- Если вход — **FIP**: скрипт находит его проект, фиксированный IP, связанный порт и ВМ, затем печатает полную сводку по ВМ.
- Если вход — **имя/ID**: ищет в заданной области (`project`/`all`/текущий контекст).
- Если вход — **файл** или **список**: обрабатывает каждую строку/элемент по тем же правилам.
- Пустые строки и строки, начинающиеся с `#`, игнорируются.

## Траблы и решения

- **`❌ not found` по имени** — чаще всего не тот проект. Запустите с `all`:
```bash
    ./vm-find.sh <name> all
    ```
- **Проблемы с правами** — убедитесь, что токен/роль позволяет читать `server list/show`, `port list/show`, `floating ip list/show`, `volume attachments`.
- **Несовпадение CLI-контекста** — проверьте текущий cloud/profile (`OS_CLOUD`, `openstack config show`), либо явно задайте проект вторым аргументом.
- **Лишние пробелы/CRLF во входе** — скрипт сам триммит пробелы и `\r`, но для файлов с Windows-строками лучше прогнать через `dos2unix`.

## Заметки

- В тексте и примерах используйте одно имя файла: **`vm-find.sh`** (у вас встречался `vm-find2.sh` — это просто опечатка).
- Скрипт печатает несколько `PORT_ID`, если к фиксированному IP привязано несколько портов (редко, но бывает в сложных сетевых схемах).
- Вывод размечен «ключ: значение», чтобы удобно grep’ать:
```bash
    ./vm-find.sh vms.txt all | grep -E '^(PROJECT|SERVER_ID|FIP:|PORT_ID|VOLUME_ID):'
```

Скрипт vm-find.sh
```bash
#!/bin/bash
# usage: ./vm-find2.sh <file|name|id|comma,list|floating_ip> [project_id|all]
# выводит: PROJECT, SERVER_ID, NAME, ADDRESSES, FIP(+ID), ACCESS_IPV4(+PORT_ID), SECURITY_GROUPS, VOLUME_ID

set -e

INPUT="$1"
SCOPE="${2:-}"

[ -z "$INPUT" ] && { echo "Usage: $0 <file|name|id|comma,list|ip> [project_id|all]"; exit 1; }

proj_info(){
  local P_IN="$1"
  local P_ID P_NAME
  P_ID="$(openstack project show "$P_IN" -f value -c id 2>/dev/null || true)"
  P_NAME="$(openstack project show "$P_IN" -f value -c name 2>/dev/null || true)"
  echo "${P_NAME}|${P_ID}"
}

trim(){
  echo "$1" | sed 's/^[[:space:]]\+//; s/[[:space:]]\+$//'
}

print_vm(){
  local SID="$1"
  local PROJ_HINT="$2"

  local PROJ_ID PROJ_NAME
  if [ -n "$PROJ_HINT" ] && [ "$PROJ_HINT" != "all" ]; then
    IFS='|' read -r PROJ_NAME PROJ_ID <<<"$(proj_info "$PROJ_HINT")"
    if [ -z "$PROJ_ID" ]; then
      PROJ_ID="$PROJ_HINT"
      IFS='|' read -r PROJ_NAME _ <<<"$(proj_info "$PROJ_ID")"
    fi
  else
    PROJ_ID="$(openstack server show "$SID" -f value -c project_id 2>/dev/null || true)"
    IFS='|' read -r PROJ_NAME _ <<<"$(proj_info "$PROJ_ID")"
  fi

  local NAME AIP ADDR SECG
  NAME="$(openstack server show "$SID" -f value -c name 2>/dev/null || true)"
  AIP="$(openstack server show "$SID" -f value -c accessIPv4 2>/dev/null || true)"
  ADDR="$(openstack server show "$SID" -f value -c addresses 2>/dev/null || true)"
  SECG="$(openstack server show "$SID" -f value -c security_groups 2>/dev/null || true)"

  [ -n "$PROJ_ID" ] && echo "PROJECT: ${PROJ_NAME:-<unknown>} ($PROJ_ID)"
  echo "SERVER_ID: $SID"
  [ -n "$NAME" ] && echo "NAME: $NAME"
  [ -n "$ADDR" ] && echo "ADDRESSES: $ADDR"

  local FIP=""
  if echo "$ADDR" | grep -q ","; then
    FIP="$(echo "$ADDR" | awk -F',' '{gsub(/ /,""); print $2}')"
  fi
  if [ -z "$FIP" ] && [ -n "$AIP" ] && [ -n "$PROJ_ID" ]; then
    FIP="$(openstack floating ip list --fit --project "$PROJ_ID" | awk -v ip="$AIP" 'index($0, ip){print $2; exit}' 2>/dev/null || true)"
  fi
  if [ -n "$FIP" ]; then
    echo "FIP: $FIP"
    local FIP_ID
    FIP_ID="$(openstack floating ip show "$FIP" -f value -c id 2>/dev/null || true)"
    [ -n "$FIP_ID" ] && echo "FIP_ID: $FIP_ID"
  fi

  [ -n "$AIP" ] && echo "ACCESS_IPV4: $AIP"

  if [ -n "$AIP" ] && [ -n "$PROJ_ID" ]; then
    local PORT_IDS
    PORT_IDS="$(openstack port list --fit --project "$PROJ_ID" | grep -F "$AIP" | awk '{print $2}' 2>/dev/null || true)"
    if [ -n "$PORT_IDS" ]; then
      local FIRST=1
      while read -r P; do
        [ -z "$P" ] && continue
        if [ "$FIRST" -eq 1 ]; then
          echo "ACCESS_IPV4_PORT_ID: $P"
          FIRST=0
        fi
        echo "PORT_ID: $P"
      done <<< "$PORT_IDS"
    fi
  fi

  [ -n "$SECG" ] && echo "SECURITY_GROUPS: $SECG"

  openstack server show "$SID" -f value -c volumes_attached \
    | awk -F"'" '/id=/{print "VOLUME_ID: "$2}'

  echo
}

emit_items(){
  if [ -f "$INPUT" ]; then
    while IFS= read -r LINE || [ -n "$LINE" ]; do
      LINE="$(trim "${LINE%$'\r'}")"
      [ -z "$LINE" ] && continue
      case "$LINE" in "#"*) continue ;; esac
      echo "$LINE"
    done < "$INPUT"
  else
    echo "$INPUT" | tr ',' '\n' | while read -r ITEM; do
      ITEM="$(trim "${ITEM%$'\r'}")"
      [ -n "$ITEM" ] && echo "$ITEM"
    done
  fi
}

while read -r ITEM; do
  echo ">> $ITEM"

  FIP_ID_TRY="$(openstack floating ip show "$ITEM" -f value -c id 2>/dev/null || true)"
  if [ -n "$FIP_ID_TRY" ]; then
    F_PID="$(openstack floating ip show "$ITEM" -f value -c project_id 2>/dev/null || true)"
    F_FIX="$(openstack floating ip show "$ITEM" -f value -c fixed_ip_address 2>/dev/null || true)"
    IFS='|' read -r F_PNAME F_PID_N <<<"$(proj_info "$F_PID")"

    echo "  FIP: $ITEM"
    echo "  FIP_ID: $FIP_ID_TRY"
    [ -n "$F_PID_N" ] && echo "  PROJECT: ${F_PNAME:-<unknown>} ($F_PID_N)"
    [ -n "$F_FIX" ] && echo "  FIXED_IP: $F_FIX"

    if [ -n "$F_PID_N" ] && [ -n "$F_FIX" ]; then
      PORT_IDS="$(openstack port list --fit --project "$F_PID_N" | grep -F "$F_FIX" | awk '{print $2}' 2>/dev/null || true)"
      if [ -n "$PORT_IDS" ]; then
        FIRST_PORT="$(echo "$PORT_IDS" | head -n1)"
        echo "  PORT_ID: $FIRST_PORT"
        SID="$(openstack port show "$FIRST_PORT" -f value -c device_id 2>/dev/null || true)"
        [ -n "$SID" ] && print_vm "$SID" "$F_PID_N"
      else
        for SID in $(openstack server list --project "$F_PID_N" -f value -c ID); do
          AIP_S="$(openstack server show "$SID" -f value -c accessIPv4 2>/dev/null || true)"
          if [ "$AIP_S" = "$F_FIX" ]; then
            print_vm "$SID" "$F_PID_N"
            break
          fi
        done
      fi
    fi
    echo
    continue
  fi

  if openstack server show "$ITEM" >/dev/null 2>&1; then
    print_vm "$ITEM" "$SCOPE"
    continue
  fi

  SID=""
  if [ "$SCOPE" = "all" ]; then
    SID="$(openstack server list --all -f value -c ID --name "$ITEM" | head -n1 || true)"
  elif [ -n "$SCOPE" ]; then
    SID="$(openstack server list --project "$SCOPE" -f value -c ID --name "$ITEM" | head -n1 || true)"
  else
    SID="$(openstack server list -f value -c ID --name "$ITEM" | head -n1 || true)"
  fi

  if [ -n "$SID" ]; then
    print_vm "$SID" "$SCOPE"
  else
    echo "   ❌ not found"
    echo
  fi
done < <(emit_items)
```
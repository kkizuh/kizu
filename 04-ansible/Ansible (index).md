playbook rule 
- Понимание принципов функционирования и отличий основных модулей (Shell, raw, command)
- Написание простых ролей по установке и конфигурированию, включая модули по тестированию (линтеры - возможно). Пример: на 100 серверах обновить ключи/сменить пароли/ добавить пользователя
- Через PR/MR вносить в существующие роли правки
	- Прокатывать роли на существующей инфраструктуре
- - Скелет роли (tasks/handlers/templates/vars) + пример inventory.
- Чек-лист идемпотентности и safety флагов.
  https://www.youtube.com/watch?v=Ck1SGolr6GI
- https://docs.ansible.com/
в
# Ansible (index)

Мини-оглавление-шпаргалка: минимум теории, максимум практики.

---

## 1) Быстрый старт (1 экран)

```bash
# установка
pipx install ansible-core            # или: pip install ansible-core
ansible --version

# инвентори (ini)
cat > hosts.ini <<'EOF'
[web]
10.0.0.11
10.0.0.12

[db]
10.0.0.21

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
EOF

# пинг модулем ping
ansible -i hosts.ini all -m ping

# запуск простого плейбука
ansible-playbook -i hosts.ini site.yml
```

---

## 2) Модули: **command vs shell vs raw**

- `command`: выполняет бинарник **без оболочки**. Надёжно, безопасно. **Без** редиректов/pipe.
    
    ```yaml
    - command: /usr/bin/apt-get update
    ```
    
- `shell`: через `/bin/sh` (**нужен** для `|`, `>`, `&&`, переменных, heredoc).
    
    ```yaml
    - shell: "echo '{{ item }}' >> /etc/motd"
    ```
    
- `raw`: **вообще без Python/модулей**, прямой ssh. Полезно, когда на хосте нет Python.
    
    ```yaml
    - raw: "apt-get -y install python3"
    ```
    

> Правило: по умолчанию `command`; если нужна оболочка — `shell`; если Python не установлен — `raw`.

Полезные альтернативы с идемпотентностью:

- `package`, `apt`, `dnf`, `yum` — вместо `apt-get …`
    
- `user`, `authorized_key`, `copy`, `template`, `service`, `lineinfile`, `blockinfile`, `file`, `git`, `unarchive`.
    

---

## 3) Каркас роли (skeleton)

```bash
ansible-galaxy init roles/base
tree roles/base
roles/base/
├── defaults/main.yml      # значения "по умолчанию" (самая низкая приор.)
├── vars/main.yml          # жёсткие значения (избегай без нужды)
├── tasks/main.yml         # основной сценарий
├── handlers/main.yml      # notify -> restart/reload
├── templates/             # *.j2 (Jinja2)
├── files/                 # статические файлы
└── meta/main.yml          # зависимости роли
```

Пример `tasks/main.yml`:

```yaml
- name: Ensure user exists
  user:
    name: "{{ base_user }}"
    shell: /bin/bash

- name: Push ssh key
  authorized_key:
    user: "{{ base_user }}"
    key: "{{ lookup('file', 'files/id_ed25519.pub') }}"

- name: Configure motd
  template:
    src: motd.j2
    dest: /etc/motd
  notify: Restart sshd

# handlers/main.yml
- name: Restart sshd
  service:
    name: ssh
    state: restarted
```

---

## 4) Инвентори: примеры (INI/YAML)

**INI**

```ini
[web]
web01 ansible_host=10.0.0.11 env=prod
web02 ansible_host=10.0.0.12 env=prod

[db]
db01 ansible_host=10.0.0.21

[all:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

**YAML (inventory)**

```yaml
all:
  vars:
    ansible_user: ubuntu
  children:
    web:
      hosts:
        web01: { ansible_host: 10.0.0.11 }
        web02: { ansible_host: 10.0.0.12 }
    db:
      hosts:
        db01: { ansible_host: 10.0.0.21 }
```

---

## 5) Типовые плейбуки (100 серверов)

### a) Обновить **authorized_keys** всем пользователям группы

```yaml
# playbooks/update_keys.yml
- hosts: all
  become: true
  vars:
    users:
      - name: deploy
        pubkey_path: files/deploy.pub
      - name: admin
        pubkey_path: files/admin.pub
  tasks:
    - name: Ensure users exist
      user:
        name: "{{ item.name }}"
      loop: "{{ users }}"

    - name: Push keys
      authorized_key:
        user: "{{ item.name }}"
        state: present
        key: "{{ lookup('file', item.pubkey_path) }}"
      loop: "{{ users }}"
```

Запуск:

```bash
ansible-playbook -i hosts.ini playbooks/update_keys.yml --limit web
```

### b) Сменить пароли (через хэш)

```yaml
# vars/users.yml
users:
  - name: admin
    password_hash: "{{ 'NewP@ssw0rd' | password_hash('sha512') }}"
---
# playbooks/change_passwords.yml
- hosts: all
  become: true
  vars_files: [vars/users.yml]
  tasks:
    - user:
        name: "{{ item.name }}"
        password: "{{ item.password_hash }}"
      loop: "{{ users }}"
```

### c) Добавить пользователя с sudo

```yaml
- hosts: all
  become: true
  tasks:
    - user:
        name: deploy
        groups: sudo
        append: true
        shell: /bin/bash
    - copy:
        content: "deploy ALL=(ALL) NOPASSWD:ALL\n"
        dest: /etc/sudoers.d/deploy
        mode: '0440'
        validate: 'visudo -cf %s'
```

---

## 6) Идемпотентность + safety чек-лист

- ✅ Используй модули вместо `shell/command`, где возможно.
    
- ✅ Указывай `creates:` / `removes:` для `command`/`shell` (чтобы не перезапускать зря).
    
- ✅ `changed_when:` / `failed_when:` — корректный статус.
    
- ✅ `validate:` для конфигов (`visudo`, `nginx -t`, `sshd -t`).
    
- ✅ `check_mode: yes` (или `--check`) — dry-run.
    
- ✅ `serial:`/`max_fail_percentage:` — деплой волнами.
    
- ✅ `--limit` — ограничивай зону действия.
    
- ✅ `retries` + `delay` + `until` — ожидание готовности.
    
- ✅ Не хранить секреты в гите → **ansible-vault**.
    

Примеры флагов запуска:

```bash
ansible-playbook site.yml --check --diff                # dry-run + показать изменения
ansible-playbook site.yml --limit web --serial 20%      # по 20% хостов за раз
ansible-playbook site.yml --start-at-task "Push keys"   # с конкретной задачи
```

---

## 7) Lint/тесты

```bash
pipx install ansible-lint yamllint molecule
ansible-lint
yamllint .
# molecule для ролей (локальные проверки/контейнеры)
molecule init role base_role -d docker
molecule test
```

---

## 8) Vault (секреты)

```bash
# создать зашифрованный файл
ansible-vault create group_vars/all/vault.yml
# редактировать
ansible-vault edit group_vars/all/vault.yml
# запуск с паролем/файлом
ansible-playbook -i hosts.ini site.yml --ask-vault-pass
# или
ansible-playbook -i hosts.ini site.yml --vault-password-file ~/.vault_pass.txt
```

Использование:

```yaml
# group_vars/all/vault.yml (зашифровано)
vault_db_password: "s3cr3t"
---
# vars
db_password: "{{ vault_db_password }}"
```

---

## 9) Ревью и PR/MR флоу (коротко)

1. Ветка → изменения роли/плейбука.
    
2. `ansible-lint`, `yamllint`, `molecule test` в CI.
    
3. PR/MR → ревью (проверить идемпотентность, `--check`, `--diff`).
    
4. Прогон на **stage** (`--limit` + `serial`).
    
5. Промо на **prod** волнами.
    

---

## 10) Полезные приёмы

- **Факты**: `setup` или `gather_facts: false` для ускорения, когда не нужно.
    
- **become**: поднимай привилегии точечно у задач, а не глобально.
    
- **delegate_to**: делать действия на другом хосте (напр., балансировщик).
    
- **handlers + notify**: рестарт сервисов только при изменениях.
    
- **with_fileglob / loop_control**: аккуратные циклы.
    

---

## 11) Мини-шаблон `site.yml`

```yaml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - role: base
      vars:
        base_user: "deploy"

- hosts: web
  roles:
    - role: nginx_role

- hosts: db
  roles:
    - role: postgres_role
```

---

## 12) Ссылки

- Docs: [https://docs.ansible.com/](https://docs.ansible.com/)
    
- Видео-разбор (понятно и по делу): [https://www.youtube.com/watch?v=Ck1SGolr6GI](https://www.youtube.com/watch?v=Ck1SGolr6GI)
    

> Хватай этот index как входную точку: сверху — команды и различия модулей; ниже — каркас роли, инвентори, готовые плейбуки на 100 серверов, чек-лист безопасности/идемпотентности, lint и vault.
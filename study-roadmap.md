# 0) Приоритеты (что даст больше всего за меньшее время)

**P0 (база, без этого дальше больно):**

- Linux + сеть: `systemd`, логи, процессы, файловая система, `ss/ip/route`, DNS.
- Docker: образ → контейнер → сеть/тома → compose.
- Git + GitHub/Gitea: ветки, PR/MR, тэги, релизы

**P1 (практика SRE/DevOps):*
- Kubernetes (только essentials): `kubectl`, Deploy/Service/Ingress, rollout/rollback, HPA.
- CI/CD: типовой пайплайн (build→test→scan→sign→push→deploy), контейнерный релиз    
- IaC: Ansible (provision) + Terraform (VPC/VM/SG/Key/Volume базо 

**P2 (повысит ценность):**

- Мониторинг/Логи: Prometheus/Alertmanager, Loki/Fluent Bit, Grafana (3–5 дашбордов).
    
- Cloud/OpenStack: проекты/квоты/сети/FIP/volume attach (оперативные команды).
    
- Безопасность: `USER` в Docker, capabilities/seccomp, secrets, RBAC в k8s (минимум).
    
- БД: Postgres базовые операции + бэкап/restore.
    

---

# 1) Маршрут на 4 недели (можно сжимать/растягивать)

## Неделя 1 — Linux + Сеть + Git (P0)

**Цели:** уверенно читать/чинить логи, смотреть сеть, управлять сервисами.

- Ежедневно: 30–45 мин “диагностика”: `journalctl`, `dmesg`, `top/btop`, `ss -lntup`, `ip r`, `dig/nslookup`.
    
- Практика:
    
    - Написать 2–3 systemd-unit’а (simple/oneshot/timer) + логи в `journalctl`.
        
    - Настроить UFW/nftables «deny-all + allow 22/80/443».
        
    - Скрипт бэкапа (`tar|rsync`) + ротация логов.
        
- Git: ветка → PR → squash-merge, релизный тэг.
    

**Мини-проект:** “Веб-сервис + таймер”: systemd-service для простого HTTP-скрипта + timer для бэкапа.

---

## Неделя 2 — Docker + Compose (P0→P1)

**Цели:** собирать минимальные образы, работать non-root, публиковать.

- Dockerfile: multi-stage, `USER 10001`, `HEALTHCHECK`, labels, slim/distroless.
    
- Compose: `web+app+db+redis`, `.env`, volumes, depends_on.
    
- Registry: login, tag, push/pull, **digest pinning**, полиси «no `latest`».
    
- Безопасность: read-only rootfs, drop-cap, секреты (env/file/compose).
    

**Мини-проект:** “Стек блогика”: `nginx` (reverse proxy) + `app` + `postgres` в compose, миграции при старте, healthchecks.

---

## Неделя 3 — Kubernetes основы (P1)

**Цели:** деплой/обновление/откат, базовая сеть и HPA.

- Локально: `k3d` или `kind`.
    
- Манифесты: Deployment, Service (ClusterIP/NodePort), Ingress, HPA, PDB.
    
- Rollout: `kubectl rollout status/history/undo`, `--record`.
    
- Траблшут: `describe`, `events`, `logs -f`, `exec`, `top pods`, `kubectl get --raw='/readyz?verbose'`.
    

**Мини-проект:** задеплоить прошлый блог-стек в k8s + Ingress + HPA по CPU, провести **canary** через 2 Deployment’а или через weights в Ingress-контроллере.

---

## Неделя 4 — CI/CD + IaC + Мониторинг (P1→P2)

**Цели:** автоматизировать билд, проверку и выкладку, наблюдаемость.

- CI/CD: Actions/GitLab CI — build→trivy→SBOM→cosign→push→helm deploy (staging→prod, manual gate).
    
- Ansible: базовые роли (user, sshd, firewall, docker
    
- Terraform: VPC/Network/Subnet + VM + SG + Key + Output; `tflint`/`tfsec`.
- Мониторинг: Prometheus + node-exporter + blackbox; Grafana: дашборды ошибок/латентности; Alertmanager: 2–3 правила (без спама).

**Мини-проект:** “Пайплайн на один файл”: CI, который собирает образ, сканирует, подписывает, пушит и раскатывает helm-релиз в локальный k8s.

---

# 2) Формат занятий

**Ежедневно (60–90 мин):**

- 10 мин — “разминка” (3–5 команд из шпор).
- 40–60 мин — практическая задача (микро-проект).
- 10–15 мин — “капсула памяти”: фиксируешь в свою wiki: что делал, 3 ошибки, 1 правило.

**Еженедельно:**

- 1 демо себе: «что поднял/сломал/починил».
- 1 ретро: “что ускоряет → что мешает → что выбросить”.

---

# 3) Чек-листы «готов/не готов»

**Linux/Sys:**

-  Могу найти причину падения сервиса в `journalctl`.
-  Умею писать unit/timer и понимать `ExecStartPre/Restart/Limit*`.
-  Настрою SSH (no-root-login, ключи, fail2ban/ufw)

**Docker/Compose:**
-  Собираю образ <300MB с multi-stage, non-root, healthcheck.
-  Понимаю тома vs bind mounts, сети bridge/host, порты.
-  Пушу в registry с тегом и **digest**, без `latest`.    

**Kubernetes:**

-  Создам Deploy/Service/Ingress/HPA, сделаю rollout/undo.


-  Найду ImagePullBackOff/CrashLoop/Pending и починю.
-  Покажу `events`, объясню readiness/liveness.

**CI/CD:**
-  Пайплайн падает на HIGH/CRITICAL (trivy).
-  Генерю SBOM, подписываю образ (cosign), verify на входе.
-  Helm-релиз в стейдж, ручной gate в прод.

**IaC:**
-  Ansible роль: пользователь/ssh/ufw/docker.
-  Terraform: сеть+ВМ+SG; `plan` чистый, валидаторы проходят.

**Мониторинг:**

-  3 панели в Grafana: ошибки (rate), латентность (p95), saturation (CPU/RAM).    
-  Алерты с задержкой/дедупликацией (без спама).

---

# 4) Мини-проекты (готовые идеи)

1. **Web+DB на compose** с миграциями и healthcheck’ами.
2. **Тот же стек в k8s** (Ingress + HPA + canary).
3. **CI/CD полностью**: build→scan→sign→deploy.    
4. **Мониторинг стека**: метрики/логи + алерты.

---

# 5) Учёба «на телефоне/в дороге»

- Читать свои шпаргалки (ты их уже делаешь) + карточки (Anki): 10–15 карт в день.
- Termux/ssh к своему серверу: `kubectl get`, `docker ps`, `journalctl -xe` (набивать мышцу глаз).    
- 1 “why” в день: выбрать лог/ошибку и разобрать 5 минут.

---

# 6) Шаблоны ответов на собес (по 1 фразе)

- **“Разница контейнер/ВМ?”** — namespaces + cgroups vs гипервизор, общий ядро vs изолированное.
- **“Rollout сломался — что делаешь?”** — `kubectl rollout undo`, смотрю `events/logs`, фиксирую `image: digest`.
- **“Где хранить секреты?”** — Vault/CI-secrets/KMS, не в `ENV/ARG`, в k8s — `SealedSecrets`/External Secrets.
- **“Почему non-root в контейнере?”** — принцип наименьших привилегий, меньше blast radius, совместимость с PSP/OPA.

---

Если хочешь, могу свернуть это в **один чек-лист** под печать (A4) и **один compose/k8s стартовый репо** (скелет с makefile/ci), но и в таком виде можно сразу идти делать мини-проекты.
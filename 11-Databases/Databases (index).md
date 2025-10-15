# Databases

## Быстрый старт

- [[SQL (cheatsheet)|SQL: SELECT/INSERT/UPDATE/DELETE, JOIN, агрегаты]]
- [[Clients (psql-mysql-sqlite)|Клиенты: psql/mysql/sqlite3, подключение]]
- [[Docker (templates)|Docker Compose для PG/MySQL + админки]]

## Администрирование СУБД
- [[PostgreSQL (admin)|PostgreSQL: роли, схемы, конфиги, pg_hba, EXPLAIN]]
- [[MySQL (admin)|MySQL/MariaDB: пользователи/привилегии, конфиги, InnoDB]]
- [[SQLite (notes)|SQLite: файлы, WAL, ограничения]]

## Производительность и дизайн
- [[Performance (design)|Индексы, планы запросов, N+1, пагинация (keyset)]]
- [[Query patterns (cte-window)|CTE, оконные функции, подзапросы, upsert]]
- [[Explain (cookbook)|EXPLAIN/ANALYZE: разбор типовых кейсов]]

## Транзакции и изоляция
- [[Transactions (isolation)|ACID, уровни изоляции, блокировки, deadlocks]]

## Резервные копии и восстановление
- [[Backup (ops)|pg_dump/pg_restore, mysqldump/mysqlpump, PITR, проверка DR]]

## Безопасность и доступ
- [[Security (access)|RBAC, минимальные права, TLS, секреты, аудит]]

## Миграции и CI/CD
- [[Migrations (tools)|Flyway/Liquibase: версии, checksum, пайплайны]]
- [[DB Dev workflow|Code review схемы, миграции, фикстуры, тестовые данные]]

## Мониторинг и эксплуатация
- [[Monitoring (db)|pg_stat_statements, slow_query_log, экспортёры]]
- [[Connection pooling|PgBouncer/ProxySQL: пулы подключений]]

## Полезные ресурсы

1. [PostgreSQL Docs ](https://www.postgresql.org/docs/)— планировщик и индексация
2. [Use The Index, Luke!](https://use-the-index-luke.com/) — практики по индексам и запросам



- Виды баз и отличие SQL и NoSQL БД
- Создание/удаление/модификация БД
- Бэкапирование
- Кластеризация
- Что такое пуллеры коннектов и для чего они нужны
- Redis - общее понимание работы, состава, принципы работы (есть Standalon, а есть Sentinel)
- - Бэкапы: PITR vs snapshot, проверка восстановления.

- Быстрый triage: «встал MySQL/PG — что смотрим по шагам».
## Быстрый старт
- [[prometheus-quickstart|Prometheus + node_exporter]] — метрики хоста за 5 минут.
- [[grafana-quickstart|Grafana]] — подключить Prometheus, импортнуть дашборд **1860**.

## Метрики (Prometheus)
- [[promql-cookbook|PromQL Cookbook]] — ошибки, p95/p99, saturation, throttling.
- [[slo-template|SLI/SLO + алерты]] — error budget, правила без спама.
- [[alertmanager-routing|Alertmanager]] — маршрутизация, quiet hours, dedup.

## Логи
- [[logs-elk-loki|ELK vs Loki]] — быстрые рецепты: Filebeat/Logstash/ES/Kibana или Promtail/Loki/Grafana.
- [[log-patterns|Паттерны логов]] — уровни, кореляция с метриками, парсинг.

## Трейсинг
- [[otel-tempo|OpenTelemetry + Tempo/Jaeger]] — распределённые трассы, связка с метриками.

## Интеграции
- [[k8s-metrics|K8s метрики]] — kube-state-metrics, cAdvisor, готовые дашборды.
- [[exporters|Экспортёры]] — nginx/mysql/postgres/node_exporter и т.п.

## Эксплуатация
- [[capacity-retention|Хранение и ретеншн]] — объёмы, downsampling, VM/Thanos.
- [[runbooks|Ранбуки]] — что делать при «RPS упал / p95 взлетел / диски 90%+».

## Zabbix (по необходимости)
- [[zabbix-quick|Zabbix quickstart]] — когда нужен all-in-one: агенты, SNMP, триггеры.

## Готовые панели/сниппеты
- [[dashboards|Ссылки на дашборды]] — Node Exporter 1860, K8s, Nginx, Postgres.
- [[alerts-snippets|Шаблоны алертов]] — page/ticket уровни, примеры expr/for/labels.
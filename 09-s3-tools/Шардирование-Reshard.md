## Что такое «шарды» и зачем

Индекс бакета (список ключей/метаданных) режется на N шардов. Когда в одном шарде слишком много записей, листинг/лайфсайкл/удаления начинают тупить. Динамическое ресхардинг-ядро RGW умеет **автоматически** добавлять шарды, чтобы разгрузить индекс. ([docs.ceph.com](https://docs.ceph.com/en/latest/radosgw/dynamicresharding/?utm_source=chatgpt.com "RGW Dynamic Bucket Index Resharding - Ceph Documentation"))

---

## Когда бить тревогу (симптомы)

- Падает скорость `ListObjectsV2`/`aws s3 ls`, «зависают» `sync`, lifecycle/expirer косячит.
- На версионированных бакетах — странные задержки из-за мусорных OLH-записей. ([docs.ceph.com](https://docs.ceph.com/en/latest/radosgw/dynamicresharding/?utm_source=chatgpt.com "RGW Dynamic Bucket Index Resharding - Ceph Documentation"))

---

## Диагностика (быстро)

```bash
# Сколько шардов у бакета
radosgw-admin bucket stats --bucket <bkt> | jq '{bucket:.bucket, shards:.num_shards, objects:.usage}'
# Общий health кластера (важно при ресхарде)
ceph -s
```

Проверь, включён ли динамический ресхардинг и его пороги:

```bash
ceph config get client.rgw rgw_dynamic_resharding
ceph config get client.rgw rgw_max_objs_per_shard
# (дефолт ~100k объектов на шарду)
```

Если значение порога мало для твоей нагрузки — подними. Динамический ресхардинг по умолчанию включён; он в фоне будет добавлять шарды, когда одна шарда перевалит за порог. ([Ceph](https://ceph.io/en/news/blog/2017/new-luminous-rgw-dynamic-bucket-sharding/?utm_source=chatgpt.com "New in Luminous: RGW dynamic bucket sharding"))

---

## Как чинить (3 пути)

### 1) Дать «автомату» сделать своё дело (предпочтительно)

Оставляешь `rgw_dynamic_resharding=true`, адекватный `rgw_max_objs_per_shard`, и даёшь сервису время. Это безопаснее и без ручных танцев. ([Ceph](https://ceph.io/en/news/blog/2017/new-luminous-rgw-dynamic-bucket-sharding/?utm_source=chatgpt.com "New in Luminous: RGW dynamic bucket sharding"))

### 2) Запланировать ручной ресхард (онлайн, через очередь)

Подходит, когда конкретный бакет уже «задушен» и ждать нельзя:

```bash
# Добавить задачу в очередь ресхарда
radosgw-admin reshard add --bucket <bkt> --num-shards 64

# Посмотреть очередь
radosgw-admin reshard list

# Запустить обработку очереди (если нет демона, делающего это в фоне)
radosgw-admin reshard process

# Статус конкретного бакета
radosgw-admin reshard status --bucket <bkt>
```

Это переразобьёт индекс на большее число шардов. Делай в окно — операция тяжёлая, может бить по латентности. ([people.redhat.com](https://people.redhat.com/bhubbard/nature/default/radosgw/dynamicresharding/?utm_source=chatgpt.com "RGW Dynamic Bucket Index Resharding"))

### 3) Мгновенный ресхард «в лоб»

Если надо прям сейчас и точечно:

```bash
radosgw-admin bucket reshard --bucket <bkt> --num-shards 64 --yes-i-really-mean-it
```

Создаст новый набор индекс-объектов, размажет записи и переключит бакет на новый инстанс. Тоже выполняй в окно обслуживания. ([IBM](https://www.ibm.com/docs/en/storage-ceph/7.1.0?topic=resharding-bucket-index-manually&utm_source=chatgpt.com "Resharding bucket index manually"))

---

## Особые случаи

- **Multisite.** На версиях до Reef динамический ресхардинг в мультисайте был ограничен; с Reef ситуация стала лучше. Если у тебя multisite, планируй операции и обязательно тестируй. ([croit](https://www.croit.io/blog/how-to-optimize-ceph-rgw-with-dynamic-bucket-re-sharding?utm_source=chatgpt.com "How to Optimize Ceph RGW with Dynamic Bucket Re-sharding"))
- **Версионированные бакеты (OLH-мусор).** На Reef есть утилита для чистки лишних OLH-записей, мешающих листингу/lifecycle:
```bash
    radosgw-admin bucket check olh --bucket <bkt> --fix
```
    
    Используй после больших миграций/переливов. ([Ceph](https://ceph.io/en/news/blog/2023/v18-2-1-reef-released/?utm_source=chatgpt.com "v18.2.1 Reef released"))
    

---

## Контроль после ресхарда

```bash
# Проверяем, что число шардов выросло, а бакет «здоров»
radosgw-admin bucket stats --bucket <bkt> | jq '{bucket:.bucket, shards:.num_shards}'
# Следим за RGW и Ceph
ceph -s
journalctl -u ceph-\* | rg rgw   # или путь к логам RGW
```

---

## Рекомендации по планированию

- Ставь **реалистичный** `rgw_max_objs_per_shard` под свою среднюю/пиковую нагрузку.
- Индексы/метаданные RGW держи на **быстрых носителях** (NVMe).
- Для бакетов с миллионами ключей — **префиксируй** пути (размазывай ключи).
- Всегда делай ресхард в окно и наблюдай метрики (latency, 4xx/5xx, queue). ([redhat.com](https://www.redhat.com/es/blog/ceph-rgw-dynamic-bucket-sharding-performance-investigation-and-guidance?channel=%2Fes%2Fblog%2Fchannel%2Fred-hat-storage&utm_source=chatgpt.com "Ceph RGW dynamic bucket sharding"))

Хочешь — добавлю этот блок в твою страницу **Troubleshooting S3** как готовый раздел «Шардирование/Reshard».
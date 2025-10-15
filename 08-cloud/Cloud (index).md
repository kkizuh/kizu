
- Виды виртуализации
- Основные компоненты (на примере VMware/zVirt) 
- Какую проблему решает вирутализация
- Основные возможности виртуализации

виды гипервизоров 

какие системы виртуализации и к какому виду гипервизоров

что такое вложенная виртуализация почему может работать и почему не может

загрузи образ, удали образ, сделай его приватным (только стажер может использовать при создании ВМ)

накидать статей и по сути проверить что теоретические знания есть
- - Раздели: OpenStack, IAM/Ключи, квоты, образа, сети.
        
    - «Руководство по выживанию в облаке» (частые фейлы: security groups, FIP, volume attach).
[[WAF SolidWall|Общее представление о WAF SolidWall ]]
[[reverse-proxy-compare|Caddy vs Traefik vs Nginx]]
[[cloud-services|Облочные сервисы]]
Вот компактное оглавление для твоей личной методички (Cloud):

## Cloud (index)

- **Модели/база**
    
    - [[cloud-models|IaaS / PaaS / SaaS — разница]]
        
    - [[cloud-core-services|Core: Compute / Storage / Network / IAM / Obs]]
        
    - [[cloud-shared-responsibility|Shared Responsibility — кто за что]]
        
- **IaaS быстро**
    
    - [[iaas-compute|ВМ/образы/ASG — основы]]
        
    - [[iaas-networking|VPC/подсети/маршруты/SG/LB/NAT/FIP]]
        
    - [[iaas-storage|S3/Block/File — когда что]]
        
    - [[iaas-security|IAM/RBAC/KMS/ключи]]
        
    - [[iaas-observability|Логи/метрики/алерты]]
        
    - [[iaas-cost|ФинОпс: теги/бюджеты/лимиты]]
        
- **PaaS / Managed**
    
    - [[paas-databases|Managed DB (Postgres/MySQL/Redis)]]
        
    - [[paas-containers|Managed containers/registry]]
        
    - [[paas-functions|FaaS: когда уместно]]
        
    - [[paas-messaging|SQS/Rabbit/Kafka-as-a-service]]
        
    - [[paas-edge|LB/CDN/DNS/WAF/Certs]]
        
- **Практика/выживание**
    
    - [[cloud-survival-guide|Частые фейлы: SG/FIP/DNS/quotas]]
        
    - [[cloud-troubleshoot|Траблшут: сеть/доступ/диски/лимиты]]
        
    - [[cloud-migration|Миграции: данные/образы/cutover]]
        
    - [[cloud-multicloud|Мульти/гибрид — минимально больно]]
        
- **OpenStack (on-prem)**
    
    - [[openstack-cli|CLI: source/list/show/migrate]]
- **Инструменты вокруг**
    - [[cloud-terraform|Terraform: модули, backend, tfsec/tflint]]
    - [[cloud-ansible|Ansible в облаке: инвентори/секреты]]
    - [[cloud-ci-cd|CI/CD для облака: IaC/образы/канарейки]]
- **Сеть/прокси/WAF**
    - [[reverse-proxy-compare|Caddy vs Traefik vs Nginx]]
    - [[WAF SolidWall|WAF SolidWall — базово]]
    - [[cloud-services|Каталог облачных сервисов]]

> Хватит на старт. Если времени мало — сначала заполни: **cloud-survival-guide**, **iaas-networking**, **iaas-security**, **cloud-terraform**, **openstack-cli-cheatsheet**.
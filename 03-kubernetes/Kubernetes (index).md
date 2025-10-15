# Kubernetes (index)

Мини-оглавление — только то, что пригодится быстро вспомнить.

## 1) Самое нужное

- [[k8s-cheatsheet|kubectl: get/describe/logs/exec/rollout]] — один лист команд.
- [[k8s-mini-starters|Шаблон: Deployment + Service + Ingress(+canary)]] — копипаст и в бой.
- [[k8s-rollout|Rollout/Rollback за 5 минут]] — статус, история, undo.

## 2) Шаблоны манифестов

- [[k8s-deploy-service-ingress|Deployment/Service/Ingress(+canary)]]
- [[k8s-hpa|HPA — авто-скейлинг по CPU/RAM/метрикам]]
- [[k8s-pdb|PDB — «сколько можно уронить»]]
- [[k8s-podsecurity|PodSecurity — baseline/restricted]]
- [[k8s-networkpolicy|NetworkPolicy — deny-all + allow-list]]

## 3) Helm (добавлено)

- [[k8s-helm|Helm: install/upgrade/rollback, values, templating]]
- [[k8s-helm-starters|Готовые чарты: nginx, postgres, ingress-nginx, cert-manager]]    

## 4) Архитектура коротко

- [[k8s-architecture|Control plane: apiserver/etcd/scheduler/controllers; Node: kubelet/kube-proxy]]
- [[k8s-what|Что такое K8s и зачем]] — на один экран.

## 5) Сеть и доступ

- [[k8s-networking|Service (ClusterIP/NodePort/LB) + Ingress — как экспозить]]
- [[k8s-cni|CNI: Calico/Flannel/Cilium — когда что]]

## 6) Хранилище

- [[k8s-storage|PVC/StorageClass, AccessModes (RWO/RWX), snapshot/backup]]

## 7) Траблшут за 10 минут

- [[k8s-troubleshoot|ImagePullBackOff/CrashLoop/Pending — чек-лист]]
- [[k8s-events-logs|Events/describe/logs — где болит]]

## 8) Кластеры и локалка

- [[k8s-kubeadm|kubeadm: ваниль/HA, токены, join]]
- [[k8s-k3s|k3s — лёгкий прод/dev]]
- [[k8s-k3d|k3d/kind — кластер в Docker для dev/CI]]
- [[k8s-cluster-api|Cluster API (CAPI): clusterctl, контроллеры]]

---

## Быстрые ссылки (читать потом)

- HA режимы/базовые концепты: kubernetes.io/docs/concepts/architecture/
- «The Hard Way» (etcd/контрольная плоскость): kelseyhightower/kubernetes-the-hard-way
- Что такое K8s человеческим языком: bytebytego guide
- Cluster API концепции: cluster-api.sigs.k8s.io/user/concepts
- Шпаргалка по Deployments/StatefulSets/DaemonSets: статья на Medium
- kubeadm скрипты: techiescamp/kubeadm-scripts
- Nixys: обзор по HA и практике: https://habr.com/ru/companies/nixys/articles/658985/

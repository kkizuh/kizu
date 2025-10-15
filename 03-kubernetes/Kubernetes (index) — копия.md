	- Шаблоны манифестов: Deployment+Service+Ingress(+canary), HPA, PDB, PodSecurity, NetworkPolicy.
- «Rollout/rollback за 5 минут» + команды `kubectl` на память.
- claster api
- k3s
- k8s
- k3d
- Кристиан, [13.09.2025 14:50]
https://habr.com/ru/companies/nixys/articles/658985/

Кристиан, [13.09.2025 14:50]
режим ha режимы кубера
https://cluster-api.sigs.k8s.io/user/concepts

Кристиан, [13.09.2025 14:55]
https://bytebytego.com/guides/what-is-k8s-kubernetes/

Кристиан, [13.09.2025 18:05]
https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md

Кристиан, [13.09.2025 18:06]
controller kamaji

Кристиан, [13.09.2025 18:06]
capi controller manager

Кристиан, [13.09.2025 18:10]
https://kubernetes.io/docs/concepts/architecture/

Кристиан, [13.09.2025 18:10]
https://cluster-api.sigs.k8s.io/
kube-proxy
Ingress
kubelet
StatefulSet
bitnami/postgresql - Docker Image
fluentd - Da
helm
kind:
- DaemonSet
- Service 
- Deployment
https://medium.com/@muppedaanvesh/a-hands-on-guide-to-kubernetes-deployments-statefulsets-and-daemonsets-%EF%B8%8F-20167634775d

Кристиан, [04.09.2025 2:22]
Автомасштабируем узлы кластера Kubernetes. Часть 1 / Хабр https://share.google/h8XReq47HH8LxU1M8

Кристиан, [05.09.2025 9:46]
кластер K8S состоит из, барабанная дробь, рабочих узлов. В узлах или нодах (Nodes, Worker nodes), помимо контейнеров компонентов самого кластера, размещаются контейнеры наших проектов и сервисов.

Worker nodes состоит из компонентов:

kubelet - сервис или агент, который контролирует запуск компонентов (контейнеров) кластера

kube-proxy - конфигурирует правила сети на узлах

Кристиан, [05.09.2025 9:46]
Плоскость управления (Master nodes) управляет рабочими узлами и подами в кластере. Там располагаются компоненты, которые управляют узлами кластера и предоставляют доступ к API.

Control plane состоит из компонентов:

kube-apiserver - предоставляет API кубера

etcd - распределенное key-value хранилище для всех данных кластера. Необязательно располагается внутри мастера, может стоять как отдельный кластер

kube-scheduler - планирует размещение подов на узлах кластера

kube-controller-manager - запускает контроллеры

kubelet - сервис или агент, который контролирует запуск основных компонентов (контейнеров) кластер

Кристиан, [05.09.2025 13:38]
kubectl get --raw='/readyz?verbose'

Кристиан, [05.09.2025 13:41]
root@vm-k8s:/home/ubuntu/kubeadm-scripts/scripts# kubeadm token create --print-join-command
kubeadm join 176.119.3.42:6443 --token x2j0l8.uq1y36m04ipounvx --discovery-token-ca-cert-hash sha256:fdef89ef5faa4a30e07d318dc815fab1fb44f604c09a46faa20661795ee3c3b6

Кристиан, [05.09.2025 13:43]
kubectl label node vm-k8s-worker node-role.kubernetes.io/worker=worker
node/vm-k8s-worker labeled
root@vm-k8s:/home/ubuntu/kubeadm-scripts/scripts# kubectl get node
NAME            STATUS   ROLES           AGE     VERSION
vm-k8s          Ready    control-plane   9m45s   v1.30.0
vm-k8s-worker   Ready    worker          3m41s   v1.30.0

Кристиан, [05.09.2025 13:43]
https://github.com/techiescamp/kubeadm-scripts
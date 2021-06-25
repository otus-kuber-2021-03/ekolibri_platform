# ekolibri_platform
ekolibri Platform repository

Задание №5 TEMPLATES
1. Установлены ingress-nginx, cert-manager, chartmuseum
2. Как пользоваться chartmuseum:
helm repo add chartmuseum https://chartmuseum.104.198.124.28.nip.io
helm repo update and it will fetch chart metadata from ChartMuseum as well
3. harbor:
kubectl create ns harbor
helm upgrade --install harbor harbor/harbor --wait --namespace=harbor --version=1.1.2 -f ekolibri_platform/kubernetes-templating/harbor/values.yaml
helm upgrade --install harbor harbor/harbor --wait --namespace=harbor --version=1.1.2 -f ekolibri_platform/kubernetes-templating/harbor/values.yaml
4. Создан helmfile
5. Создан hisper-shop chart
frontend и redis вынесены в засимости. Доступен по адресу https://hipster-shop.104.198.124.28.sslip.io
6. Создан kubecfg шаблон. В указанном в задании libsonnet старые версии api, поэтому они были заменены на другой оьсюда https://github.com/bitnami-labs/kube-libsonnet/blob/master/kube.libsonnet
7. Созданы kustomize шаблоны для сервиса recommendationservice

Задание №4 VOLUMES

1. Создан secret verysecret, куда помещены значения переменных в base64
2. Запущен statefulset

MacBook-Elena:~ elenakolibri$ kubectl describe statefulset minio
Name:               minio
Namespace:          default
CreationTimestamp:  Fri, 23 Apr 2021 23:56:18 +0300
Selector:           app=minio
Labels:             <none>
Annotations:        <none>
Replicas:           1 desired | 1 total
Update Strategy:    RollingUpdate
  Partition:        0
Pods Status:        0 Running / 1 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=minio
  Containers:
   minio:
    Image:      minio/minio:RELEASE.2019-07-10T00-34-56Z
    Port:       9000/TCP
    Host Port:  0/TCP
    Args:
      server
      /data
    Liveness:  http-get http://:9000/minio/health/live delay=120s timeout=1s period=20s #success=1 #failure=3
    Environment Variables from:
      verysecret  Secret  Optional: false
    Environment:  <none>

------------------------------------------------------------------------------------
------------------------------------------------------------------------------------
Задание №2
1. при запуске пода возникает ошибка:
MacBook-Elena:ekolibri_platform elenakolibri$ kubectl apply -f kubernetes-controllers/frontend-replicaset.yaml 
error: error validating "kubernetes-controllers/frontend-replicaset.yaml": error validating data: ValidationError(ReplicaSet.spec): missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec; if you choose to ignore these errors, turn validation off with --validate=false
не указан selector
MacBook-Elena$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
MacBook-Elena$ kubectl get pods -l app=frontend
NAME             READY   STATUS    RESTARTS   AGE
frontend-vp7ss   1/1     Running   0          2m6sfrontend   1         0         0       10s

2. Почему при изменении образа в файле  frontend-replicaset.yaml  и apply -f из него ничего не происходит, а версия меняется только после удаления подов?
Потому что ReplicaSet управляет количеством запущенных подов, и не умеет пересоздавать их налету  при изменении конфигурации, это умеет Deployment.  При удалении подов ReplicaSet видит что их нет, а надо 3, считывает конфиг и запускает уже поды с новой версией.  

3. Создан Deployment 
MacBook-Elena:~ elenakolibri$ kubectl apply -f ekolibri_platform/kubernetes-controllers/paymentservice-deployment.yaml | kubectl get pods -l app=payment -w
NAME                       READY   STATUS    RESTARTS   AGE
payment-86766bc788-fvzsz   1/1     Running   0          3m59s
payment-86766bc788-tv5gr   1/1     Running   0          3m59s
payment-86766bc788-xdvsv   1/1     Running   0          3m59s
payment-7c8f7fc785-f6sss   0/1     Pending   0          0s
payment-7c8f7fc785-f6sss   0/1     Pending   0          0s
payment-7c8f7fc785-f6sss   0/1     ContainerCreating   0          0s
payment-7c8f7fc785-f6sss   1/1     Running             0          9s
payment-86766bc788-xdvsv   1/1     Terminating         0          4m19s
payment-7c8f7fc785-w65jw   0/1     Pending             0          7s
payment-7c8f7fc785-w65jw   0/1     Pending             0          19s
payment-7c8f7fc785-w65jw   0/1     ContainerCreating   0          21s
payment-86766bc788-xdvsv   0/1     Terminating         0          4m59s
payment-7c8f7fc785-w65jw   1/1     Running             0          47s
payment-86766bc788-xdvsv   0/1     Terminating         0          5m39s
payment-86766bc788-xdvsv   0/1     Terminating         0          5m39s
payment-86766bc788-tv5gr   1/1     Terminating         0          6m15s
payment-7c8f7fc785-8lfwz   0/1     Pending             0          5s
payment-7c8f7fc785-8lfwz   0/1     Pending             0          8s
payment-7c8f7fc785-8lfwz   0/1     ContainerCreating   0          11s
payment-7c8f7fc785-8lfwz   0/1     ErrImagePull        0          31s
payment-86766bc788-tv5gr   0/1     Terminating         0          6m48s
payment-7c8f7fc785-8lfwz   0/1     ImagePullBackOff    0          45s
payment-7c8f7fc785-8lfwz   1/1     Running             0          49s
payment-86766bc788-fvzsz   1/1     Terminating         0          7m10s

Пересозданы с новым образом:
acBook-Elena:~ elenakolibri$ kubectl get pods -l app=payment -o jsonpath="{.items[0:3].spec.containers[0].image}"
polovina/paymentservice:v0.0.2 polovina/paymentservice:v0.0.2 polovina/paymentservice:v0.0.2

Присутствуют две RS
MacBook-Elena:~ elenakolibri$ kubectl get rs
NAME                 DESIRED   CURRENT   READY   AGE
payment-7c8f7fc785   3         3         3       9m37s
payment-86766bc788   0         0         0       13m

Откат прошел до первой версии:
MacBook-Elena:~ elenakolibri$ kubectl rollout undo deployment payment --to-revision=1 | kubectl get rs -l app=payment -w
NAME                 DESIRED   CURRENT   READY   AGE
payment-7c8f7fc785   3         3         3       11m
payment-86766bc788   0         0         0       15m
payment-86766bc788   0         0         0       16m
payment-86766bc788   1         0         0       16m
payment-86766bc788   1         1         1       17m
payment-7c8f7fc785   2         3         3       13m
payment-86766bc788   2         1         1       17m
payment-7c8f7fc785   2         3         3       13m
payment-7c8f7fc785   2         2         2       13m
payment-86766bc788   2         1         1       17m
payment-86766bc788   2         2         1       17m
payment-86766bc788   2         2         2       17m
payment-7c8f7fc785   1         2         2       14m
payment-86766bc788   3         2         2       18m
payment-7c8f7fc785   1         2         2       14m
payment-7c8f7fc785   1         1         1       14m
payment-86766bc788   3         2         2       18m
payment-86766bc788   3         3         2       18m
payment-86766bc788   3         3         3       18m
payment-7c8f7fc785   0         1         1       14m
payment-7c8f7fc785   0         1         1       15m
payment-7c8f7fc785   0         0         0       15m

MacBook-Elena:~ elenakolibri$ kubectl get pods -l app=payment -o jsonpath="{.items[0:3].spec.containers[0].image}"
polovina/paymentservice:v0.0.1 polovina/paymentservice:v0.0.1 polovina/paymentservice:v0.0.1


4. Создан деплоймент paymentservice-deployment-bg.yaml
Здесь при выставлении maxSurge: 100% поднимаются три ноды с новой версией, однако старые не терминируются. Поэтому необходимо  поставить деплой на паузу, и сделать, например, scale:
MacBook-Elena:~ elenakolibrikubectl rollout pause deployment paymentservice
deployment.apps/paymentservice paused
MacBook-Elena:~ elenakolibri$ kubectl scale deployment.apps/paymentservice --replicas=3 
deployment.apps/paymentservice scaled
MacBook-Elena:~ elenakolibri$ kubectl get pods -l app=paymentservice
NAME                             READY   STATUS    RESTARTS   AGE
paymentservice-8799cb596-h6lll   1/1     Running   0          3m1s
paymentservice-8799cb596-rvtf6   1/1     Running   0          3m1s
paymentservice-8799cb596-sdkzj   1/1     Running   0          3m1s
MacBook-Elena:~ elenakolibri$ kubectl get pods -l app=paymentservice -o=jsonpath="{.items[0:3].spec.containers[0].image}"
polovina/paymentservice:v0.0.2 polovina/paymentservice:v0.0.2 polovina/paymentservice:v0.0.2

Еще как вариант, можно после постановки на паузу изменить конфиг деплоя, например убрать maxSurge и maxUnavailable тогда не отвечающие версии образа ноды тоже затерминируются. 

5. Создан деплоймент paymentservice-deployment-reversr.yaml

6. Создан деплоймент frontend c livenesProbe, при изменении на /_health ошибка:
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  47s               default-scheduler  Successfully assigned default/frontend-97d48946b-r4v6j to kind-worker2
  Normal   Pulling    45s               kubelet            Pulling image "polovina/frontend:v0.0.2"
  Normal   Pulled     32s               kubelet            Successfully pulled image "polovina/frontend:v0.0.2" in 13.879496623s
  Normal   Created    31s               kubelet            Created container frontend
  Normal   Started    30s               kubelet            Started container frontend
  Warning  Unhealthy  9s (x2 over 19s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Warning  Unhealthy  6s (x2 over 16s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404 

7. В node-exporter для запуска на мастер нодах необходимо добавить 
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

----------------------------------------------------------------------------------

----------------------------------------------------------------------------------
Задание 1: 
1. Установлен minikube 
2. Ответ на вопрос почему поды в kube-system перезапускаются после удаления:
MacBook-Elena:~ elenakolibri$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-svblh            1/1     Running   0          166m
etcd-minikube                      1/1     Running   0          166m
kube-apiserver-minikube            1/1     Running   0          166m
kube-controller-manager-minikube   1/1     Running   0          166m
kube-proxy-cc6cx                   1/1     Running   0          166m
kube-scheduler-minikube            1/1     Running   0          166m
 etcd, apiserver, controller-manager и scheduler относятся к статичным подам, которые управляются kubelet напрямую, а не через api. В конфиге kubelet мы видим директорию, где хранятся манифесты статичных подов: 
$ sudo cat /var/lib/kubelet/config.yaml | grep static 
staticPodPath: /etc/kubernetes/manifests

$ sudo ls -la /etc/kubernetes/manifests/
-rw------- 1 root root 2277 Apr 11 10:22 etcd.yaml
-rw------- 1 root root 3585 Apr 11 10:22 kube-apiserver.yaml
-rw------- 1 root root 2895 Apr 11 10:22 kube-controller-manager.yaml
-rw------- 1 root root 1385 Apr 11 10:22 kube-scheduler.yaml
Файлы конфигурации и данных etcd хранятся локально на хосте и монтируются в контейнер, что позволяет при удалении контейнера etcd сохранить данные кластера, что в свою очередь позволяет запустить apiserver, controller-manager и scheduler.
/etc/kubernetes/manifests/etcd.yaml
  volumes:
  - hostPath:
      path: /var/lib/minikube/certs/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/minikube/etcd
      type: DirectoryOrCreate
    name: etcd-data
kube-proxy в minicube является DaemonSet объектом и управляется daemonset контроллером, который обеспечивает наличие пода на каждой ноде
MacBook-Elena:~ elenakolibri$ kubectl get daemonset.apps/kube-proxy -n kube-system -o yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2021-04-11T10:23:27Z"
  generation: 1
  labels:
    k8s-app: kube-proxy
  name: kube-proxy
 
  			.  .  .
status:
  currentNumberScheduled: 1
  desiredNumberScheduled: 1
  numberAvailable: 1
  numberMisscheduled: 0
  numberReady: 1
  observedGeneration: 1
  updatedNumberScheduled: 1

coredns управляется с помощью  ReplicaSet, которое обеспечивает указанное количество реплик пода, в данном случае 1
MacBook-Elena:~ elenakolibri$ kubectl describe rs/coredns-74ff55c5b -n kube-system
Name:           coredns-74ff55c5b
Namespace:      kube-system
Selector:       k8s-app=kube-dns,pod-template-hash=74ff55c5b
Labels:         k8s-app=kube-dns
                pod-template-hash=74ff55c5b
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/coredns
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
##########################################################################
3. Cобран образ nginx из докерфайла (Dockerfile_old)  в который копируется index.html и nginx.conf 
создан web-app.yaml файл, запушен, на localhost:8000 висит страничка, отображающая дату
4. Переделан докерфайл (Dockerfile) в него копируется только конфиг nginx.conf который при обращению к location / перенаправляет на index.html, лежащий в /app
MacBook-Elena:~ elenakolibri$ kubectl apply -f web-pod.yaml && kubectl get pods -w
pod/web-hw1 created
NAME      READY   STATUS     RESTARTS   AGE
web-hw1   0/1     Init:0/1   0          0s
web-hw1   0/1     Init:0/1   0          8s
web-hw1   0/1     PodInitializing   0          9s
web-hw1   1/1     Running           0          12s
По адресу localhost:8000 загружается страничка index.html

5. В последнем задании необходимо было добавить conainerPort и переменные в файл yaml

. 


# ekolibri_platform
ekolibri Platform repository
Проделанная работа:
1. Установлен minikube 

##################################################################################
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
total 16
drwxr-xr-x 2 root root  120 Apr 11 10:22 .
drwxr-xr-x 4 root root  160 Apr 11 10:22 ..
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

##########################################################################
4. Переделан докерфайл (Dockerfile) в него копируется только конфиг nginx.conf который при обращению к location / перенаправляет на index.html, лежащий в /app
Dockerfile:
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf 
EXPOSE 8000
ENTRYPOINT ["nginx", "-g", "daemon off;"]

web-pod.yaml:
apiVersion: v1
kind: Pod
metadata:
  name: web-hw1
  labels:
    app: web
spec:
  containers:
    - name: web-hw1-v1
      image: polovina/hw1:latest
      volumeMounts:
      - name: app
        mountPath: "/app"
      ports:
        - containerPort: 8000
  initContainers:
    - name: init-web
      image: busybox:latest
      command: ['sh', '-c', 'wget -O-  https://tinyurl.com/otus-k8s-intro | sh']
      volumeMounts:
      - name: app
        mountPath: "/app"
  volumes:
  - name: app
    emptyDir: {}


MacBook-Elena:~ elenakolibri$ kubectl apply -f web-pod.yaml && kubectl get pods -w
pod/web-hw1 created
NAME      READY   STATUS     RESTARTS   AGE
web-hw1   0/1     Init:0/1   0          0s
web-hw1   0/1     Init:0/1   0          8s
web-hw1   0/1     PodInitializing   0          9s
web-hw1   1/1     Running           0          12s
^CMacBook-Elena:~ elenakolibrikubectl port-forward --address 0.0.0.0 pod/web-hw1 8000:8000
Forwarding from 0.0.0.0:8000 -> 8000
Handling connection for 8000
Handling connection for 8000

По адресу localhost:8000 загружается страничка index.html



# Домашнее задание: Сетевое взаимодействие в Kubernetes

## Задание 1: Настройка Service (ClusterIP и NodePort)

### Манифесты

**deploy_task1.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: web
          image: nginx:latest
          ports:
            - containerPort: 80
        - name: tool
          image: wbitt/network-multitool:latest
          ports:
            - containerPort: 8080
          env:
            - name: HTTP_PORT
              value: "8080"
```

**serv_cluster_task1.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-cluster
spec:
  selector:
    app: my-app
  ports:
    - name: web
      protocol: TCP
      port: 9001
      targetPort: 80
    - name: tool
      protocol: TCP
      port: 9002
      targetPort: 8080
  type: ClusterIP
```

**serv_node_task1.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-node
spec:
  selector:
    app: my-app
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

### Порядок выполнения

```bash
kubectl apply -f deploy_task1.yaml
kubectl apply -f serv_cluster_task1.yaml
kubectl apply -f serv_node_task1.yaml
```

### Проверка изнутри кластера (ClusterIP)

```bash
kubectl run test-pod --image=wbitt/network-multitool --rm -it -- sh
curl service-cluster:9001   # nginx
curl service-cluster:9002   # multitool
```

**Результат:**
- `curl service-cluster:9001` → страница "Welcome to nginx!"
- `curl service-cluster:9002` → `WBITT Network MultiTool (with NGINX) - <pod-name> - <pod-ip> - HTTP: 8080, HTTPS: 443.`

📸 *[Скриншот: вывод обеих curl-команд внутри test-pod]*

### Проверка снаружи кластера (NodePort)

```bash
kubectl get svc service-node
# NAME           TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
# service-node   NodePort   10.152.183.198   <none>        80:32743/TCP   ...

curl http://<node-ip>:32743
```

**Результат:** `200 OK`, страница "Welcome to nginx!" — доступ через NodePort подтверждён.

📸 *[Скриншот: curl к NodePort]*

---

## Задание 2: Настройка Ingress

### Включение Ingress-контроллера

```bash
microk8s enable ingress
```
Ingress-контроллер в MicroK8s — **Traefik** (`traefik.io/ingress-controller`), доступен через `IngressClass` с именем `public`.

### Манифесты

**deploy_frontend_task2.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app-frontend
  template:
    metadata:
      labels:
        app: my-app-frontend
    spec:
      containers:
        - name: frontend
          image: nginx:latest
          ports:
            - containerPort: 80
```

**deploy_backend_task2.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app-backend
  template:
    metadata:
      labels:
        app: my-app-backend
    spec:
      containers:
        - name: backend
          image: wbitt/network-multitool:latest
          ports:
            - containerPort: 80
```

**serv_frontend_task2.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-frontend
spec:
  selector:
    app: my-app-frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

**serv_backend_task2.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-backend
spec:
  selector:
    app: my-app-backend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

**ingress_task2.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: service-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-strip-api-prefix@kubernetescrd
spec:
  ingressClassName: public
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: service-backend
                port:
                  number: 80
```

**middleware.yaml** (Traefik Middleware — убирает префикс `/api` перед проксированием на backend, чтобы контейнер `multitool` отдавал свою страницу, а не `404`)
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-api-prefix
spec:
  stripPrefix:
    prefixes:
      - /api
```

> Примечание: в шаблоне задания указывалась аннотация `nginx.ingress.kubernetes.io/rewrite-target: /`, предназначенная для контроллера **ingress-nginx**. Поскольку в MicroK8s Ingress-контроллером по умолчанию является **Traefik**, эта аннотация не применяется, и вместо неё используется нативный для Traefik механизм — `Middleware` типа `stripPrefix`, подключаемый через аннотацию `traefik.ingress.kubernetes.io/router.middlewares`.

### Порядок выполнения

```bash
kubectl apply -f deploy_frontend_task2.yaml
kubectl apply -f deploy_backend_task2.yaml
kubectl apply -f serv_frontend_task2.yaml
kubectl apply -f serv_backend_task2.yaml
kubectl apply -f middleware.yaml
kubectl apply -f ingress_task2.yaml
```

### Проверка состояния

```bash
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
```

```
NAME              CLASS    HOSTS   ADDRESS   PORTS   AGE
service-ingress   public   *                 80      ...
```

### Проверка доступности

```bash
curl -v http://<node-ip>/
```
**Результат:** `200 OK` — страница "Welcome to nginx!" (frontend).

📸 *[Скриншот: curl http://<node-ip>/]*

```bash
curl -vL http://<node-ip>/api
```
**Результат:** редирект `/api` → `/api/` (стандартное поведение nginx внутри `network-multitool` для путей без завершающего слэша), далее `200 OK` — страница multitool (backend).

📸 *[Скриншот: curl -vL http://<node-ip>/api]*

---

## Использованное окружение

- ОС: Ubuntu 24.04
- Kubernetes: MicroK8s (аддоны: `dns`, `helm3`, `ingress`, `dashboard`)
- Ingress-контроллер: Traefik v3.6.2 (`IngressClass: public`)
- CNI: Calico

## Список файлов в репозитории

- `deploy_task1.yaml`
- `serv_cluster_task1.yaml`
- `serv_node_task1.yaml`
- `deploy_frontend_task2.yaml`
- `deploy_backend_task2.yaml`
- `serv_frontend_task2.yaml`
- `serv_backend_task2.yaml`
- `middleware.yaml`
- `ingress_task2.yaml`
- `README.md`

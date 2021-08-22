[ReportPortal](https://reportportal.io/) – это веб-решение на базе открытого ПО, созданное разработчиками EPAM и OSS-сообщества. Его использование позволяет собрать в одном месте документы и результаты различных проектов по тестированию, которые выполняются в компании, и сделать их доступными для тестировщиков, ИТ-специалистов и бизнес-заказчиков.

Благодаря применению Report Portal становится возможным:

- Ускорить запуск продуктов в эксплуатацию совместно с автоматизацией тестирования
- Просматривать тестовые сценарии со всеми связанными данными в одном решении «здесь и сейчас», с логами, скриншотами, двоичными данными
- Связывать определенные тестовые сценарии с найденными ошибками (багами), проблемами автоматизации или проблемами системы

В этом посте будет описана установка [ReportPortal](https://reportportal.io/) в kubernetes.

### Требования:

- Kubernetes
- 3 Worker ноды kubernetes с 10ГБ ОЗУ.



### Установка Kubernetes

Базовая инструкция установки Reportportal в kubernets - https://github.com/reportportal/kubernetes/tree/master/reportportal#cloud-computing-services-platform-installation

Устанавливать будем в Managed by kubernetes в Yandex Cloud.

После того как у вас будет готов kubernetes, получаем список нод k8s.

**kubectl get nodes**

```
NAME                        STATUS   ROLES    AGE   VERSION
cl1m6caqfdut6pkfolel-elyj   Ready    <none>   22m   v1.18.18
cl1m6caqfdut6pkfolel-iwyf   Ready    <none>   22m   v1.18.18
cl1m6caqfdut6pkfolel-uben   Ready    <none>   22m   v1.18.18
```

#### Устанавливаем Kubernetes Labels на ноды

```
kubectl label nodes <NODE-1> service=api
kubectl label nodes <NODE-2> service=rabbitmq
kubectl label nodes <NODE-3> service=db
```

Меняем `<NODE-X>` на имя вашей ноды.

```
kubectl label nodes cl1m6caqfdut6pkfolel-elyj service=api
kubectl label nodes cl1m6caqfdut6pkfolel-iwyf service=rabbitmq
kubectl label nodes cl1m6caqfdut6pkfolel-uben service=db
```

#### Устанавливаем ingress-nginx

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx
```

Скачиваем репозиторий https://github.com/reportportal/kubernetes

```
git clone https://github.com/reportportal/kubernetes
cd kubernetes
```

Получаем внешний IP от ingress-ingress

```
kubectl get svc nginx-ingress-ingress-nginx-controller
NAME                                     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller   LoadBalancer   10.96.224.63   130.193.35.107   80:30958/TCP,443:31717/TCP   39s
```



### Установка Elasticsearch

Добавляем helm репозиторий Elasticsearch

```
helm repo add elastic https://helm.elastic.co && helm repo update
```

Скачиваем helm чарты

```
helm dependency build ./reportportal/
```

Строка установки Elasticsearch

```
helm install <es_chart_name> ./reportportal/charts/elasticsearch-7.6.1.tgz
```

Вместо `<es_chart_name>` придумайте название helm чарта. Пусть будет `elasticsearch`.

Устанавливаем Elasticsearch

```
helm install elasticsearch ./reportportal/charts/elasticsearch-7.6.1.tgz
```



### Установка RabbitMQ

Строка установки RabbitMQ 

```
helm install <rabbitmq_chart_name> --set auth.username=rabbitmq,auth.password=<rmq_password>,replicaCount=1 ./reportportal/charts/rabbitmq-7.5.6.tgz
```

Вместо `<rabbitmq_chart_name>` придумайте название helm чарта. Пусть будет `rabbitmq`.

Вместо `<rmq_password>` придумайте пароль от RabbitMQ. Пусть будет `password`.

```
helm install rabbitmq --set auth.username=rabbitmq,auth.password=password,replicaCount=1 ./reportportal/charts/rabbitmq-7.5.6.tgz
```

Дожидаемся что под перейдет в статус running

```
kubectl get pod rabbitmq-0
NAME         READY   STATUS    RESTARTS   AGE
rabbitmq-0   0/1     Running   0          60s
```

Настраиваем RabbitMQ memory threshold

```
kubectl exec -it rabbitmq-0 -- rabbitmqctl set_vm_memory_high_watermark 0.8
```

Отредактируем reportportal/values.yaml

```
rabbitmq:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: <rabbitmq_chart_name>.default.svc.cluster.local
    port: 5672
    user: rabbitmq
    apiport: 15672
    apiuser: rabbitmq
```

Вместо `<rabbitmq_chart_name>` меняем название helm чарта, которое вы придумали. У меня это `rabbitmq`.

### Установка PostgreSQL

Строка установки PostgreSQL

```
helm install <postgresql_chart_name> --set postgresqlUsername=rpuser,postgresqlPassword=<rpuser_password>,postgresqlDatabase=reportportal,postgresqlPostgresPassword=<postgres_password> -f ./reportportal/postgresql/values.yaml ./reportportal/charts/postgresql-8.6.2.tgz
```

Вместо `<postgresql_chart_name>` придумайте название helm чарта. Пусть будет `postgresql`.

Вместо `<rpuser_password>` придумайте пароль для rpuser. Пусть будет `password`.

Вместо `<postgres_password>` придумайте пароль для postgres. Пусть будет `password`.

```
helm install postgresql --set postgresqlUsername=rpuser,postgresqlPassword=password,postgresqlDatabase=reportportal,postgresqlPostgresPassword=password -f ./reportportal/postgresql/values.yaml ./reportportal/charts/postgresql-8.6.2.tgz
```

Отредактируем reportportal/values.yaml

```
postgresql:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: <postgresql_chart_name>.default.svc.cluster.local
    port: 5432
    user: rpuser
    dbName: reportportal
    password:
```

Вместо `<postgresql_chart_name>` меняем название helm чарта, которое вы придумали. У меня это `postgresql`.

### Установка MinIO 

Строка установки MinIO 

```
helm install minio --set accessKey=<your_minio_accesskey>,secretKey=<your_minio_secretkey>,persistence.size=40Gi stable/minio
```

Вместо `<your_minio_accesskey>` придумайте accesskey. Пусть будет `accesskey`.

Вместо `<your_minio_secretkey>` придумайте secretkey. Пусть будет `secretkey`.

```
helm install minio --set accessKey=accesskey,secretKey=secretkey,persistence.size=40Gi stable/minio
```

Отредактируем reportportal/values.yaml

```
minio:
  enabled: true
  installdep:
    enable: false
  endpoint: http://<minio-release-name>.default.svc.cluster.local:9000
  region:
  accesskey: <minio-accesskey>
  secretkey: <minio-secretkey>
```

Вместо `<minio-release-name>` меняем название helm чарта. Это `minio`.

Вместо `<minio-accesskey>` меняем accesskey, которое вы придумали. У меня это `accesskey`.

Вместо `<minio-secretkey>` меняем secretkey, которое вы придумали. У меня это `secretkey`.


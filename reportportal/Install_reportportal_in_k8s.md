[ReportPortal](https://reportportal.io/) – это веб-решение на базе открытого ПО, созданное разработчиками EPAM и OSS-сообщества. Его использование позволяет собрать в одном месте документы и результаты различных проектов по тестированию, которые выполняются в компании, и сделать их доступными для тестировщиков, ИТ-специалистов и бизнес-заказчиков.

Благодаря применению ReportPortal становится возможным:

- Ускорить запуск продуктов в эксплуатацию совместно с автоматизацией тестирования
- Просматривать тестовые сценарии со всеми связанными данными в одном решении «здесь и сейчас», с логами, скриншотами, двоичными данными
- Связывать определенные тестовые сценарии с найденными ошибками (багами), проблемами автоматизации или проблемами системы

В этом посте будет описана установка [ReportPortal](https://reportportal.io/) в kubernetes.

<cut />

### Требования:

- Kubernetes (тестировал на версии 1.18)
- 3 Worker ноды kubernetes с 2 ядрами CPU и 10ГБ ОЗУ.

### Установка Kubernetes

Базовая инструкция установки Reportportal в kubernets - https://github.com/reportportal/kubernetes/tree/master/reportportal#cloud-computing-services-platform-installation

Устанавливать будем в Managed by kubernetes в Yandex Cloud.

#### Получаем список нод k8s

После того как у вас будет готов kubernetes, получаем список нод k8s.

```
kubectl get nodes
NAME                        STATUS   ROLES    AGE     VERSION
cl1mikj7kfic31snr11h-ivow   Ready    <none>   2m13s   v1.18.18
cl1mikj7kfic31snr11h-ubus   Ready    <none>   2m12s   v1.18.18
cl1mikj7kfic31snr11h-ynax   Ready    <none>   95s     v1.18.18
```

#### Устанавливаем Kubernetes Labels на ноды

```
kubectl label nodes <NODE-1> service=api
kubectl label nodes <NODE-2> service=rabbitmq
kubectl label nodes <NODE-3> service=db
```

Меняем `<NODE-X>` на имя вашей ноды.

```
kubectl label nodes cl1mikj7kfic31snr11h-ivow service=api
kubectl label nodes cl1mikj7kfic31snr11h-ubus service=rabbitmq
kubectl label nodes cl1mikj7kfic31snr11h-ynax service=db
```

#### Устанавливаем ingress-nginx

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
helm install --atomic nginx-ingress ingress-nginx/ingress-nginx --version 3.36.0  
```

#### Скачиваем репозиторий reportportal

Скачиваем репозиторий https://github.com/reportportal/kubernetes

```
git clone https://github.com/reportportal/kubernetes
cd kubernetes
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
helm install <elastic-release-name> ./reportportal/charts/elasticsearch-7.6.1.tgz
```

Вместо `<elastic-release-name>` придумайте название helm чарта. Пусть будет `elasticsearch`.

Устанавливаем Elasticsearch. По умолчанию устанавливаются 3 ноды.

```
helm install elasticsearch ./reportportal/charts/elasticsearch-7.6.1.tgz
```

Чтобы установить и использовать 1 master ноду elasticsearch (например, для demо или devel стендов) создайте специальный yaml файл `./reportportal/single-elasticsearch/value.yaml`
```
extraEnvs:
  - name: discovery.type
    value: single-node
  - name: cluster.initial_master_nodes
    value: ""
```

и выполните команду
```
helm install elasticsearch ./reportportal/charts/elasticsearch-7.6.1.tgz --set replicas=1 -f ./reportportal/single-elasticsearch/value.yaml
```

Был сделан [pull request](https://github.com/reportportal/kubernetes/pull/205), добавляющий yaml файл для установки single node elasticsearch


### Установка RabbitMQ

Строка установки RabbitMQ 

```
helm install <rabbitmq-release-name> --set auth.username=rabbitmq,auth.password=<rmq_password>,replicaCount=1 ./reportportal/charts/rabbitmq-7.5.6.tgz
```

Вместо `<rabbitmq-release-name>` придумайте название helm чарта. Пусть будет `rabbitmq`.

Вместо `<rmq_password>` придумайте пароль от RabbitMQ. Пусть будет `password`.

```
helm install rabbitmq --set auth.username=rabbitmq,auth.password=password,replicaCount=1 ./reportportal/charts/rabbitmq-7.5.6.tgz
```

Примечание:
- Если release-name в helm назван "rabbitmq", то адрес будет "rabbitmq.default.svc.cluster.local".
- Если release-name в helm назван "rabbit", то адрес будет "rabbit-rabbitmq.default.svc.cluster.local".

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

Отредактируем `reportportal/values.yaml`

```
rabbitmq:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: <rabbitmq-release-name>.default.svc.cluster.local
    port: 5672
    user: rabbitmq
    apiport: 15672
    apiuser: rabbitmq
```

Вместо `<rabbitmq-release-name>` меняем название helm чарта, которое вы придумали. У меня это `rabbitmq`.

Итоговый фрагмент получается вот таким:

```
rabbitmq:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: rabbitmq.default.svc.cluster.local
    port: 5672
    user: rabbitmq
    apiport: 15672
    apiuser: rabbitmq
```


### Установка PostgreSQL

Строка установки PostgreSQL

```
helm install  <postgresql-release-name> --set postgresqlUsername=rpuser,postgresqlPassword=<rpuser_password>,postgresqlDatabase=reportportal,postgresqlPostgresPassword=<postgres_password> -f ./reportportal/postgresql/values.yaml ./reportportal/charts/postgresql-10.9.4.tgz
```

Вместо `<postgresql-release-name>` придумайте название helm чарта. Пусть будет `postgresql`.

Вместо `<rpuser_password>` придумайте пароль для rpuser. Пусть будет `password`.

Вместо `<postgres_password>` придумайте пароль для postgres. Пусть будет `password`.

```
helm install postgresql --set postgresqlUsername=rpuser,postgresqlPassword=password,postgresqlDatabase=reportportal,postgresqlPostgresPassword=password -f ./reportportal/postgresql/values.yaml ./reportportal/charts/postgresql-10.9.4.tgz
```

Отредактируем `reportportal/values.yaml`

```
postgresql:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: <postgresql-release-name>.default.svc.cluster.local
    port: 5432
    user: rpuser
    dbName: reportportal
    password:
```

Примечание:
- Если release-name в helm назван "postgresql", то address будет "postgresql.default.svc.cluster.local".
- Если release-name в helm назван "postgres", то address будет "postgres-postgresql.default.svc.cluster.local".

Вместо `<postgresql-release-name>` меняем название helm чарта, которое вы придумали. У меня это `postgresql`.

Итоговый фрагмент получается вот таким:

```
postgresql:
  SecretName: ""
  installdep:
    enable: false
  endpoint:
    address: postgresql.default.svc.cluster.local
    port: 5432
    user: rpuser
    dbName: reportportal
    password:
```


### Установка MinIO 

Строка установки MinIO 

```
helm install  <minio-release-name> --set accessKey.password=<your_minio_accesskey>,secretKey.password=<your_minio_secretkey>,persistence.size=5Gi ./reportportal/charts/minio-7.1.9.tgz
```

Вместо `<minio-release-name>` меняем название helm чарта, которое вы придумали. У меня это `minio`.

Вместо `<your_minio_accesskey>` придумайте accesskey. Пусть будет `accesskey`.

Вместо `<your_minio_secretkey>` придумайте secretkey. Пусть будет `secretkey`.

```
helm install minio --set accessKey.password=accesskey,secretKey.password=secretkey,persistence.size=5Gi ./reportportal/charts/minio-7.1.9.tgz
```

Отредактируем `reportportal/values.yaml`

```
minio:
  secretName: ""
  enabled: true
  installdep:
    enable: false
  endpoint: http://<minio-release-name>.default.svc.cluster.local:9000
  endpointshort: <minio-release-name>.default.svc.cluster.local:9000
  region:
  accesskey: <minio-accesskey>
  secretkey: <minio-secretkey>
  bucketPrefix:
  defaultBucketName:
  integrationSaltPath:
```

Вместо `<minio-release-name>` меняем название helm чарта. Это `minio`.

Вместо `<minio-accesskey>` меняем accesskey, которое вы придумали. У меня это `accesskey`.

Вместо `<minio-secretkey>` меняем secretkey, которое вы придумали. У меня это `secretkey`.

Итоговый фрагмент получается вот таким:

```
minio:
  secretName: ""
  enabled: true
  installdep:
    enable: false
  endpoint: http://minio.default.svc.cluster.local:9000
  endpointshort: minio.default.svc.cluster.local:9000
  region:
  accesskey: accesskey
  secretkey: secretkey
  bucketPrefix:
  defaultBucketName:
  integrationSaltPath:
```


#### Настройка Ingress для доступа к reportportal

Отредактируем `reportportal/values.yaml`

```
ingress:
  enable: true
  # IF YOU HAVE SOME DOMAIN NAME SET INGRESS.USEDOMAINNAME to true
  usedomainname: false
  hosts:
    - reportportal.k8.com
```

Необходимо поправить домен `reportportal.k8.com` на ваш домен.

Так как у меня нет домена, но ingress-nginx выставлен наружу, то можно воспользоваться сервисами nip.io, xip.io или sslip.io.

Устанавливаем домен исходя из вашего публичного IP адреса.

Получаем внешний IP от ingress-ingress

```
kubectl get svc nginx-ingress-ingress-nginx-controller
NAME                                     TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller   LoadBalancer   10.96.165.32   178.154.210.20   80:31767/TCP,443:31456/TCP   4m56s
```

Внешний IP адрес - 178.154.210.20.

Получается можно сделать домен reportportal.178.154.210.20.sslip.io.

Итоговый фрагмент получается вот таким:

```
ingress:
  enable: true
  # IF YOU HAVE SOME DOMAIN NAME SET INGRESS.USEDOMAINNAME to true
  usedomainname: false
  hosts:
    - reportportal.178.154.210.20.sslip.io
```

#### Установка helm чарта reportportal

Упаковка helm чарта reportportal

```
helm package ./reportportal/
```

Установка reportportal

```
helm install <reportportal-release-name> --set postgresql.SecretName=<postgresql-release-name>-postgresql,rabbitmq.SecretName=<rabbitmq-release-name>-rabbitmq,minio.secretName=<minio-release-name> ./reportportal-5.5.0.tgz

```

Вместо `<reportportal-release-name>` придумайте название чарта для reportportal. Пусть будет `reportportal`.

Вместо `<postgresql-release-name>` используем название helm чарта postgresql. Это `postgresql`.

Вместо `<rabbitmq-release-name>` используем название helm чарта rabbitmq. Это `rabbitmq`.

```
helm install reportportal --set postgresql.SecretName=postgresql,rabbitmq.SecretName=rabbitmq,minio.secretName=minio ./reportportal-5.5.0.tgz
```

Логины/пароли для доступа в reportportal:
- Default User: default\1q2w3e
- Administrator: superadmin\erebus

Скриншоты:
![](https://habrastorage.org/webt/kz/le/x3/kzlex34taozpoigg18rerdugsls.png)

![](https://habrastorage.org/webt/wr/gt/7x/wrgt7xleqrv0p2fnhvw-ym_1jpi.png)


Известные ошибки:
[Memory cgroup out of memory: Killed process (uwsgi)](https://github.com/reportportal/kubernetes/issues/203)
```
[ 7063.247407] Memory cgroup out of memory: Killed process 63214 (uwsgi) total-vm:3149468kB, anon-rss:164992kB, file-rss:52176kB, shmem-rss:92kB, UID:0 pgtables:1372kB oom_score_adj:986
[ 7063.263325] oom_reaper: reaped process 63214 (uwsgi), now anon-rss:0kB, file-rss:0kB, shmem-rss:92kB
[ 7093.543707] uwsgi invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=986
[ 7093.543711] CPU: 4 PID: 63635 Comm: uwsgi Not tainted 5.4.0-77-generic #86-Ubuntu
...
[ 7093.544021] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[ 7093.544025] [  18904]     0 18904      255        1    32768        0          -998 pause
[ 7093.544028] [  20358]     0 20358    13850     3787    98304        0           986 uwsgi
[ 7093.544030] [  20425]     0 20425    15899     3877   114688        0           986 uwsgi
[ 7093.544033] [  62892]     0 62892   787367    54312  1400832        0           986 uwsgi
[ 7093.544039] [  63384]     0 63384   787367    54314  1400832        0           986 uwsgi
[ 7093.544041] [  63635]     0 63635   328986    37869  1069056        0           986 uwsgi
[ 7093.544046] [  63794]     0 63794   202492    24945   753664        0           986 uwsgi
[ 7093.544048] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=7c29217c3af6fef0b7bcc4150d2502f684ef8f83d2ea377411c9a2265e8d4843,mems_allowed=0,oom_memcg=/kubepods/burstable/pod05347bee-e95c-4857-8435-410a406f1f7a,task_memcg=/kubepods/burstable/pod05347bee-e95c-4857-8435-410a406f1f7a/7c29217c3af6fef0b7bcc4150d2502f684ef8f83d2ea377411c9a2265e8d4843,task=uwsgi,pid=63384,uid=0
[ 7093.544177] Memory cgroup out of memory: Killed process 63384 (uwsgi) total-vm:3149468kB, anon-rss:164988kB, file-rss:52176kB, shmem-rss:92kB, UID:0 pgtables:1368kB oom_score_adj:986
```

Этот issue исправляется pull request - https://github.com/reportportal/kubernetes/pull/208

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
cl1v785b1pfsu02d7o01-eriz   Ready    <none>   48m   v1.18.18
cl1v785b1pfsu02d7o01-ykyk   Ready    <none>   48m   v1.18.18
cl1v785b1pfsu02d7o01-yxiz   Ready    <none>   48m   v1.18.18
```

#### Устанавливаем Kubernetes Labels на ноды

```
kubectl label nodes <NODE-1> service=api
kubectl label nodes <NODE-2> service=rabbitmq
kubectl label nodes <NODE-3> service=db
```

Меняем `<NODE-X>` на имя вашей ноды.

```
kubectl label nodes cl1v785b1pfsu02d7o01-eriz service=api
kubectl label nodes cl1v785b1pfsu02d7o01-ykyk service=rabbitmq
kubectl label nodes cl1v785b1pfsu02d7o01-yxiz service=db
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



### Установка Elasticsearch

Добавляем helm репозиторий Elasticsearch

```
helm repo add elastic https://helm.elastic.co && helm repo update
```

Скачиваем helm чарты

```
helm dependency build ./reportportal/
```

Устанавливаем Elasticsearch

```
helm install elasticsearch ./reportportal/charts/elasticsearch-7.6.1.tgz
```


CI / CD с использованием Knative и Tekton Pipelines

https://docs.ventuscloud.eu/docs/tutorials/tekton-pipelines

**Tekton** - это собственный конвейерный ресурс Kubernetes. Проект Tekton Pipelines предоставляет ресурсы в стиле Kubernetes для объявления конвейеров в стиле CI / CD.

**Knative** - это платформа на основе Kubernetes для развертывания современных бессерверных рабочих нагрузок и управления ими. Нативные компоненты строятся на основе Kubernetes, абстрагируясь от сложных деталей и позволяя разработчикам сосредоточиться на том, что важно.

В этом руководстве мы будем использовать расширение Kubernetes, Knative, Tekton Pipelines, Dashboard и Webhooks для настройки CI? CD с вашим репозиторием GitHub.


### Оглавление

1)   Создание нового кластера Kubernetes

2)   Получение доступа к вашему кластеру с помощью cli

3)   Развертывание расширения Tekton Pipelines, Dashboard и Webhooks

4)   Развертывание Tekton Dashboard

5)   Развертывание расширения Tekton Webhooks

6)   Настройка объектов Tekton

7)   Создание нового веб-перехватчика


### Создание нового кластера Kubernetes

Создайте новый кластер, используя эту статью: [Kubernetes Cluster](https://ventuscloud.eu/docs/kubernetes/kubernetes-cluster)

Используйте следующие параметры для вашего кластера:

- `Master count`: 1
- `Node count`: 1
- `Docker volume size (GB)`: 100
- `Node flavor`: VC-4
- `Master node flavor`: VC-2


### Получение доступа к вашему кластеру с помощью cli

Теперь вы можете получить доступ к своему кластеру с помощью этой статьи: [Доступ к кластеру Kubernetes с помощью CLI.](https://ventuscloud.eu/docs/kubernetes/access-by-cli)
 

### Развертывание расширения Tekton Pipelines, Dashboard и Webhooks

**Примечание:**

Все установки выполняются через cli с использованием kubectl. Kubectl - это интерфейс командной строки для запуска команд в кластерах Kubernetes.

**Совет: установите kubectl**

Для установки kubectl используйте официальные документы kubernetes: https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux

1) Установите последнюю версию Tekton Pipelines (на момент написания этого руководства была последней версией 0.5.2):

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_pipelines_1.png)


2) Убедитесь, что все модули Tekton Pipelines работают:

```
kubectl get pods -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_pipelines_2.png)


### Развертывание Tekton Dashboard 

1) Установите Tekton Dashboard (на момент написания этого руководства была последней версией 0.1.1):

```
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.1/release.yaml
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_dashboard_1.png)

2) Убедитесь, что модуль Tekton Dashboard запущен:

```
kubectl get pods -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_dashboard_2.png)

3) Измените тип службы Tekton Dashboard с «ClusterIP» на «LoadBalancer», чтобы получить внешний IP-адрес и сделать его доступным извне:

```
kubectl patch service tekton-dashboard -n tekton-pipelines -p '{"spec": {"type": "LoadBalancer"}}'
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_dashboard_3.png)

4) Подождите, пока вашему сервису не будет присвоен внешний IP (обычно меньше минуты):

```
kubectl get svc -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_dashboard_4.png)

5) Получите свой внешний IP-адрес и порт и откройте панель управления в браузере:

http://188.40.161.51:9097

 ![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_dashboard_5.png)

### Развертывание расширения Tekton Webhooks

**Важная заметка:**

На момент написания этого руководства расширение Tekton Webhooks было частью «экспериментального» репозитория и находилось в стадии разработки. Имейте в виду, что некоторые шаги могут измениться в будущем. Вот ссылка на инструкции по установке, которые мы рассмотрим дальше: https://github.com/tektoncd/experimental/blob/master/webhooks-extension/README.md

1) Установите необходимые компоненты:

·    Первыми обязательными компонентами являются Tekton Pipelines и Tekton Dashboard, и мы их уже установили.

·    Установите Istio, следуя инструкции https://knative.dev/docs/install/installing-istio/, или вы можете запустить этот скрипт https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_istio.sh вот так: `./install_istio.sh 1.1.7` (рекомендуется Istio версии 1.1.7).

o  Если вы выбрали запуск скрипта, вам нужно сначала установить Helm (https://github.com/helm/helm/releases):

```
wget https://raw.githubusercontent.com/helm/helm/master/scripts/get

chmod +x ./get

./get

helm init
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_helm_1.png)

**Продолжите скрипт:**

```
wget https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_istio.sh

chmod +x ./install_istio.sh

./install_istio.sh 1.1.7
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_istio_1.png)

·    Убедитесь, что необходимые компоненты Istio (нам нужны только два: istio-ingressgateway и istio-pilot) работают:

```
kubectl get pods -n istio-system
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_istio_1.png)

·    Установите Knative Eventing, Eventing Sources & Serving, следуя инструкции https://knative.dev/docs/install/index.html, или вы можете запустить этот скрипт https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/ scripts / install_knative.sh, например: `./install_knative.sh v0.6.0` (настоятельно рекомендуется Knative версии 0.6.0).

Если вы выбрали запуск сценария, выполните следующие действия:

```
wget https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_knative.sh

chmod +x ./install_knative.sh

./install_knative.sh v0.6.0

```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_knative_1.png)

```
kubectl apply -f https://github.com/knative/eventing-sources/releases/download/v0.6.0/eventing-sources.yaml
```

Убедитесь, что необходимые компоненты Knative запущены:

```
kubectl get pods --all-namespaces | grep knative
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_knative_2.png)

2) Выполните следующую команду, чтобы установить расширение Tekton Webhooks и его зависимости:

```
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.1/webhooks-extension_release.yaml -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_webhooks_1.png)

·    Убедитесь, что все модули работают:

```
kubectl get pods -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_webhooks_2.png)

Убедитесь, что раздел «Webhooks» добавлен в вашу панель инструментов Tekton:

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/deploy_webhooks_3.png)

### Настройка объектов Tekton

В этой части руководства мы настроим все необходимые объекты Tekton и репозитории GitHub и Docker.

Начнем с подготовительных действий. Сначала будет репозиторий GitHub:

1) Войдите в свою учетную запись (или создайте новую). 2) Используйте существующий репозиторий, который вы хотите протестировать, или используйте мою версию вилки:

·    Перейдите на https://github.com/TenSt/simple-app (это тоже форк `mchmarny/simple-app`, но с моими изменениями).

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/github_repo_1.png)

·    Нажмите кнопку «Fork» в правом верхнем углу и выберите свою учетную запись в качестве места назначения для создания новой вилки.

·    В результате у вас должен быть собственный репозиторий с уникальным полным именем, например: https://github.com/TenSt/simple-app

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/github_repo_2.png)

Затем нам нужно создать токен, который мы будем использовать для аутентификации из нашего расширения Tekton Webhooks в репозитории GitHub:

1) Нажмите кнопку своего профиля в правом верхнем углу и перейдите в настройки «Settings».

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/github_token_1.png)

2) Перейдите в «Developer settings» в нижней части меню «Personal settings», а затем в «Personal access tokens»

3) Нажмите кнопку «Generate new token».

 ![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/github_token_2.png)

4) Дайте ему имя в поле «Note».

5) Установите только флажок “admin: repo_hook “ - это даст полный доступ к веб-перехватчикам репо (которые мы будем использовать позже).

6) Нажмите кнопку «Generate token» в самом низу.

7) Вы вернетесь на страницу «Personal access tokens» и увидите следующее сообщение: Make sure to copy your new personal access token now. You won’t be able to see it again!”. Оно переводится следующим образом: «Обязательно скопируйте свой новый личный токен доступа сейчас. Вы больше не сможете его увидеть! " Сохраните новый токен где-нибудь, так как GitHub покажет его только один раз (если вы его потеряете, вам нужно будет сгенерировать новый). Мы будем использовать его позже в этом уроке.


Теперь давайте настроим ваш репозиторий Docker (я использую Docker Hub - https://hub.docker.com). 

1) Войдите в свой реестр Docker.

2) Создайте новый репозиторий с тем же именем, что и ваш репозиторий GitHub (например, simple-app).

3) Убедитесь, что он общедоступный (можно использовать частные репозитории Docker, но в этом руководстве это не рассматривается).


Отлично! Мы закончили настройку репозиториев. В результате у вас должно получиться:

·    Репозиторий GitHub с приложением и Dockerfile в корневом каталоге, который мы будем развертывать

·    Токен GitHub с доступом администратора к веб-перехватчикам репо

·    Публичный репозиторий докеров, где мы будем хранить наши изображения

Следующим шагом будет настройка домена для Knative, который мы развернули ранее. Для правильной интеграции с другими сервисами настоятельно рекомендуется настроить свой собственный домен (который доступен из Интернета) в Knative, который будет использовать его для правильной маршрутизации запросов к различным сервисам в вашем кластере Kubernetes.


1) В командной строке выполните следующую команду, чтобы получить внешний IP-адрес службы istio-ingressgateway:

```
kubectl get svc istio-ingressgateway -n istio-system
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/knative_domain_1.png)

2) Настройте запись DNS с подстановочными знаками типа «A», которая указывает на этот ВНЕШНИЙ IP-адрес из предыдущей команды. В моем случае: * .tekton-tutorial.tenst.ml указывает на 188.40.161.33.

3) Создайте новый файл yaml с именем `knative-domain-config.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  # Default domain, provided without selector.
  tekton-tutorial.tenst.ml: |
```


4) Измените `tekton-tutorial.tenst.ml` на свой домен, который вы настроили ранее.

5) Примените его к своему кластеру:

```
kubectl apply -f knative-domain-config.yaml
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/knative_domain_2.png)

Теперь все запросы к * .tekton-tutorial.tenst.ml будут перенаправляться на нашу службу istio-ingressgateway, которая направит их на соответствующие службы в нашем кластере Kubernetes.

Еще одна вещь, которую необходимо настроить, - это источник событий Knative - GitHubSource:

Создайте файл `gitHubSource.yaml` - он создаст Knative GitHubSource, который указывает на репозиторий GitHub:

```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: GitHubSource
metadata:
  name: githubsourcesample
  namespaces: tekton-pipelines
spec:
  eventTypes:
    - push
  ownerAndRepository: TenSt/simple-app
  accessToken:
    secretKeyRef:
      name: tenst-github-token
      key: accessToken
  secretToken:
    secretKeyRef:
      name: tenst-github-token
      key: secretToken
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: webhooks-extension-sink
```

2) Заполните следующие поля:

·    ownerAndRepository - смените его на свой аккаунт / репо

·    accessToken: secretKeyRef: name - измените его на какое-нибудь уникальное имя

·    secretToken: secretKeyRef: name - должно быть таким же, как и предыдущее

3) Примените его к своему кластеру:

```
kubectl apply -f gitHubSource.yaml -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/knative_github_sources_1.png)

Давайте продолжим и настроим создание новых задач, ресурсов и конвейера Tekton:

1) Примените задачу buildah - она будет использоваться для сборки нашего приложения:

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/buildah/buildah.yaml -n tekton-pipelines
```

2) Создайте файл `knctl-role.yaml` - он создаст и привяжет необходимые разрешения для уже существующей учетной записи службы:

 ```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: knctl-deployer
  namespace: tekton-pipelines
rules:
  - apiGroups: ["serving.knative.dev"]
    resources: ["*"]
    verbs: ["get", "list", "create", "update", "delete", "patch", "watch"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "create", "update", "delete", "patch", "watch"]
---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: knctl-deployer-binding
subjects:
- kind: ServiceAccount
  name: tekton-webhooks-extension
  namespace: tekton-pipelines
roleRef:
  kind: ClusterRole
  name: knctl-deployer
  apiGroup: rbac.authorization.k8s.io
 ```

Он состоит из двух частей:

·    Создание ClusterRole в нашем кластере Kubernetes, у которого есть полный доступ ко всем ресурсам в ресурсе «serve.knative.dev» и «пространства имен».

·    Создание привязки ClusterRoleBinding, которая привяжет новую роль к существующей учетной записи tekton-webhooks-extension (под которой будут работать наши конвейеры).

3) Примените его к своему кластеру:

```
kubectl apply -f knctl-role.yaml -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/tekton_tasks_2.png)

4) Создайте файл `knctl-task.yaml` - он создаст новые таксы с именем knctl-deploy, которые будут использоваться для развертывания нашего приложения на Knative в нашем кластере:

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: knctl-deploy
spec:
  inputs:
    params:
    - name: service
      description: Name of the service to deploy
    resources:
    - name: image
      type: image
    - name: git
      type: git
  steps:
  # the first step is required as knctl doesn't support inCluster configuration.
  - name: kubeconfig
    image: gcr.io/cloud-builders/kubectl # it is huge
    command: ["/bin/bash"]
    args:
    - -c
    - mkdir -p /builder/home/.kube; kubectl config view > /builder/home/.kube/config
  - name: cut
    image: golang:latest
    command: ["/bin/bash"]
    args:
    - -c
    - echo ${inputs.resources.git.url} | cut -d "/" -f 4 | tr "[:upper:]" "[:lower:]" > /workspace/account
  - name: namespace
    image: gcr.io/cloud-builders/kubectl # it is huge
    command: ["/bin/bash"]
    args:
    - -c
    - kubectl create ns `cat /workspace/account` --dry-run=true -o yaml | kubectl apply -f -
  - name: rollout
    image: tens/knctl
    command: ["/bin/bash"]
    args:
    - -c
    - knctl deploy --service ${inputs.params.service} --image ${inputs.resources.image.url} --namespace `cat /workspace/account`
 ```

Он состоит из 4 частей:

·    Получите kubeconfig из кластера и вставьте его в `/builder/home/.kube/config` - это необходимо, поскольку knctl не поддерживает конфигурацию inCluster.

·    Получите имя учетной записи из URL-адреса репозитория GitHub

·    Создайте новое пространство имен так же, как GitHub

·    Разверните наше приложение на Knative

5) Примените его к своему кластеру:

```
kubectl apply -f knctl-task.yaml -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/tekton_tasks_3.png)

6) Создайте файл `build-and-deploy-pipeline.yaml` - он создаст новый конвейер с именем `build-and-deploy-pipeline`, который будет использоваться для развертывания нашего приложения на Knative в нашем кластере:

 ```
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-build-and-deploy
spec:
  params:
    - name: repository-name
      type: string
      description: repository name which also will be used as namespace
    - name: image-tag
      type: string
      description: commit id
  resources:
    - name: git-source
      type: git
    - name: docker-image
      type: image
  tasks:
    - name: build-buildah
      taskRef:
        name: buildah
      params:
        - name: DOCKERFILE
          value: ./Dockerfile
      resources:
        inputs:
          - name: source
            resource: git-source
        outputs:
          - name: image
            resource: docker-image
    - name: knctl-deploy
      taskRef:
        name: knctl-deploy
      params:
        - name: service
          value: $(params.repository-name)
      resources:
        inputs:
          - name: image
            resource: docker-image
            from:
              - build-buildah
          - name: git
            resource: git-source
 ```

Он состоит из 2 заданий, которые мы создали ранее:

·    buildah - для создания и отправки изображения с помощью нашего приложения.

·    kntcl-deploy - для развертывания вновь созданного образа на Knative в нашем кластере.

7) Примените его к своему кластеру:

```
kubectl apply -f build-and-deploy-pipeline.yaml -n tekton-pipelines
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/tekton_tasks_4.png)

После того, как мы применили последние шаги, у нас теперь есть:

·    Задачи для сборки и развертывания

·    Роли и разрешения

·    Пайплайн

Последняя, но не менее важная часть конфигурации - это создание секретов для реестра GitHub и Docker - они будут использоваться расширением Webhooks для клонирования вашего репозитория GitHub и отправки нового образа в реестр Docker.

1) Войдите в свою панель управления Tekton и выберите «Secrets» в меню навигации.

2) Нажмите кнопку «Add Secret» и укажите все данные:

·    Name: уникальное имя (например, tenst-github)

·    Namespace: выберите «tekton-pipelines» из выпадающего списка.

·    Access To: Git Server

·    Username: имя пользователя учетной записи GitHub

·    Password/Token: пароль учетной записи GitHub

·    Service Account: выберите «tekton-webhooks-extension»

·    Server URL: оставьте в первом поле «tekton.dev/git-0» и укажите «https://github.com» во втором.

 

3) Нажмите кнопку «Submit».

4) Вы увидите, что был добавлен новый секрет. Теперь нажмите «Add Secret» еще раз, чтобы добавить его в реестр Docker:

·    Name: уникальное имя (например, десятки-докер)

·    Namespace: выберите «tekton-pipelines» из выпадающего списка.

·    Access To: Реестр Docker

·    Username: имя пользователя учетной записи реестра Docker

·    Password/Token: пароль учетной записи реестра Docker

·    Service Account: выберите «tekton-webhooks-extension»

·    Server URL: оставьте в первом поле «tekton.dev/docker-0» и поместите ссылку на свой реестр Docker во втором (используйте «https://index.docker.io/v1/», если вы используете Docker Hub).


5) Нажмите кнопку «Submit».

6) Вы увидите, что был добавлен новый секрет.

 

### Создание нового webhook

Теперь мы подошли к последнему разделу этого руководства - созданию нового веб-перехватчика и наблюдению за тем, как наше приложение будет автоматически скомпилировано и развернуто в нашем Knative в кластере Kubernetes.

1) Войдите в свою панель Tekton Dashboard и выберите «Webhooks» в меню навигации

2) На странице «Create Webhook» заполните все необходимые данные:

·    Name: уникальное имя веб-перехватчика (например, tenst-simple-app-webhook)

·    Repository URL: полный URL-адрес вашего репозитория GitHub

·    Access Token:

o  Нажмите кнопку «+»

o  Заполните уникальное имя для токена (например, tenst-github-token)

o  Вставьте токен, который мы создали ранее

o  Нажмите кнопку «Create».

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_1.png)

·    Namespace: tekton-pipelines

·    Pipeline: конвейер сборки и развертывания

·    Service Account: tekton-webhook-extension

·    Docker Registry: имя вашей учетной записи в реестре Docker (например, десятки)

·    Нажмите кнопку «Create».

3) Убедитесь, что веб-перехватчик был успешно создан.

 ![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_2.png)

Теперь мы можем его протестировать!

1) Внесите некоторые изменения в репозиторий GitHub и зафиксируйте их. Это создаст новое событие push и запустит наш конвейер:

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_3.png)

2) Дождитесь завершения работы конвейера:

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_4.png)

3) Перейдите в cli и выполните следующую команду, чтобы увидеть URL-адрес вашего приложения:

```
kubectl получить ksvc -n АККАУНТ
```

Измените ACCOUNT на имя своей учетной записи GitHub. Для меня это:

```
kubectl get ksvc -n tenst
```

![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_5.png)


4 ) Получите URL из вывода и откройте его в браузере

 ![](https://docs.ventuscloud.eu/assets/img/tutorials/tekton-pipelines/webhook_create_6.png)

Поздравляю! Если вы видите это сообщение, значит, вы успешно закончили это руководство.

Вот что происходит за кулисами со всем настроенным:

·    Новый коммит создал событие `push` в GitHub

·    GitHub отправит подробную информацию о созданном событии на созданную нами ссылку на веб-перехватчик.

·    Knative обслуживает Сервис для нашего веб-перехватчика, который будет получать информацию о событиях

·    Событие Knative будет создано и отправлено в службу `sink` в Knative.

·    Служба `sink` сгенерирует новый PipelineRun для созданного нами конвейера.

·    PipelineRun вызовет два TaskRuns - `buildah` (для сборки) и `knctl-deploy` (для развертывания).

·    `buildah` TaskRun клонирует репозиторий GitHub, создаст новый образ с помощью Dockerfile и отправит его в реестр Docker.

·    `knctl-deploy` TaskRun запустится после успешного завершения buildah - он создаст новый Knative Service, и его URL-адрес будет иметь вид http://repo_name.account.domain.com

Давайте еще раз напомним, что мы сделали:

·    Создали новый кластер Kubernetes.

·    Развернули Tekton Pipelines.

·    Развернули панель Tekton Dashboard.

·    Развернули расширение Tekton Webhooks.

·    Развернули версию Istio.

·    Развернули Knative.

·    Создали задачи для пайплайнов.

·    Создали пайплайн.

·    Создали секреты для репозиториев GitHub и Docker.

·    Создали новый веб-перехватчик Tekton.

·    Проверили, что приложение успешно создано и развернуто после того, как новый коммит был отправлен в репозиторий GitHub.

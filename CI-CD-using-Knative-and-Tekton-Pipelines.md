CI / CD с использованием Knative и Tekton Pipelines

https://docs.ventuscloud.eu/docs/tutorials/tekton-pipelines

Tekton - это собственный конвейерный ресурс Kubernetes. Проект Tekton Pipelines предоставляет ресурсы в стиле Kubernetes для объявления конвейеров в стиле CI / CD.

Knative - это платформа на основе Kubernetes для развертывания современных бессерверных рабочих нагрузок и управления ими. Нативные компоненты строятся на основе Kubernetes, абстрагируясь от сложных деталей и позволяя разработчикам сосредоточиться на том, что важно.

В этом руководстве мы будем использовать расширение Kubernetes, Knative, Tekton Pipelines, Dashboard и Webhooks для настройки CI? CD с вашим репозиторием GitHub.

 

Оглавление

1)   Создание нового кластера Kubernetes

2)   Получение доступа к вашему кластеру с помощью cli

3)   Развертывание расширения Tekton Pipelines, Dashboard и Webhooks

4)   Развертывание Tekton Dashboard

5)   Развертывание расширения Tekton Webhooks

6)   Настройка объектов Tekton

7)   Создание нового веб-перехватчика

 

Создание нового кластера Kubernetes

Создайте новый кластер, используя эту статью: Kubernetes Cluster

Используйте следующие параметры для вашего кластера:

·    Master count: 1

·    Node count: 1

·    Docker volume size (GB): 100

·    Node flavor: VC-4

·    Master node flavor: VC-2

 

Получение доступа к вашему кластеру с помощью cli

Теперь вы можете получить доступ к своему кластеру с помощью этой статьи: Доступ к кластеру Kubernetes с помощью CLI.

 

Развертывание расширения Tekton Pipelines, Dashboard и Webhooks

Примечание:

Все установки выполняются через cli с использованием kubectl. Kubectl - это интерфейс командной строки для запуска команд в кластерах Kubernetes.

Совет: установите kubectl

Для установки kubectl используйте официальные документы kubernetes: https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux

1) Установите последнюю версию Tekton Pipelines (на момент написания этого руководства была последней версией 0.5.2):

kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml

 

2) Убедитесь, что все модули Tekton Pipelines работают:

kubectl get pods -n tekton-pipelines

 

Развертывание Tekton Dashboard 

1) Установите Tekton Dashboard (на момент написания этого руководства была последней версией 0.1.1):

kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.1/release.yaml

 

2) Убедитесь, что модуль Tekton Dashboard запущен:

kubectl get pods -n tekton-pipelines

 

3) Измените тип службы Tekton Dashboard с «ClusterIP» на «LoadBalancer», чтобы получить внешний IP-адрес и сделать его доступным извне:

kubectl patch service tekton-dashboard -n tekton-pipelines -p '{"spec": {"type": "LoadBalancer"}}'

 

4) Подождите, пока вашему сервису не будет присвоен внешний IP (обычно меньше минуты):

kubectl get svc -n tekton-pipelines

 

5) Получите свой внешний IP-адрес и порт и откройте панель управления в браузере:

http://188.40.161.51:9097

 

 

 

Развертывание расширения Tekton Webhooks

Важная заметка:

На момент написания этого руководства расширение Tekton Webhooks было частью «экспериментального» репозитория и находилось в стадии разработки. Имейте в виду, что некоторые шаги могут измениться в будущем. Вот ссылка на инструкции по установке, которые мы рассмотрим дальше: https://github.com/tektoncd/experimental/blob/master/webhooks-extension/README.md

1) Установите необходимые компоненты:

·    Первыми обязательными компонентами являются Tekton Pipelines и Tekton Dashboard, и мы их уже установили.

·    Установите Istio, следуя инструкции https://knative.dev/docs/install/installing-istio/, или вы можете запустить этот скрипт https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_istio.sh вот так: ./install_istio.sh 1.1.7 (рекомендуется Istio версии 1.1.7).

o  Если вы выбрали запуск скрипта, вам нужно сначала установить Helm (https://github.com/helm/helm/releases):

§ wget https://raw.githubusercontent.com/helm/helm/master/scripts/get

§ chmod +x ./get

§ ./get

§ helm init

o  Продолжите скрипт:

§ wget https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_istio.sh

§ chmod +x ./install_istio.sh

§ ./install_istio.sh 1.1.7

 

·    Убедитесь, что необходимые компоненты Istio (нам нужны только два: istio-ingressgateway и istio-pilot) работают:

kubectl get pods -n istio-system

 

·    Установите Knative Eventing, Eventing Sources & Serving, следуя инструкции https://knative.dev/docs/install/index.html, или вы можете запустить этот скрипт https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/ scripts / install_knative.sh, например: ./install_knative.sh v0.6.0 (настоятельно рекомендуется Knative версии 0.6.0).

o  Если вы выбрали запуск сценария, выполните следующие действия:

§ wget https://raw.githubusercontent.com/tektoncd/experimental/master/webhooks-extension/scripts/install_knative.sh

§ chmod +x ./install_knative.sh

§ ./install_knative.sh v0.6.0

§ kubectl apply -f https://github.com/knative/eventing-sources/releases/download/v0.6.0/eventing-sources.yaml

o  Убедитесь, что необходимые компоненты Knative запущены:

kubectl get pods --all-namespaces | grep knative

 

 

2) Выполните следующую команду, чтобы установить расширение Tekton Webhooks и его зависимости:

·    kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.1/webhooks-extension_release.yaml -n tekton-pipelines

·    Убедитесь, что все модули работают:

kubectl get pods -n tekton-pipelines

Убедитесь, что раздел «Webhooks» добавлен в вашу панель инструментов Tekton:

 

 

 

Настройка объектов Tekton

В этой части руководства мы настроим все необходимые объекты Tekton и репозитории GitHub и Docker.

Начнем с подготовительных действий. Сначала будет репозиторий GitHub:

1) Войдите в свою учетную запись (или создайте новую). 2) Используйте существующий репозиторий, который вы хотите протестировать, или используйте мою версию вилки:

·    Перейдите на https://github.com/TenSt/simple-app (это тоже форк mchmarny / simple-app, но с моими изменениями).

·    Нажмите кнопку «Fork» в правом верхнем углу и выберите свою учетную запись в качестве места назначения для создания новой вилки.

·    В результате у вас должен быть собственный репозиторий с уникальным полным именем, например: https://github.com/TenSt/simple-app

Затем нам нужно создать токен, который мы будем использовать для аутентификации из нашего расширения Tekton Webhooks в репозитории GitHub:

1)Нажмите кнопку своего профиля в правом верхнем углу и перейдите в настройки «Settings».

2) Перейдите в «Developer settings» в нижней части меню «Personal settings», а затем в «Personal access tokens»

3) Нажмите кнопку «Generate new token».

 

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

kubectl get svc istio-ingressgateway -n istio-system

2) Настройте запись DNS с подстановочными знаками типа «A», которая указывает на этот ВНЕШНИЙ IP-адрес из предыдущей команды. В моем случае: * .tekton-tutorial.tenst.ml указывает на 188.40.161.33.

3) Создайте новый файл yaml с именем knative-domain-config.yaml:

4) Измените «tekton-tutorial.tenst.ml» на свой домен, который вы настроили ранее.

5) Примените его к своему кластеру:

kubectl apply -f knative-domain-config.yaml

 

Теперь все запросы к * .tekton-tutorial.tenst.ml будут перенаправляться на нашу службу istio-ingressgateway, которая направит их на соответствующие службы в нашем кластере Kubernetes.

Еще одна вещь, которую необходимо настроить, - это источник событий Knative - GitHubSource:

Создайте файл gitHubSource.yaml - он создаст Knative GitHubSource, который указывает на репозиторий GitHub:

2) Заполните следующие поля:

·    ownerAndRepository - смените его на свой аккаунт / репо

·    accessToken: secretKeyRef: name - измените его на какое-нибудь уникальное имя

·    secretToken: secretKeyRef: name - должно быть таким же, как и предыдущее

3) Примените его к своему кластеру:

kubectl apply -f gitHubSource.yaml -n tekton-pipelines

 

Давайте продолжим и настроим создание новых задач, ресурсов и конвейера Tekton:

1) Примените задачу buildah - она будет использоваться для сборки нашего приложения:

kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/buildah/buildah.yaml -n tekton-pipelines

2)Создайте файл knctl-role.yaml - он создаст и привяжет необходимые разрешения для уже существующей учетной записи службы:

 

Он состоит из двух частей:

·    Создание ClusterRole в нашем кластере Kubernetes, у которого есть полный доступ ко всем ресурсам в ресурсе «serve.knative.dev» и «пространства имен».

·    Создание привязки ClusterRoleBinding, которая привяжет новую роль к существующей учетной записи tekton-webhooks-extension (под которой будут работать наши конвейеры).

3) Примените его к своему кластеру:

kubectl apply -f knctl-role.yaml -n tekton-pipelines

 

4) Создайте файл knctl-task.yaml - он создаст новые таксы с именем knctl-deploy, которые будут использоваться для развертывания нашего приложения на Knative в нашем кластере:

 

Он состоит из 4 частей:

·    Получите kubeconfig из кластера и вставьте его в /builder/home/.kube/config - это необходимо, поскольку knctl не поддерживает конфигурацию inCluster.

·    Получите имя учетной записи из URL-адреса репозитория GitHub

·    Создайте новое пространство имен так же, как GitHub

·    Разверните наше приложение на Knative

5) Примените его к своему кластеру:

kubectl apply -f knctl-task.yaml -n tekton-pipelines

 

6) Создайте файл build-and-deploy-pipeline.yaml - он создаст новый конвейер с именем build-and-deploy-pipeline, который будет использоваться для развертывания нашего приложения на Knative в нашем кластере:

 

Он состоит из 2 заданий, которые мы создали ранее:

·    buildah - для создания и отправки изображения с помощью нашего приложения.

·    kntcl-deploy - для развертывания вновь созданного образа на Knative в нашем кластере.

7) Примените его к своему кластеру:

kubectl apply -f build-and-deploy-pipeline.yaml -n tekton-pipelines

 

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

 

Создание нового веб-перехватчика

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

·    Namespace: tekton-pipelines

·    Pipeline: конвейер сборки и развертывания

·    Service Account: tekton-webhook-extension

·    Docker Registry: имя вашей учетной записи в реестре Docker (например, десятки)

·    Нажмите кнопку «Create».

3)Убедитесь, что веб-перехватчик был успешно создан.

 

Теперь мы можем его протестировать!

1)Внесите некоторые изменения в репозиторий GitHub и зафиксируйте их. Это создаст новое событие push и запустит наш конвейер:

2) Дождитесь завершения работы конвейера:

3) Перейдите в cli и выполните следующую команду, чтобы увидеть URL-адрес вашего приложения:

kubectl получить ksvc -n АККАУНТ

Измените ACCOUNT на имя своей учетной записи GitHub. Для меня это:

kubectl get ksvc -n tenst

 

4 ) Получите URL из вывода и откройте его в браузере

 

Поздравляю! Если вы видите это сообщение, значит, вы успешно закончили это руководство.

Вот что происходит за кулисами со всем настроенным:

·    Новый коммит создал событие PUSH в GitHub

·    GitHub отправит подробную информацию о созданном событии на созданную нами ссылку на веб-перехватчик.

·    Knative обслуживает Сервис для нашего веб-перехватчика, который будет получать информацию о событиях

·    Событие Knative будет создано и отправлено в службу Sink в Knative.

·    Служба приемника сгенерирует новый PipelineRun для созданного нами конвейера.

·    PipelineRun вызовет два TaskRuns - buildah (для сборки) и knctl-deploy (для развертывания).

·    buildah TaskRun клонирует репозиторий GitHub, создаст новый образ с помощью Dockerfile и отправит его в реестр Docker.

·    knctl-deploy TaskRun запустится после успешного завершения buildah - он создаст новый Knative Service, и его URL-адрес будет иметь вид http://repo_name.account.domain.com

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


Первый взгляд на Tekton Pipelines

https://octopus.com/blog/introduction-to-tekton-pipelines

Kubernetes быстро превращается из платформы оркестровки Docker в облачную операционную систему общего назначения. Благодаря [операторам](https://octopus.com/blog/operators-with-kotlin), Kubernetes получает возможность изначально управлять высокоуровневыми концепциями и бизнес-процессами, что означает, что вы больше не управляете строительными блоками модулей, служб и развертываний, а вместо этого описываете вещи, которые эти строительные блоки могут создавать, например веб-серверы, базы данных, непрерывное развертывание, управление сертификатами и многое другое.

При развертывании в кластере Kubernetes Tekton Pipelines предоставляет возможность определять и выполнять задачи сборки, входные и выходные данные в форме простых значений или сложных объектов, таких как образы Docker, и объединять эти ресурсы в конвейеры. Эти новые ресурсы Kubernetes и контроллеры, которые ими управляют, приводят к созданию автономной платформы CI / CD, размещенной в кластере Kubernetes.

В этом посте мы рассмотрим простой конвейер сборки, работающий на MicroK8S.

### Подготовка тестового кластера Kubernetes

В этой публикации я использую [MicroK8S](https://microk8s.io/) для создания кластера Kubernetes. MicroK8S здесь полезен, потому что он предлагает выбор [официальных надстроек](https://microk8s.io/docs/addons), одна из которых - реестр образов Docker. Поскольку наш конвейер создает образ Docker, нам нужно где-то его разместить, а надстройка реестра MicroK8S предоставляет нам эту функциональность с помощью одной команды:

```
microk8s.enable registry
```

Нам также необходимо включить поиск DNS из кластера MicroK8S. Это делается путем включения надстройки DNS:

```
microk8s.enable dns
```

### Установка Tekton Pipelines

Установка Tekton Pipelines выполняется с помощью одной команды `kubectl` (или `microk8s.kubectl` в нашем случае): 

```
microk8s.kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Теперь мы можем создавать ресурсы Tekton в нашем кластере Kubernetes.

### Задача "Hello World"

Задачи содержат отдельные шаги, которые выполняются для выполнения работы. В приведенном ниже примере у нас есть задача с одним шагом, которая выполняет команду `echo` с аргументами `Hello World` в контейнере, созданном из образа `ubuntu`.

В YAML ниже показан наш файл `helloworldtask.yml`:

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: echo-hello-world
spec:
  steps:
    - name: echo
      image: ubuntu
      command:
        - echo
      args:
        - "Hello World"
```



Ресурс задачи создается в кластере Kubernetes командой:

```
microk8s.kubectl apply -f helloworldtask.yml
```

Задача описывает, как должна выполняться работа, но создание ресурса задачи не приводит к выполнению каких-либо действий. Ресурс запуска задачи ссылается на задачу, и создание ресурса запуска задачи запускает Tekton для выполнения шагов в указанной задаче.

В YAML ниже показан наш файл `helloworldtaskrun.yml`:

```
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: echo-hello-world-task-run
spec:
  taskRef:
    name: echo-hello-world
```

Ресурс запуска задачи создается в кластере Kubernetes с помощью команды:

```
microk8s.kubectl apply -f helloworldtaskrun.yml
```

### Создание образа Docker

Чтобы выйти за рамки этого примера hello world, мы рассмотрим канонический вариант использования конвейера сборки Tekton, который заключается в компиляции и отправке образа Docker. Чтобы продемонстрировать эту функциональность, мы создадим наше примерное приложение [RandomQuotes](https://github.com/OctopusSamples/RandomQuotes-Java).

Мы запускаем конвейер с конвейерного ресурса. Ресурсы конвейера обеспечивают независимый метод определения входных данных для процесса сборки.

Первый вход, который нам нужен, - это репозиторий Git, в котором хранится наш код. Ресурсы конвейера имеют несколько известных типов, и здесь мы определяем ресурс конвейера git, указывающий URL-адрес и ветвь, содержащую наш код:

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: randomquotes-git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/OctopusSamples/RandomQuotes-Java.git
```

Затем мы определяем реестр Docker, содержащий наш скомпилированный образ. В этом случае полезна надстройка реестра MicroK8S, поскольку она предоставляет доступ к реестру Docker по адресу http://registry.container-registry.svc.cluster.local:5000.

Вот конвейерный ресурс типа `image`, определяющий образ Docker, который мы создадим как `registry.container-registry.svc.cluster.local:5000/randomquotes`:

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: randomquotes-image
spec:
  type: image
  params:
    - name: url
      value: registry.container-registry.svc.cluster.local:5000/randomquotes
```

Определив входной исходный код и целевой образ Docker, мы можем создать задачу для создания образа Docker и отправить его в репозиторий.

Обычно создание образов Docker выполняется клиентом Docker непосредственно в операционной системе хоста. Однако в Kubernetes все выполняется внутри Docker, поэтому возникает вопрос: как запустить Docker внутри Docker?

За последние несколько лет появилось множество инструментов, предназначенных для выполнения процессов, предоставляемых Docker CLI и демоном, но без какой-либо зависимости от самого Docker. К ним относятся такие инструменты, как [umoci](https://github.com/openSUSE/umoci) для распаковки и переупаковки образов Docker, [Kaniko](https://github.com/GoogleContainerTools/kaniko) и [Buildah](https://github.com/containers/buildah) для создания образов Docker из файла Docker и Podman для запуска образов Docker.

Мы будем использовать Kaniko в нашей задаче Tekton, чтобы создать образ Docker внутри контейнера Docker, предоставленного Kubernetes. YAML ниже показывает полную задачу:

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  inputs:
    resources:
      - name: docker-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.17.1
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url)
        - --context=$(inputs.params.pathToContext)
```

Стоит отметить несколько важных аспектов этой задачи.

В этой задаче есть два свойства, которые относятся к ресурсам конвейера, которые мы создали выше.

Входной ресурс типа `git`:

```
inputs:
    resources:
      - name: docker-source
        type: git
```

И вывод типа `image`:

```
outputs:
  resources:
    - name: builtImage
      type: image
```

Есть два дополнительных входных параметра, которые определяют пути, используемые для процесса сборки Docker:

```
spec:
  inputs:
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
```

Обратите внимание, что путь `/workspace/docker-source` - это соглашение, используемое ресурсами `git`, с каталогом `docker-source`, совпадающим с именем ввода.

Затем у нас есть один шаг, который создает образ Docker. Сборка выполняется в контейнере, созданном из образа `gcr.io/kaniko-project/executor:v0.17.1` image, который предоставляет Kaniko:

```
spec:
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.17.1
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url)
        - --context=$(inputs.params.pathToContext)
```

И наконец, запуск задачи используется для связывания ресурсов задачи и конвейера. Этот ресурс сопоставляет входные данные `docker-source` задачи с ресурсом конвейера `randomquotes-git`, а выходные данные builtImage - с ресурсом конвейера `randomquotes-image`.

После создания этого ресурса запускается сборка:

```
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: docker-source
        resourceRef:
          name: randomquotes-git
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/docker-source
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: randomquotes-image
```

### Взаимодействие со сборками

Сам Tekton не предоставляет никаких панелей управления или графического интерфейса для взаимодействия с заданиями. Однако есть [инструмент командной строки](https://github.com/tektoncd/cli) для управления заданиями Tekton.

Инструмент Tekton CLI предполагает, что `kubectl` настроен, но MicroK8S поддерживает отдельный инструмент под названием `microk8s.kubectl`. Самый простой способ настроить `kubectl` - использовать следующую команду, которая копирует файл конфигурации MicroK8S в стандартное расположение для `kubectl`:

```
sudo microk8s.kubectl config view --raw > $HOME/.kube/config
```

На этом этапе мы можем получить статус задачи с помощью команды:

```
tkn taskrun logs build-docker-image-from-git-source-task-run
```

![](https://habrastorage.org/webt/pf/57/b7/pf57b7mji4vosgcxojls0sjjdvk.png)

### Подходит ли вам **Tekton**?

Составляя сборки с образами Docker, Tekton устраняет накладные расходы на поддержку набора специализированных агентов сборки. В наши дни для каждого инструмента и языка предоставляется поддерживаемый образ Docker, что упрощает соблюдение новой нормы шестимесячного цикла выпуска для основных языковых версий.

Kubernetes также является естественной платформой для удовлетворения гибких и краткосрочных требований сборки программного обеспечения. Почему десять специализированных агентов бездействуют, если между ними может быть пять узлов, планирующих сборки?

Однако я подозреваю, что Tekton не находится на том уровне, необходимом для большинства инженерных команд. Инструмент `tkn` CLI будет знаком любому, кто раньше использовал `kubectl`, но сложно понять общее состояние ваших сборок из терминала. Не говоря уже о создании сборок с помощью `kubectl create -f taskrun.yml`, которые быстро устаревают.

Доступна [информационная панель](https://github.com/tektoncd/dashboard), но это простой пользовательский интерфейс по сравнению с существующими инструментами CI.

![](https://habrastorage.org/webt/76/r6/_q/76r6_q8jip4bd7ghg0hs6jmpoxw.png)

Тем не менее, Tekton - это мощная основа для создания инструментов для разработчиков. [Jenkins X](https://jenkins-x.io/) и [OpenShift Pipelines](https://www.openshift.com/learn/topics/pipelines) - две такие платформы, которые используют Tekton.

### Вывод

Kubernetes решает многие из требований для запуска приложений, таких как аутентификация, авторизация, инструменты интерфейса командной строки, управление ресурсами, проверки работоспособности и многое другое. Тот факт, что кластер Kubernetes может размещать полностью функциональный сервер CI с помощью одной команды, является свидетельством того, насколько гибким является Kubernetes.

С такими проектами, как [Jenkins X](https://jenkins-x.io/) и [OpenShift Pipelines](https://www.openshift.com/learn/topics/pipelines), Tekton находится в начале пути к основным рабочим процессам разработки. Но как отдельный проект Tekton находится на низком уровне своих возможностей, чтобы его могло использовать большинство команд разработчиков, хотя бы потому, что у немногих людей будет опыт, чтобы поддержать его.

Создание пайплайнов CI с помощью Tekton (Часть 2/2)

https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-2/

В этой статье мы собираемся продолжить создание пайплайна CI с [Tekton](https://tekton.dev/). В [первой](https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-1/) части мы установили Tekton на локальный кластер [kind](https://kind.sigs.k8s.io/) и определили нашу первую задачу, которая клонирует репозиторий GitHub и запускает тесты приложений для приложения Go ([repo](https://github.com/arthurk/tekton-example)).

В этой части мы собираемся создать задачу, которая создаст образ Docker для нашего приложения Go и отправит его в [DockerHub](https://hub.docker.com/). После этого мы объединим наши задачи в пайплайн.



### Добавление учетных данных DockerHub

Для создания и отправки нашего образа Docker мы используем [Kaniko](https://github.com/GoogleContainerTools/kaniko), который может создавать образы Docker внутри кластера Kubernetes, независимо от демона Docker.

Kaniko построит и запустит изображение в той же команде. Это означает, что перед запуском нашей задачи нам необходимо настроить учетные данные для DockerHub, чтобы образ докера можно было отправить в реестр.

Учетные данные сохраняются в секрете Kubernetes. Создайте файл с именем [secret.yaml](https://github.com/arthurk/tekton-example/blob/master/04-secret.yaml) со следующим содержимым и замените myusername и mypassword своими учетными данными DockerHub:

```
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
stringData:
    username: myusername
    password: mypassword
```



Обратите внимание на аннотацию tekton.dev/docker-0 в метаданных, которая сообщает Tekton, к какому реестру Docker принадлежат эти учетные данные.

Затем мы создаем ServiceAccount, который использует секрет базового доступа пользователя. Создайте файл с именем [serviceaccount.yaml](https://github.com/arthurk/tekton-example/blob/master/05-serviceaccount.yaml) со следующим содержимым:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```

Примените оба файла с помощью kubectl:

```
$ kubectl apply -f secret.yaml
secret/basic-user-pass created

$ kubectl apply -f serviceaccount.yaml
serviceaccount/build-bot created
```



Теперь мы можем использовать этот ServiceAccount (названный `build-bot`) при запуске задач или пайплайнов Tekton, указав `serviceAccountName`. Мы увидим примеры этого ниже.



### Создание задачи для сборки и отправки образа Docker

Теперь, когда учетные данные настроены, мы можем продолжить, создав задачу, которая будет создавать и отправлять образ Docker.

Создайте файл с именем [task-build-push.yaml](https://github.com/arthurk/tekton-example/blob/master/06-task-build-push.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push
spec:
  resources:
    inputs:
      - name: repo
        type: git
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v1.3.0
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
      command:
        - /kaniko/executor
        - --dockerfile=Dockerfile
        - --context=/workspace/repo/src
        - --destination=arthurk/tekton-test:latest
```



Подобно первой задаче, эта задача принимает репозиторий git в качестве входных данных (имя входа - репо) и состоит только из одного шага, поскольку Канико создает и отправляет изображение в той же команде.

Обязательно создайте репозиторий DockerHub и замените arthurk / tekton-test своим именем репозитория. В этом примере он всегда будет отмечать и отправлять изображение с последним тегом.

Tekton поддерживает [параметры](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md#specifying-parameters), позволяющие избежать жесткого кодирования таких значений. Однако, чтобы не усложнять этот урок, я их не упомянул.

Переменная окружения `DOCKER_CONFIG` требуется для того, чтобы Kaniko мог [найти учетные данные Docker](https://github.com/tektoncd/pipeline/pull/706).

Примените файл с помощью kubectl:

```
$ kubectl apply -f task-build-push.yaml
task.tekton.dev/build-and-push created
```

Есть два способа протестировать эту задачу: либо вручную создать определение TaskRun и затем применить его с помощью kubectl, либо с помощью Tekton CLI (tkn).

В следующих двух разделах я покажу оба метода.



### Запустите задачу с помощью kubectl

Чтобы запустить задачу с помощью kubectl, мы создаем TaskRun, который выглядит идентично [предыдущему](https://github.com/arthurk/tekton-example/blob/master/03-taskrun.yaml), за исключением того, что теперь мы указываем ServiceAccount (`serviceAccountName`) для использования при выполнении задачи.

Создайте файл с именем [taskrun-build-push.yaml](https://github.com/arthurk/tekton-example/blob/master/07-taskrun-build-push.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-and-push
spec:
  serviceAccountName: build-bot
  taskRef:
    name: build-and-push
  resources:
    inputs:
      - name: repo
        resourceRef:
          name: arthurk-tekton-example
```

Примените задачу и проверьте журнал модуля, перечислив все модули, которые начинаются с имени задачи `build-and-push`:

```
$ kubectl apply -f taskrun-build-push.yaml
taskrun.tekton.dev/build-and-push created

$ kubectl get pods | grep build-and-push
build-and-push-pod-c698q   2/2     Running     0          4s

$ kubectl logs --all-containers build-and-push-pod-c698q --follow
{"level":"info","ts":1588478267.3476844,"caller":"creds-init/main.go:44", "msg":"Credentials initialized."}
{"level":"info","ts":1588478279.2681644,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
{"level":"info","ts":1588478279.3249557,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}
INFO[0004] Resolved base name golang:1.14-alpine to golang:1.14-alpine
INFO[0004] Retrieving image manifest golang:1.14-alpine
INFO[0012] Built cross stage deps: map[]
...
INFO[0048] Taking snapshot of full filesystem...
INFO[0048] Resolving paths
INFO[0050] CMD ["app"]
```

Задача выполнена без проблем, и теперь мы можем вытащить / запустить наш образ Docker:

```
$ docker run arthurk/tekton-test:latest
hello world
```



### Запустите задачу с помощью Tekton CLI

Запуск задачи с помощью Tekton CLI удобнее. С помощью одной команды он генерирует манифест TaskRun из определения задачи, применяет его и отслеживает журналы.

```
$ tkn task start build-and-push --inputresource repo=arthurk-tekton-example --serviceaccount build-bot --showlog
Taskrun started: build-and-push-run-ctjvv
Waiting for logs to be available...
[git-source-arthurk-tekton-example-p9zxz] {"level":"info","ts":1588479279.271127,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
[git-source-arthurk-tekton-example-p9zxz] {"level":"info","ts":1588479279.329212,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}

[build-and-push] INFO[0004] Resolved base name golang:1.14-alpine to golang:1.14-alpine
[build-and-push] INFO[0008] Retrieving image manifest golang:1.14-alpine
[build-and-push] INFO[0012] Built cross stage deps: map[]
...
[build-and-push] INFO[0049] Taking snapshot of full filesystem...
[build-and-push] INFO[0049] Resolving paths
[build-and-push] INFO[0051] CMD ["app"]
```

То, что происходит в фоновом режиме, аналогично тому, что мы делали с kubectl в предыдущем разделе, но на этот раз нам нужно выполнить только одну команду.



### Создание конвейера

Теперь, когда у нас есть наши Задачи (тестирование, сборка и отправка), мы можем создать пайплайн, который будет запускать их последовательно: сначала он запустит тесты приложения, и если они пройдут, он создаст образ Docker и отправит его в DockerHub.

Создайте файл с именем [pipeline.yaml](https://github.com/arthurk/tekton-example/blob/master/08-pipeline.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: test-build-push
spec:
  resources:
    - name: repo
      type: git
  tasks:
    # Run application tests
    - name: test
      taskRef:
        name: test
      resources:
        inputs:
          - name: repo      # name of the Task input (see Task definition)
            resource: repo  # name of the Pipeline resource

    # Build docker image and push to registry
    - name: build-and-push
      taskRef:
        name: build-and-push
      runAfter:
        - test
      resources:
        inputs:
          - name: repo      # name of the Task input (see Task definition)
            resource: repo  # name of the Pipeline resource
```

Первое, что нам нужно определить, это то, какие ресурсы требуются нашему конвейеру. Ресурс может быть входом или выходом. В нашем случае у нас есть только ввод: репозиторий git с исходным кодом нашего приложения. Мы называем ресурс репо.

Далее мы определяем наши задачи. Каждая задача имеет `taskRef` (ссылку на задачу) и передает требуемые входные данные.

Примените файл с помощью kubectl:

```
$ kubectl apply -f pipeline.yaml
pipeline.tekton.dev/test-build-push created
```

Подобно тому, как мы можем работать как Task, создав TaskRun, мы можем запустить Pipeline, создав PipelineRun.

Это можно сделать с помощью kubectl или Tekton CLI. В следующих двух разделах я покажу оба пути.



### Запустите конвейер с помощью kubectl

Чтобы запустить файл с помощью kubectl, мы должны создать PipelineRun. Создайте файл с именем [pipelinerun.yaml](https://github.com/arthurk/tekton-example/blob/master/09-pipelinerun.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: test-build-push-pr
spec:
  serviceAccountName: build-bot
  pipelineRef:
    name: test-build-push
  resources:
  - name: repo
    resourceRef:
      name: arthurk-tekton-example
```

Примените файл, получите Pod'ы с префиксом PiplelineRun и просмотрите журналы, чтобы получить вывод контейнера:

```
$ kubectl apply -f pipelinerun.yaml
pipelinerun.tekton.dev/test-build-push-pr created

$ kubectl get pods | grep test-build-push-pr
test-build-push-pr-build-and-push-gh4f4-pod-nn7k7   0/2     Completed   0          2m39s
test-build-push-pr-test-d2tck-pod-zh5hn             0/2     Completed   0          2m51s

$ kubectl logs test-build-push-pr-build-and-push-gh4f4-pod-nn7k7 --all-containers --follow
INFO[0005] Resolved base name golang:1.14-alpine to golang:1.14-alpine
INFO[0005] Retrieving image manifest golang:1.14-alpine
...
INFO[0048] Taking snapshot of full filesystem...
INFO[0048] Resolving paths
INFO[0050] CMD ["app"]
```

Затем мы запустим тот же пайплайн, но вместо этого мы будем использовать Tekton CLI.



### Запуск конвейера с помощью Tekton CLI

При использовании CLI нам не нужно писать PipelineRun, он будет сгенерирован из манифеста Pipeline. Используя аргумент `--showlog`, он также отображает журналы задач (контейнеров):

```
$ tkn pipeline start test-build-push --resource repo=arthurk-tekton-example --serviceaccount build-bot --showlog

Pipelinerun started: test-build-push-run-9lmfj
Waiting for logs to be available...
[test : git-source-arthurk-tekton-example-k98k8] {"level":"info","ts":1588483940.4913514,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
[test : git-source-arthurk-tekton-example-k98k8] {"level":"info","ts":1588483940.5485842,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}

[test : run-test] PASS
[test : run-test] ok  	_/workspace/repo/src	0.006s

[build-and-push : git-source-arthurk-tekton-example-2vqls] {"level":"info","ts":1588483950.2051432,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
[build-and-push : git-source-arthurk-tekton-example-2vqls] {"level":"info","ts":1588483950.2610846,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}

[build-and-push : build-and-push] INFO[0003] Resolved base name golang:1.14-alpine to golang:1.14-alpine
[build-and-push : build-and-push] INFO[0003] Resolved base name golang:1.14-alpine to golang:1.14-alpine
[build-and-push : build-and-push] INFO[0003] Retrieving image manifest golang:1.14-alpine
...
```



### Резюме

В [первой](https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-1/) части мы установили Tekton в локальном кластере Kubernetes, определили задачу и протестировали ее, создав TaskRun через манифест YAML, а также через Tekton CLI tkn.

В этой части мы создали наш первый Tektok Pipeline, который состоит из двух задач. Первый клонирует репо с GitHub и запускает тесты приложений. Второй создает образ Docker и отправляет его в DockerHub.

Все примеры кода доступны [здесь](https://github.com/arthurk/tekton-example).

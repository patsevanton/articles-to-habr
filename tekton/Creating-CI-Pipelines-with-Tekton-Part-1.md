Создание пайплайнов CI с помощью Tekton (Часть 1/2)

https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-1/

В этой статье мы собираемся создать пайплайн непрерывной интеграции (CI) с [Tekton](https://tekton.dev/), фреймворком с открытым исходным кодом для создания конвейеров CI / CD в Kubernetes.

Мы собираемся подготовить локальный кластер Kubernetes через [kind](https://kind.sigs.k8s.io/) и установить на нем Tekton. После этого мы создадим пайплайн, состоящий из двух шагов, который будет запускать модульные тесты приложения, создавать образ Docker и отправлять его в DockerHub.

Это 1 из 2 частей, в которой мы установим Tekton и создадим задачу, запускающую тест нашего приложения. Вторая часть доступна [здесь](https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-2/).



### Создание кластера k8s

Мы используем [kind](http://kind.sigs.k8s.io/) для создания кластера Kubernetes для нашей установки Tekton:

```
$ kind create cluster --name tekton
```



### Установка Tekton

Мы можем установить Tekton, применив файл release.yaml из последней версии репозитория [tektoncd/pipeline](tektoncd/pipeline) на GitHub:

```
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.20.1/release.yaml
```



Это установит Tekton в пространство имен tekton-pipelines. Мы можем проверить успешность установки, указав модули в этом пространстве имен и убедившись, что они находятся в состоянии выполнения.

```
$ kubectl get pods --namespace tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-pipelines-controller-74848c44df-m42gf   1/1     Running   0          20s
tekton-pipelines-webhook-6f764dc8bf-zq44s      1/1     Running   0          19s
```



### Настройка Tekton CLI

Установка интерфейса командной строки не является обязательной, но я считаю, что это удобнее, чем kubectl, при управлении ресурсами Tekton. Примеры, приведенные ниже, покажут оба пути.

Мы можем установить его через Homebrew:

```
$ brew tap tektoncd/tools
$ brew install tektoncd/tools/tektoncd-cli

$ tkn version
Client version: 0.16.0
Pipeline version: v0.20.1
```



### Концепции

Tekton предоставляет пользовательские определения ресурсов ([CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) для Kubernetes, которые можно использовать для определения наших пайплайнов. В этом руководстве мы будем использовать следующие настраиваемые ресурсы:

- Задача: серия шагов, которые выполняют команды (в CircleCI это называется *Job*).

- Пайплайн: набор задач (в CircleCI это называется рабочим процессом *Workflow*)

- PipelineResource: ввод или вывод Pipeline (например, репозиторий git или файл tar)


Мы будем использовать следующие два ресурса для определения выполнения наших задач и пайплайна:

- `TaskRun`: определяет выполнение задачи
- `PipelineRun`: определяет выполнение пайплайна

Например, если мы пишем задачу и хотим ее протестировать, мы можем выполнить ее с помощью `TaskRun`. То же самое относится и к пайплайну: для выполнения конвейера нам нужно создать `PipelineRun`.



### Код приложения

В нашем примере Pipeline мы собираемся использовать приложение Go, которое просто выводит сумму двух целых чисел. Вы можете найти код приложения, тест и Dockerfile в каталоге `src/` этого [репо](https://github.com/arthurk/tekton-example).

 

### Создание нашей первой задачи

Наша первая задача будет запускать тесты приложения внутри клонированного репозитория git. Создайте файл [01-task-test.yaml](https://github.com/arthurk/tekton-example/blob/master/01-task-test.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test
spec:
  resources:
    inputs:
      - name: repo
        type: git
  steps:
    - name: run-test
      image: golang:1.14-alpine
      workingDir: /workspace/repo/src
      command: ["go"]
      args: ["test"]
```



Блок resources: определяет входные данные, необходимые нашей задаче для выполнения своих шагов. Нашему шагу (названному run-test) требуется клонированный репозиторий git с [примером tekton](https://github.com/arthurk/tekton-example/) в качестве входных данных, и мы можем создать эти входные данные с помощью PipelineResource.

Создайте файл с названием [02-pipelineresource.yaml](https://github.com/arthurk/tekton-example/blob/master/02-pipelineresource.yaml):

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: arthurk-tekton-example
spec:
  type: git
  params:
    - name: url
      value: https://github.com/arthurk/tekton-example
    - name: revision
      value: master
```



Тип ресурса git будет использовать git для клонирования репозитория в каталог `/workspace/$input_name` при каждом запуске задачи. Поскольку наш ввод называется repo, код будет клонирован в `/workspace/repo`. Если бы наш ввод был назван foobar, он был бы клонирован в `/workspace/foobar`.

Следующий блок в нашей задаче (`steps:`) определяет команду для выполнения и образ Docker, в котором следует выполнить эту команду. Мы собираемся использовать образ [golang](https://hub.docker.com/_/golang) Docker, так как Go уже установлен.

Для запуска команды go test нам нужно сменить каталог. По умолчанию команда запускается в каталоге `/workspace/repo`, но в нашем репозитории с [примером tekton](https://github.com/arthurk/tekton-example) приложение Go находится в каталоге `src`. Мы делаем это, установив рабочий каталог: `/workspace/repo/src`.

Затем мы указываем команду для запуска (`go test`), но обратите внимание, что команда (`go`) и args (`test`) должны быть определены отдельно в файле YAML.

Примените Task и PipelineResource с помощью kubectl:

```
$ kubectl apply -f 01-task-test.yaml
task.tekton.dev/test created

$ kubectl apply -f 02-pipelineresource.yaml
pipelineresource.tekton.dev/arthurk-tekton-example created
```



### Выполняем нашу задачу

Чтобы запустить нашу задачу, мы должны создать `TaskRun`, который ссылается на ранее созданную задачу и передает все необходимые входные данные (`PipelineResource`).

Создайте файл [03-taskrun.yaml](https://github.com/arthurk/tekton-example/blob/master/03-taskrun.yaml) со следующим содержимым:

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: testrun
spec:
  taskRef:
    name: test
  resources:
    inputs:
      - name: repo
        resourceRef:
          name: arthurk-tekton-example
```



Это примет нашу задачу (`taskRef` - это ссылка на нашу ранее созданную задачу с именем test) с нашим репозиторием git [tekton-example](https://github.com/arthurk/tekton-example) в качестве входных данных (`resourceRef` - это ссылка на наш `PipelineResource` с именем `arthurk-tekton-example`) и выполнит ее.

Примените файл с помощью kubectl, а затем проверьте ресурсы Pods и TaskRun. Pod пройдет через статус `Init:0/2` и `PodInitializing`, а затем успешно:

```
$ kubectl apply -f 03-taskrun.yaml
pipelineresource.tekton.dev/arthurk-tekton-example created

$ kubectl get pods
NAME                READY   STATUS      RESTARTS   AGE
testrun-pod-pds5z   0/2     Completed   0          4m27s

$ kubectl get taskrun
NAME      SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
testrun   True        Succeeded   70s         57s
```

Чтобы увидеть вывод контейнеров, мы можем запустить следующую команду. Обязательно замените `testrun-pod-pds5z` на имя модуля из выходных данных выше (оно будет отличаться для каждого запуска).

```
$ kubectl logs testrun-pod-pds5z --all-containers
{"level":"info","ts":1588477119.3692405,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
{"level":"info","ts":1588477119.4230678,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}
PASS
ok  	_/workspace/repo/src	0.003s
```

Наши тесты прошли, и наша задача была выполнена. Затем мы воспользуемся Tekton CLI, чтобы увидеть, как мы можем упростить весь этот процесс.



### Использование Tekton CLI для запуска задачи

Tekton CLI обеспечивает более быстрый и удобный способ запуска задач.

Вместо того, чтобы вручную писать манифест TaskRun, мы можем запустить следующую команду, которая берет нашу задачу (с именем test), генерирует TaskRun (со случайным именем) и отображает ее журналы:

```
$ tkn task start test --inputresource repo=arthurk-tekton-example --showlog
Taskrun started: test-run-8t46m
Waiting for logs to be available...
[git-source-arthurk-tekton-example-dqjfb] {"level":"info","ts":1588477372.740875,"caller":"git/git.go:136","msg":"Successfully cloned https://github.com/arthurk/tekton-example @ 301aeaa8f7fa6ec01218ba6c5ddf9095b24d5d98 (grafted, HEAD, origin/master) in path /workspace/repo"}
[git-source-arthurk-tekton-example-dqjfb] {"level":"info","ts":1588477372.7954974,"caller":"git/git.go:177","msg":"Successfully initialized and updated submodules in path /workspace/repo"}

[run-test] PASS
[run-test] ok  	_/workspace/repo/src	0.006s
```



### Вывод

Мы успешно установили Tekton в локальном кластере Kubernetes, определили задачу и протестировали ее, создав TaskRun через манифест YAML, а также через Tekton CLI tkn.

Весь пример кода доступен [здесь](https://github.com/arthurk/tekton-example).

В следующей части мы собираемся создать задачу, которая будет использовать [Kaniko](https://github.com/GoogleContainerTools/kaniko) для создания образа Docker для нашего приложения, а затем будет отправлять его в DockerHub. Затем мы создадим пайплайн, который последовательно будет запускать обе наши задачи (запускать тесты приложения, сборку и отправку).

Часть 2 доступна [здесь](https://www.arthurkoziel.com/creating-ci-pipelines-with-tekton-part-2/).

https://www.alibabacloud.com/blog/helm-chart-and-template-basics---part-1_595489

Основы работы с Helm чартами и темплейтами - Часть 1

В этом руководстве мы кратко обсудим, как Helm может помочь упростить управление приложениями Kubernetes, и узнаем, как использовать Helm для создания базового чарта.

Управление приложениями - сложный аспект Kubernetes. Helm значительно упрощает его, предоставляя единый метод упаковки программного обеспечения, поддерживающий контроль версий. Helm устанавливает пакеты (называются Чартами в Helm) для Kubernetes и управляет ими, как это делают yum и apt.

В этом руководстве мы позволим Helm создать для нас базовый чарт. В этом руководстве предполагается, что у вас есть хотя бы базовое понимание того, что такое Helm. Если вы не знакомы с ним, я предлагаю вам ознакомиться с этим руководством, прежде чем приступить к статье: https://www.alibabacloud.com/help/doc-detail/86511.htm

Затем мы будем постепенно вносить изменения, чтобы узнать, как файл значений и части шаблона работают вместе.

С таким базовым рабочим чартом легче работать, чем начинать с нуля.

[Чарт](https://docs.helm.sh/using_helm/) - это пакет Helm. Он содержит все определения ресурсов, необходимые для запуска приложения, инструмента или службы внутри кластера Kubernetes.

Думайте об этом как о Kubernetes-эквиваленте формулы Homebrew, Apt dpkg или Yum RPM-файле.

### Создание полной структуры каталогов рабочего чарта

```
helm create myhelm1
Creating myhelm1
```

Это создает полный рабочий чарт со всеми необходимыми файлами в каталоге myhelm.

```
myhelm1/
  |
  |- .helmignore     # Contains patterns for files to ignore when packaging Helm charts.
  |
  |- Chart.yaml      # Meta Information about your chart
  |
  |- values.yaml     # The default values for your templates
  |
  |- charts/         # Charts that this chart depends on: dependencies 
  |
  |- templates/      # The template files
```

Вам будут представлены некоторые из этих файлов на протяжении всего этого руководства - только тогда, когда нам нужно узнать об этих конкретных файлах.

Цель - как можно скорее использовать чарт для создания работающего экземпляра. Затем мы исследуем, что создал чарт и как он это сделал.

Сначала идет файл `values.yaml`. Он содержит наши значения по умолчанию для объектов Kubernetes, которые мы хотим создать.

Вверху мы видим, что он использует `nginx`. Это загрузка весом 55 МБ. Я предпочитаю быстрые действия во время обучения с помощью `busybox` - загрузка 650 КБ.

Исходные `values.yaml`

```
nano ./myhelm1/values.yaml

# Default values for myhelm1.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  tag: stable
  pullPolicy: IfNotPresent
```

Измените `values.yaml` вверху, чтобы использовать busybox, как показано ниже. Обратите внимание на изменения тегов.

```
nano ./myhelm1/values.yaml

# Default values for myhelm1.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent
```

Далее идет файл `deployment.yaml`.

Это развертывание (deployment), как и любое другое, которое вы используете в Kubernetes. Основное отличие состоит в том, что большинство значений полей он получает из только что отредактированного файла значений.

Отредактируйте файл `deployment.yaml` около строки 27 - добавьте команду. Мы используем образ busybox. Если мы создадим наши поды, они сразу же выйдут, так как ни одна команда или программа не запущены. Команда позволила нашему поду `busybox` спать 60 секунд.

(Вы можете увидеть в отрывке из шаблона ниже, как будут извлечены значения из `values.yaml`. Мы перейдем к синтаксису позже - пока мы сосредотачиваемся на общей картине.)

```
nano ./myhelm1/templates/deployment.yaml

    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          command: ['sh', '-c', 'sleep 60']
```

Теперь мы готовы позволить Helm установить наш отредактированный чарт.

Запустите `helm install ./myhelm1/` и исследуйте вывод.

```
helm install ./myhelm1/

NAME:   loopy-otter
LAST DEPLOYED: Thu Feb 14 08:48:42 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME                 TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
loopy-otter-myhelm1  ClusterIP  10.109.163.87  <none>       80/TCP   0s

==> v1/Deployment
NAME                 DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
loopy-otter-myhelm1  1        0        0           0          0s

==> v1/Pod(related)
NAME                                  READY  STATUS   RESTARTS  AGE
loopy-otter-myhelm1-67b67bf4c8-tsdcq  0/1    Pending  0         0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=myhelm1,app.kubernetes.io/instance=loopy-otter" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

Helm автоматически генерирует название выпуска для вашего: `NAME: loopy-otter`

Ваш будет другим. Ненавижу эти глупые имена. Позже мы будем использовать наши собственные имена.

Мы видим сервис, развертывание и создание пода.

Грубо говоря, Helm прочитал все шаблоны `.yaml` в каталоге шаблонов, а затем интерпретировал эти шаблоны, извлекая значения из файла `values.yaml`.

Примечания относятся к исходному приложению nginx. Это совершенно неправильно для нашего приложения `busybox`.

Эти примечания взяты из NOTES.txt, другого файла шаблона.

Через несколько секунд мы увидим, что наш Pod работает.

```
kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
loopy-otter-myhelm1-67b67bf4c8-tsdcq   0/1     Running   0          13s
```

Демонстрация общего обзора демо готова. Используйте `helm delete`, чтобы удалить наш первый выпуск.

[Релиз](https://docs.helm.sh/using_helm/) - это экземпляр чарта, работающей в кластере Kubernetes.

```
helm delete loopy-otter
release "loopy-otter" deleted
```

### helmignore NOTES.txt

Теперь отредактируйте файл `.helmignore` и добавьте `NOTES.txt` внизу.

`.helmignore` содержит список имен файлов и шаблонов имен файлов, которые Helm должен игнорировать.

```
nano ./myhelm1/.helmignore

NOTES.txt
```

Если вы снова запустите установку, вы увидите, что эти примечания больше не отображаются. 

```
helm install .\myhelm1\ --name test1
NAME:   test1
LAST DEPLOYED: Thu Feb 14 08:56:10 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
test1-myhelm1  ClusterIP  10.96.102.116  <none>       80/TCP   0s

==> v1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test1-myhelm1  1        0        0           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS             RESTARTS  AGE
test1-myhelm1-6f77bf4459-9nxpz  0/1    ContainerCreating  0         0s
```

(Позже мы будем использовать такие заметки, но здесь и сейчас этот файл нам не нужен.)

Удалите наш тестовый релиз test1.

```
helm delete test1
release "test1" deleted
```

### --dry-run --debug

Мы используем `--dry-run` и `--debug`, чтобы исследовать, как Helm интерпретирует наш шаблон и файлы YAML в наших чартах.

Таким образом мы не засоряем наш узел Kubernetes несколькими ненужными объектами.

Давайте попробуем.

```
helm install .\myhelm1\ --name test1 --dry-run --debug
[debug] Created tunnel using local port: '49958'

[debug] SERVER: "127.0.0.1:49958"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

Error: a release named test1 already exists.
Run: helm ls --all test1; to check the status of the release
Or run: helm del --purge test1; to delete it
```

Как видите, релиз может существовать только один раз.

Проверим статус release

```
helm ls --all test1
NAME    REVISION        UPDATED                         STATUS  CHART           APP VERSION     NAMESPACE
test1   1               Thu Feb 14 08:56:10 2019        DELETED myhelm1-0.1.0   1.0             default
```

Мы просто удалили его.
Для тестирования отладки (`debug`) нам понадобится другое название релиза: мы используем test2:

```
helm install .\myhelm1\ --name test2 --dry-run --debug

[debug] Created tunnel using local port: '49970'

[debug] SERVER: "127.0.0.1:49970"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test2
REVISION: 1
RELEASED: Thu Feb 14 08:59:22 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
ingress:
  annotations: {}
  enabled: false
  hosts:
  - chart-example.local
  paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []


HOOKS:
---
# test2-myhelm1-test-connection
apiVersion: v1
kind: Pod
metadata:
  name: "test2-myhelm1-test-connection"
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test2
    app.kubernetes.io/managed-by: Tiller
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:  ['test2-myhelm1:80']
  restartPolicy: Never


MANIFEST:
---
# Source: myhelm1/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: test2-myhelm1
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test2
    app.kubernetes.io/managed-by: Tiller
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: myhelm1
    app.kubernetes.io/instance: test2


---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test2-myhelm1
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test2
    app.kubernetes.io/managed-by: Tiller
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myhelm1
      app.kubernetes.io/instance: test2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myhelm1
        app.kubernetes.io/instance: test2
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          command: ['sh', '-c', 'sleep 60']

          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
```

Очень полезно, но слишком много информации, если мы хотим постоянно редактировать и устанавливать наш чарт.

Прямо сейчас я не буду пытаться все это разобрать, давайте сначала уменьшим вывод.

Под Хуками есть тестовое соединение. Это было полезно для тестирования исходного nginx. Нам это не нужно.

Примерно через 20 строк мы находим `# Source: myhelm1 / templates / service.yaml ... kind: Service` - нам это не нужно - нам нужен только работающий Pod.

Его легко исправить, просто отредактируйте .helmignore и добавьте эти два имени файла внизу.

```
nano ./myhelm1/.helmignore

test-connection.yaml
service.yaml
```

Нашему поду busybox не нужны порты или livenessProbes.

Удалите строки с 29 по 42 из `deployment.yaml`

```
nano ./myhelm1/templates/deployment.yaml

          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
```

Эти ярлыки ниже не добавляют ценности этому руководству, поэтому они удаляются из вывода всех приведенных ниже команд `helm install`.

```
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test4
    app.kubernetes.io/managed-by: Tiller

  selector:
    matchLabels:
      app.kubernetes.io/name: myhelm1
      app.kubernetes.io/instance: test4

    metadata:
      labels:
        app.kubernetes.io/name: myhelm1
        app.kubernetes.io/instance: test4
```

Давайте запустим снова нашу установку. 

```
helm install .\myhelm1\ --name test2 --dry-run --debug
[debug] Created tunnel using local port: '49976'

[debug] SERVER: "127.0.0.1:49976"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test2
REVISION: 1
RELEASED: Thu Feb 14 09:09:55 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
ingress:
  annotations: {}
  enabled: false
  hosts:
  - chart-example.local
  paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test2-myhelm1
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          command: ['sh', '-c', 'sleep 60']
```

Давайте разберемся  данных командах:

- USER-SUPPLIED VALUES: мы их не предоставляли, поэтому здесь ничего не указано. Мы воспользуемся этим через минуту.
- COMPUTED VALUES: показывает рассчитанные значения из `values.yaml`. Он отображается в алфавитном порядке, в то время как наш файл находится в случайном порядке.
- HOOKS: не используются в этом руководстве для начинающих.
- Внизу мы видим наш `deployment.yaml`. Он показывает шаблон со значениями, взятыми из файла `values.yaml`.

Вы можете неоднократно вносить изменения в свои значения и шаблоны и тестировать их с помощью `--dry-run --debug`. Он только показывает, что произойдет, не делая этого. Очень полезно: отладить установку Helm ДО того, как это будет сделано.

Мы довольны результатами отладки, давайте запустим установку.

```
helm install .\myhelm1\ --name test2
NAME:   test2
LAST DEPLOYED: Thu Feb 14 09:12:01 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test2-myhelm1  1        0        0           0          0s

==> v1/Pod(related)
NAME                            READY  STATUS             RESTARTS  AGE
test2-myhelm1-5bd9bb65c7-6pr4q  0/1    ContainerCreating  0         0s
```

Как и ожидалось - происходит развертывание и его Pod. Через несколько секунд Pod запускается.

```
kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
test2-myhelm1-5bd9bb65c7-6pr4q   1/1     Running   0          10s
```

```
helm delete test2
release "test2" deleted
```

### imagePullPolicy = Never

Значения в `values.yaml` заменяют свои заполнители (placeholders) в файлах шаблонов.

Файлы шаблонов также могут получать свои значения от пользователя. Пользователи передают значения программному обеспечению, которое они устанавливают, с помощью флага `--set `в команде установки.

В этой части руководства демонстрируется передача imagePullPolicy в командной строке.

Редактирование не требуется, просто обратите внимание на последнюю строку извлечения файла значений ниже.

Файл значений по умолчанию должен называться values.yaml.

```
nano ./myhelm1/values.yaml

# Default values for myhelm1.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent
```

Теперь посмотрите, где он используется в шаблоне. (в диапазоне 22-25)

```
nano ./myhelm1/templates/deployment.yaml
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

`.Values.image.pullPolicy` получает значение из

- файла values.yaml ... .Values
- содержимого .image.pullPolicy

```
image:
  pullPolicy: IfNotPresent
```

До сих пор в этом руководстве мы использовали `pullPolicy: IfNotPresent`. (Возможно, вы захотите пролистать страницу и увидеть, что это так везде.)

Предположим, что для этого тестового прогона мы НЕ хотим, чтобы образ было извлечен из репозитория. (`imagePullPolicy: Never`)

Из документации Kubernetes:
*imagePullPolicy: Never: предполагается, что образ существует локально. Попытки вытащить изображение не предпринимаются.*

Смотрите приведенную ниже команду пробного запуска, как мы указываем политику с помощью `--set`.

```
helm install .\myhelm1\ --set imagePullPolicy=Never --name test3 --dry-run --debug
[debug] Created tunnel using local port: '50101'

[debug] SERVER: "127.0.0.1:50101"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test3
REVISION: 1
RELEASED: Thu Feb 14 10:10:37 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
imagePullPolicy: Never

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
imagePullPolicy: Never
ingress:
  annotations: {}
  enabled: false
  hosts:
  - chart-example.local
  paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test3-myhelm1
  
spec:
  replicas: 1
  
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          command: ['sh', '-c', 'sleep 60']
```

USER-SUPPLIED VALUES кажутся правильными: `imagePullPolicy: Never`

COMPUTED VALUES: указывают на наличие проблемы:

```
image:
  pullPolicy: IfNotPresent
  tag: base
imagePullPolicy: Never
```

Наша политика `--set` не заменяет политику скачивания образов.

Они имеют разные имена и находятся на разных уровнях yaml.

В развертывании мы видим: `imagePullPolicy: IfNotPresent`: переопределение не выполнено.

Давайте исправим это: смотрите на вторую попытку:

```
helm install .\myhelm1\ --set image.PullPolicy=Never --name test3 --dry-run --debug
[debug] Created tunnel using local port: '50107'

[debug] SERVER: "127.0.0.1:50107"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test3
REVISION: 1
RELEASED: Thu Feb 14 10:14:11 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
image:
  PullPolicy: Never            < - - - - - -

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  PullPolicy: Never            < - - - - - -
  pullPolicy: IfNotPresent     < - - - - - -
  repository: radial/busyboxplus
```

Почти готово, но все равно неверно. Теперь у нас есть две политики, написанные по-разному. (Первая буква в нижнем регистре - это правильная буква, которая появляется в файле значений).

Соглашение гласит, что мы должны называть наши значения, начиная со строчной буквы. Наши values.yaml верны. Наше переопределение командной строки неверно.

Третья попытка, смотрите команду ниже.

```
helm install .\myhelm1\ --set image.pullPolicy=Never --name test3 --dry-run --debug  

[debug] Created tunnel using local port: '50113'

[debug] SERVER: "127.0.0.1:50113"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test3
REVISION: 1
RELEASED: Thu Feb 14 10:15:10 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
image:
  pullPolicy: Never     < - - - - - - - - - - - - -

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: Never
  repository: radial/busyboxplus
  tag: base
ingress:
  annotations: {}
  enabled: false
  hosts:
  - chart-example.local
  paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
replicaCount: 1
resources: {}
service:
  port: 80
  type: ClusterIP
tolerations: []

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test3-myhelm1
  
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: Never     < - - - - - - - - - - - 

          command: ['sh', '-c', 'sleep 60']
```

В приведенном выше развертывании показано, как `imagePullPolicy: Never` ... прошло успешно.

COMPUTED VALUES показывают, что переопределение выполнено правильно.

```
COMPUTED VALUES:
image:
  pullPolicy: Never
```

Вывод отладки выглядит хорошо. Мы готовы установить этот выпуск вживую.

Я хочу скрыть все остальные значения, которые нам не нужны. Отредактируйте файл значений так, чтобы только первые 5 значений не закомментировались.

```
nano ./myhelm1/values.yaml

# Default values for myhelm1.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent

#nameOverride: ""
#fullnameOverride: ""

#service:
#  type: ClusterIP
#  port: 80

#ingress:
#  enabled: false
#  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
#  paths: []
#  hosts:
#    - chart-example.local
#  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

#resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

#nodeSelector: {}

#tolerations: []

#affinity: {}
```

Установим наш чарт

```
helm install .\myhelm1\ --set image.pullPolicy=Never --name test3 --dry-run --debug
[debug] Created tunnel using local port: '50125'

[debug] SERVER: "127.0.0.1:50125"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

Error: render error in "myhelm1/templates/ingress.yaml": template: myhelm1/templates/ingress.yaml:1:14: executing "myhelm1/templates/ingress.yaml" at <.Values.ingress.enab...>: can't evaluate field enabled in type interface {}
```

`Values.ingress.enabled` используется в `myhelm1/templates/ingress.yaml`

Нам не нужен ingress - это часть чарта nginx, с которого мы начали.

Добавьте `ingress.yaml` в конец нашего файла игнорирования.

```
nano ./myhelm1/.helmignore
ingress.yaml
```

Вторая попытка: установить чарт myhelm1 с помощью `image.pullPolicy = Never`
плюс мы добавили `--set replicaCount=3`

```
helm install .\myhelm1\ --set image.pullPolicy=Never --set replicaCount=3 --name test3 --dry-run --debug
[debug] Created tunnel using local port: '50140'

[debug] SERVER: "127.0.0.1:50140"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test3
REVISION: 1
RELEASED: Thu Feb 14 10:23:43 2019
CHART: myhelm1-0.1.0


USER-SUPPLIED VALUES:
image:
  pullPolicy: Never        < * * * = = = = = = = = = = = = = 
replicaCount: 3    < - - - - - - - - - - - - - - - - 


COMPUTED VALUES:
image:
  pullPolicy: Never        < * * * = = = = = = = = = = = = =
  repository: radial/busyboxplus
  tag: base
replicaCount: 3    < - - - - - - - - - - - - - - - - 

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test3-myhelm1
spec:
  replicas: 3   < - - - - - - - - - - - - - - - - - -
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: Never        < * * * = = = = = = = = = = = = =

          command: ['sh', '-c', 'sleep 60']
```

`--set replicaCount` правильно переопределяет значение в deployment.yaml

Сделаем живую установку.

```
helm install .\myhelm1\ --set image.pullPolicy=Never --set replicaCount=3 --name test3
NAME:   test3
LAST DEPLOYED: Thu Feb 14 10:34:45 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test3-myhelm1  3        0        0           0          0s

==> v1/Pod(related)
NAME                          READY  STATUS             RESTARTS  AGE
test3-myhelm1-878d8d7c-7xshs  0/1    Pending            0         0s
test3-myhelm1-878d8d7c-fnjqn  0/1    ContainerCreating  0         0s
test3-myhelm1-878d8d7c-gjw4m  0/1    Pending            0         0s
```

Успешно. ЖЕЛАТЕЛЬНОЕ развертывание - 3, и мы видим, что создаются 3 модуля.

Через несколько секунд у нас есть 3 работающих пода. Обратите внимание на использование команды `helm status`.

```
helm status test3
LAST DEPLOYED: Thu Feb 14 10:34:45 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
test3-myhelm1  3        3        3           3          20s

==> v1/Pod(related)
NAME                          READY  STATUS   RESTARTS  AGE
test3-myhelm1-878d8d7c-7xshs  1/1    Running  0         20s
test3-myhelm1-878d8d7c-fnjqn  1/1    Running  0         20s
test3-myhelm1-878d8d7c-gjw4m  1/1    Running  0         20s
```

Демо завершено. Удалите наш релиз test3.

```
helm delete test3
release "test3" deleted
```

### Определение нового value

Пока что мы удалили значения из `values.yaml`.

Мы также передали значения переопределения в командной строке.

Теперь мы создаем собственное новое значение: `terminationGracePeriodSeconds`

*[terminationGracePeriodSeconds](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#podspec-v1-core) - Необязательная продолжительность в секундах, необходимая для корректного завершения работы модуля. grace period - это продолжительность в секундах после того, как процессы, запущенные в модуле, отправляют сигнал завершения, и время, когда процессы принудительно останавливаются с сигналом уничтожения. Установите это значение больше, чем ожидаемое время очистки для вашего процесса. По умолчанию 30 секунд.*

Добавьте `terminationGracePeriodSeconds: 30` в свой файл `values.yaml`, чтобы ваши строки 5–12 выглядели так, как показано ниже:

```
nano ./myhelm1/values.yaml

replicaCount: 1

terminationGracePeriodSeconds: 30

image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent
```

Отредактируйте файл развертывания, чтобы он использовал это новое значение (строки с 22 по 29 должны быть такими, как показано ниже)

```
nano ./myhelm1/templates/deployment.yaml

      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
          
          command: ['sh', '-c', 'sleep 60']
```

Сделайте пробный запуск.

```
helm install .\myhelm1\ --name test4 --dry-run --debug
[debug] Created tunnel using local port: '50239'

[debug] SERVER: "127.0.0.1:50239"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test4
REVISION: 1
RELEASED: Thu Feb 14 10:54:58 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
replicaCount: 1
terminationGracePeriodSeconds: 30     < - - - - - - -

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test4-myhelm1
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          terminationGracePeriodSeconds: 30     < - - - - - -

          command: ['sh', '-c', 'sleep 60']
```

Успешно. COMPUTED VALUES: показывает их правильно и развертывание внизу использует их правильно.

Еще один тест: давайте отладим тест, переопределив значение `terminationGracePeriodSeconds` на 10.

```
helm install .\myhelm1\ --set terminationGracePeriodSeconds=10 --name test4 --dry-run --debug
[debug] Created tunnel using local port: '50245'

[debug] SERVER: "127.0.0.1:50245"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test4
REVISION: 1
RELEASED: Thu Feb 14 10:56:33 2019
CHART: myhelm1-0.1.0

USER-SUPPLIED VALUES:
terminationGracePeriodSeconds: 10     < - - - - - -

COMPUTED VALUES:
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
replicaCount: 1
terminationGracePeriodSeconds: 10     < - - - - - -

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test4-myhelm1
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          terminationGracePeriodSeconds: 10     < - - - - - -

          command: ['sh', '-c', 'sleep 60']
```

Успешно. COMPUTED VALUES: правильно показывает `10` и при развертывании внизу правильно используется 10.

Мы даже не посмотрели на `_helpers.tpl` или каталог чартов. (Это касается зависимостей. Это тема для другого руководства из этого набора.)

Мы внесли несколько изменений в наш файл значений, а также в файл развертывания и увидели его результаты с помощью отладки и команд живой установки.

Вы также можете скрыть ненужные файлы с чарта. (`.helmignore`)

На работе вы создадите свои собственные скелетные базовые чарты, из которых будете копировать.

Мы изучили базовые концепции Helm в первый же день, изменив чарт `nginx` в соответствии с нашими требованиями. `--dry-run --debug` - лучшая функция Helm: пробный запуск и отладка перед установкой.

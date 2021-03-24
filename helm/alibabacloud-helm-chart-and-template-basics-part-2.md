https://habr.com/ru/post/548720/

Основы работы с Helm чартами и темплейтами — Часть 2

В этом руководстве мы кратко обсудим, как Helm может помочь упростить управление приложениями Kubernetes, и узнаем, как использовать Helm для создания базового чарта.

В этом руководстве объясняется, как шаблон Helm `deployment.yaml` преобразуется из шаблона в манифест YAML.

Статья не соответствует традиционному справочному руководству по программированию: операторы и функции перечислены в алфавитном порядке.

Статья объясняет синтаксис шаблона по мере необходимости, чтобы объяснить `deployment.yaml` от начала до конца.

### Использование значений в шаблонах

Вот полный файл deployment.yaml для справки:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myhelm1.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "myhelm1.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "myhelm1.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
          
          command: ['sh', '-c', 'sleep 60']
                    
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
    {{- end }}
    {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
    {{- end }}
```

Выдержка из этого `deployment.yaml`:

```
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  replicas: {{ .Values.replicaCount }}
      app.kubernetes.io/instance: {{ .Release.Name }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

`.Release.Name` и `.Release.Service` встроены в объекты - полный список на https://docs.helm.sh/chart_template_guide/ Встроенные объекты. К сожалению, нет ссылок на определенные части их ОЧЕНЬ длинных веб-страниц - вам нужно прокрутить вниз и найти.

Ниже показано, как отображается финальный файл `deployment.yaml`, если вы используете:

```
helm install .\myhelm1\ --name test5 --dry-run --debug
```

В первой половине руководства вам не надо выполнять никаких команд, просто прочтите. (Даже при отладке выводится слишком много, если все, что нам нужно, - это исследовать синтаксис трехстрочного шаблона.)

(После того, как вы прочитали полный учебник, вы можете захотеть прочитать его еще раз, но на этот раз отредактировав файлы шаблонов и значений и запустив helm install с отладкой для воспроизведения / обучения)

```
    app.kubernetes.io/instance: test5
    app.kubernetes.io/managed-by: Tiller
      app.kubernetes.io/instance: test5
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent
```

Эти 3 значения ниже взяты из файла values.yaml:

```
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

Извлекаем `values.yaml`

```
image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent
```

**- name: {{ .Chart.Name }}** из файла Chart.yaml. Его содержимое показано ниже.

```
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for Kubernetes
name: myhelm1
version: 0.1.0
```

Вероятно, половина всех шаблонов - это как раз такие простые замены значений.

Вы видели, что в файлах YAML есть директивы шаблонов, встроенные в `{{ and }}`.

У вас должно быть пустое пространство после открытия `{{ и пустое пространство перед закрытием }}`.

Значения, которые передаются в шаблон, можно рассматривать как объекты с пространством имен, где точка `(.)`  разделяет каждый элемент с пространством имен.

Первая точка перед чартом указывает на то, что мы начинаем с самого верхнего пространства имен. Прочтите `.Chart.name` как «начните с верхнего пространства имен, найдите объект `Chart`, затем посмотрите внутри него на предмет с именем `name`».

### With

```
**values.yaml** extract: 

nodeSelector:
    disktype: ssd
    gpu: Nvidia
```

```
**deployment.yaml** extract: 

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Отображается как:

```
      nodeSelector:
        disktype: ssd
        gpu: Nvidia
```

**ОЧЕНЬ важно: пробелы имеют синтаксическое значение в YAML.**

Перед визуализированными двумя селекторами стоит 8 пробелов, поскольку в `deployment.yaml` есть `{{- toYaml. | nindent 8}}`

`nindent 8` делает отступ на 8 пробелов.

`with` - конструкция цикла. Со значениями в `.Values.nodeSelector`: преобразуйте его в Yaml (`toYaml`).
Точка после `toYaml` - это текущее значение `.Values.nodeSelector` в цикле. Это должно быть там.
Считайте его похожим на `sum(1,34,454)` ... как `toYaml(.)` ... это значение переданного параметра.

[Символ |](https://en.wikipedia.org/wiki/Pipeline_(Unix)?spm=a2c65.11461447.0.0.524e1b6aycQLfv) работает так же, если вы знакомы с оболочкой Linux.

`affinity`: и `tolerations`: `with` работают точно так же.

К сожалению, эти примеры не показывают, как with также является текущим модификатором области видимости. Это хорошо объясняется в разделе [MODIFYING SCOPE USING WITH](https://docs.helm.sh/chart_template_guide/?spm=a2c65.11461447.0.0.524e1b6aycQLfv)

За исключением `include`, вы теперь полностью понимаете, как выполняется рендеринг всего `deployment.yaml` с использованием значений из `values.yaml`, `Chart.yaml` и встроенных объектов.

Полный `service.yaml` ниже:

Теперь вы тоже это полностью понимаете.

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myhelm1.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

### Переменные и range (диапазон) Helm

Извлечение первых и последних 3 строк из `ingress.yaml`

```
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "myhelm1.fullname" . -}}
{{- $ingressPaths := .Values.ingress.paths -}}
... rest of yaml ....
    {{- end }}
  {{- end }}
{{- end }}
```

```
**values.yaml** extract: 
ingress:
  enabled: false
```

Все содержимое `ingress.yaml` заключено в большой `if` ..., начиная со строки 1 и заканчивая самой последней строкой. Если вход включен - false, содержимое yaml не создается - как мы этого хотим.

Строки 2 и 3 демонстрируют, как объявлять переменные шаблона Helm.

Обратите внимание на дефис в `{{ и }}`

Эти дефисы / тире съедают символы пробела. `{{-` съедает все пробелы слева

`-}}` означает, что пробелы справа должны быть удалены - включая новую строку - строка полностью удаляется.

Извлечение `values.yaml`

```
ingress:
  enabled: false
  hosts:
    - chart-example.local
  tls:
    - secretName: chart-example-tls
      hosts:
        - chart-example.local-1
        - chart-example.local-2
        - chart-example.local-3
```

Извлечение deployment.yaml :

```
{{- if .Values.ingress.tls }}
  tls:
  {{- range .Values.ingress.tls }}
    - hosts:
      {{- range .hosts }}
        - {{ . | quote }}
      {{- end }}
      secretName: {{ .secretName }}
  {{- end }}
{{- end }}
```

Отображается как:

```
spec:
  tls:
    - hosts:
        - "chart-example.local-1"
        - "chart-example.local-2"
        - "chart-example.local-3"
      secretName: chart-example-tls
```

Обратите внимание, как цикл диапазона (`range`) генерирует список хостов. `quote` окружает каждого хоста кавычками.

Также существует цикл `range .Values.ingress.tls`, который выполняется только один раз. Присвоение этому циклу 3 значений продемонстрирует, как он будет колебаться в пределах значений.

```
Extract of **values.yaml** 

ingress:
  enabled: false
  hosts:
    - chart-example.local
  tls:
    - secretName: chart-example-tls-a
      hosts:
        - chart-example.local-1-a
        - chart-example.local-2-a
        - chart-example.local-3-a

    - secretName: chart-example-tls-b
      hosts:
        - chart-example.local-1-b
        - chart-example.local-2-b

    - secretName: chart-example-tls-c
      hosts:
        - chart-example.local-1-c
        - chart-example.local-2-c
        - chart-example.local-3-c
        - chart-example.local-4-c
```

Отображается как:

```
  tls:
    - hosts:
        - "chart-example.local-1-a"
        - "chart-example.local-2-a"
        - "chart-example.local-3-a"
      secretName: chart-example-tls-a
    - hosts:
        - "chart-example.local-1-b"
        - "chart-example.local-2-b"
      secretName: chart-example-tls-b
    - hosts:
        - "chart-example.local-1-c"
        - "chart-example.local-2-c"
        - "chart-example.local-3-c"
        - "chart-example.local-4-c"
      secretName: chart-example-tls-c
```

### importance of -

Оригинальный шаблон с дефисами.

```
{{- if .Values.ingress.tls }}
  tls:
  {{- range .Values.ingress.tls }}
    - hosts:
      {{- range .hosts }}
        - {{ . | quote }}
      {{- end }}
      secretName: {{ .secretName }}
  {{- end }}
{{- end }}
```

шаблон с удаленными дефисами:

```
{{ if .Values.ingress.tls }}
  tls:
  {{ range .Values.ingress.tls }}
    - hosts:
      {{ range .hosts }}
        - {{ . | quote }}
      {{ end }}
      secretName: {{ .secretName }}
  {{ end }}
{{ end }}
```

Отображается как:

```
  tls:
    - hosts:

        - "chart-example.local-1-a"

        - "chart-example.local-2-a"

        - "chart-example.local-3-a"

      secretName: chart-example-tls-a
      
    - hosts:

        - "chart-example.local-1-b"

        - "chart-example.local-2-b"

      secretName: chart-example-tls-b
      
    - hosts:

        - "chart-example.local-1-c"

        - "chart-example.local-2-c"

        - "chart-example.local-3-c"

        - "chart-example.local-4-c"

      secretName: chart-example-tls-c
      
```

У вас должны быть дефисы на конце строки и пробела.

### _helpers.tpl

Теперь мы понимаем, как значения используются для создания всех наших шаблонов.

Одно исключение: `name: {{include "myhelm1.fullname". }}` - включение.

Теперь мы исследуем `include`

Мы используем include для включения других шаблонов в наши YAML-шаблоны.

Файл `_helpers.tpl` - это стандартный способ определения нескольких коротких фрагментов шаблона, которые мы хотим включить в другие шаблоны.

Вот первый именованный шаблон в `_helpers.tpl:`

```
{{- define "myhelm1.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

В первой строке мы даем имя фрагменту шаблона. Затем мы ссылаемся на это имя, чтобы включить его.

Вторая строка дает myhelm1.name значение по умолчанию: .Chart.Name.

Если значение по умолчанию не существует, myhelm1.name получает значение `.Values.nameOverride`.

`trunc 63` обрезает его до 63 символов.

`trimSuffix "-"` удаляет ОДИН завершающий `-` если он существует.
но

`trimSuffix "-`" удаляет только два завершающих `-` если они есть.

(Это не работает, как в некоторых языках программирования, где обрезка удаляет все завершающие символы)

`app.kubernetes.io/name: {{ include "myhelm1.name" . }}`

рендерится как

`app.kubernetes.io/name: myhelm1`

Далее: код шаблона

`helm.sh/chart: {{ include "myhelm1.chart" . }}`

рендерится как

`helm.sh/chart: myhelm1-0.1.0`

Это функция шаблона:

```
{{- define "myhelm1.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

`printf "%s-%s" .Chart.Name .Chart.Version` объединяет `.Chart.Name` и `.Chart.Version` - плюс ставит дефис между ними.

`replace "+" "_"` заменяет символы плюса на символы подчеркивания.

Теперь, когда вы понимаете эти две однострочные функции, вы должны легко понять 10-строчное определение `myhelm1.fullname`.

Если у вас есть опыт программирования, вы увидите, что if / else работает должным образом:

```
if condition
 do something
else
 do something else
end
```

Единственное отличие - это синтаксис шаблона `{{ и }}`.

### Быстрое изучение синтаксиса шаблона Helm

Официальная документация Helm содержит подробную справочную информацию о чартах и шаблонах.

The Chart Developer's Guide: https://helm.sh/docs/topics/charts/

The Chart Template Developer's Guide: https://docs.helm.sh/chart_template_guide/

Чтобы полностью изучить всю информацию – понадобится не меньше дня . Лучший способ изучить всю информацию - интерактивное использование.

В этой части руководства объясняется, как изучить Helm в интерактивном режиме.

Изучение включает:

•	редактировать как можно меньше файлов
•	показывать только как можно меньше визуализированных строк шаблона
•	не делайте живых установок. Отлаживайте пробные запуски

Давайте как можно быстрее сконвертируем наши текущие файлы диаграмм в это - помните, что это некрасиво, а взлом - БЫСТРЫЙ.

Отредактируйте файл `values.yaml`, чтобы он выглядел так, как показано ниже:

```
replicaCount: 1

terminationGracePeriodSeconds: 30

image:
  repository: radial/busyboxplus
  tag: base
  pullPolicy: IfNotPresent
```

Убедитесь, что `./myhelm1/.helmignore` содержит эти строки, показанные ниже:

```
NOTES.txt
test-connection.yaml
service.yaml
ingress.yaml
```

Сделайте содержимое deployment.yaml, как показано ниже:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
```

ВСЕ, что нам нужно, это yaml (даже НЕПРАВИЛЬНЫЙ), чтобы его запустить.

Сделайте пробный запуск:

```
helm install .\myhelm1\  --name test5 --dry-run --debug
```

Получается слишком длинный вывод, как показано ниже:

```
PS C:\k8> helm install .\myhelm1\  --name test5 --dry-run --debug
[debug] Created tunnel using local port: '50327'

[debug] SERVER: "127.0.0.1:50327"

[debug] Original chart version: ""
[debug] CHART PATH: C:\k8\myhelm1

NAME:   test5
REVISION: 1
RELEASED: Fri Feb 15 13:47:49 2019
CHART: myhelm1-0.1.0
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
replicaCount: 1
terminationGracePeriodSeconds: 30

HOOKS:
MANIFEST:

---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test5
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          terminationGracePeriodSeconds: 30
```

Избавьтесь от первых нескольких «бесполезных» строк с помощью grep.

```
helm install .\myhelm1\  --name test5 --dry-run --debug | grep -vE 'debug]|NAME|REVIS|RELEA|ART:|OKS:|FEST:'
```

Смотрите что ниже. Это все, что нам нужно: некоторые значения для игры и некоторый контент шаблона yaml для игры с синтаксисом шаблона.

```
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
image:
  pullPolicy: IfNotPresent
  repository: radial/busyboxplus
  tag: base
replicaCount: 1
terminationGracePeriodSeconds: 30


---
# Source: myhelm1/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myhelm1
    helm.sh/chart: myhelm1-0.1.0
    app.kubernetes.io/instance: test5
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myhelm1
          image: "radial/busyboxplus:base"
          imagePullPolicy: IfNotPresent

          terminationGracePeriodSeconds: 30
```

А теперь займитесь изучением синтаксиса:
Смотрите ниже `deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec: #-------------------->> learn spacing << ------------------------
  replicas1: {{ .Values.replicaCount }}
  replicas2:   {{ .Values.replicaCount }}
  replicas3:    {{ .Values.replicaCount }}
  replicas4: '{{ .Values.replicaCount }}'
  replicas5: "{{ .Values.replicaCount }}"
  replicas6: "{{    .Values.replicaCount }}"
  replicas7: "{{    .Values.replicaCount       }}"
  replicas: "{{    .Values.replicaCount       }}'
  replicas: '{{    .Values.replicaCount       }}"
  replicas: {{    .Values.replicaCount       }}"
  replicas: "{{    .Values.replicaCount       }}

  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image1: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          image2: "{{ .Values.image.repository }} {{ .Values.image.tag }}"
          image3: "{{ .Values.image.repository }}{{ .Values.image.tag }}"
          image4: {{ .Values.image.repository }}{{ .Values.image.tag }}
          
          imagePullPolicy1: {{ .Values.image.pullPolicy }}
          imagePullPolicy2: {{ .Values.image.pullPolicyzzz }}
          imagePullPolicy3: {{ .Values.image.pullPolicyeeeeeeeeeee }}
          
          terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
```

На выходе:

```
spec: #-------------------->> learn spacing << ------------------------
  replicas1: 1
  replicas2:   1
  replicas3:    1
  replicas4: '1'
  replicas5: "1"
  replicas6: "1"
  replicas7: "1"
  template:
    spec:
      containers:
        - name: myhelm1
          image1: "radial/busyboxplus:base"
          image2: "radial/busyboxplus base"
          image3: "radial/busyboxplusbase"
          image4: radial/busyboxplusbase

          imagePullPolicy1: IfNotPresent
          imagePullPolicy2:
          imagePullPolicy3:

          terminationGracePeriodSeconds: 30
```

Посмотрите, сколько синтаксиса вы выучили за секунды.

Делайте ошибки, чтобы понять, что на самом деле означают ошибки. Вы также узнаете, когда сообщение об ошибке помогает, а когда вводит в заблуждение.

Отредактируйте `deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec: #
  replicas1: {{ .Values.replicaCount }}

  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image1: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          
          imagePullPolicy-correct: {{ .Values.image.pullPolicy }}

          imagePullPolicy1: {{ Values.image.pullPolicy }}
          imagePullPolicy2: {{ .Valu.image.pullPolicyzzz }}
          imagePullPolicy3: {{ ..Values.image.pullPolicyeeeeeeeeeee }}
```

Сделайте пробный запуск:

```
helm install .\myhelm1\  --name test5 --dry-run --debug | grep -vE 'debug]|NAME|REVIS|RELEA|ART:|OKS:|FEST:'
```

```
Error: parse error in "myhelm1/templates/deployment.yaml": template: myhelm1/templates/deployment.yaml:19: function "Values" not defined
```

ПРОЧИТАЙТЕ, поймите и исправьте ошибку, отправьте повторно.

```
Error: parse error in "myhelm1/templates/deployment.yaml": template: myhelm1/templates/deployment.yaml:21: unexpected . after term "."
```

ПРОЧИТАЙТЕ, поймите и исправьте ошибку, отправьте повторно.

```
Error: render error in "myhelm1/templates/deployment.yaml": template: myhelm1/templates/deployment.yaml:20:36: executing "myhelm1/templates/deployment.yaml" at 
<.Valu.image.pullPoli...>: can't evaluate field image in type interface {}
```

```
helm install .\myhelm1\  --name test5 --dry-run --debug | grep -vE 'debug]|NAME|REVIS|RELEA|ART:|OKS:|FEST:'
```

Ниже приведены несколько различных синтаксических экспериментов:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec: #
  replicas1: {{ .Values.replicaCount }}

  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image1: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          
          imagePullPolicy1: {{ quote .Values.image.pullPolicy }}

          imagePullPolicy2: {{ .Values.image.pullPolicy | quote }}

          imagePullPolicy3: "{{ .Values.image.pullPolicy }}"

          imagePullPolicy4: {{ .Values.image.pullPolicy | upper }}

          imagePullPolicy5: {{ .Values.image.pullPolicy | lower }}

{{ $variable := 123 }}

          variable: $variable           
          variable: {{ $variable }}
```

См. Дополнительные 3 строки внизу - use those -символы, чтобы удалить их. Удалите все 3 строки.

Helm не такой интерактивный, как Python, но таким образом вы почти можете это сделать.

Отображается как:

```
      containers:
        - name: myhelm1
          image1: "radial/busyboxplus:base"

          imagePullPolicy1: "IfNotPresent"

          imagePullPolicy2: "IfNotPresent"

          imagePullPolicy3: "IfNotPresent"

          imagePullPolicy4: IFNOTPRESENT

          imagePullPolicy5: ifnotpresent



          variable: $variable
          variable: 123     
```

Еще одна уловка. Смотрите, imagePullPolicy с 1 по 3 выглядит одинаково. Что мы сделали? Вы можете заменить уродливые названия вот так:

`deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myhelm1.name" . }}
    helm.sh/chart: {{ include "myhelm1.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec: #
  replicas1: {{ .Values.replicaCount }}

  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image1: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          
#          imagePullPolicy1: {{ quote .Values.image.pullPolicy }}
          imagePullPolicy1: {{ quote .Values.image.pullPolicy }}

#          imagePullPolicy2: {{ .Values.image.pullPolicy | quote }}
          imagePullPolicy2: {{ .Values.image.pullPolicy | quote }}

          imagePullPolicy3: " .Values.image.pullPolicy "
          imagePullPolicy3: "{{ .Values.image.pullPolicy }}"

          imagePullPolicy4:  .Values.image.pullPolicy | upper 
          imagePullPolicy4: {{ .Values.image.pullPolicy | upper }}

          imagePullPolicy5:  .Values.image.pullPolicy | lower 
          imagePullPolicy5: {{ .Values.image.pullPolicy | lower }}

{{ $variable := 123 }}

          variable: $variable           
          variable: {{ $variable }}
```

```
helm install .\myhelm1\  --name test5 --dry-run --debug | grep -vE 'debug]|NAME|REVIS|RELEA|ART:|OKS:|FEST:'
```

На выходе получается:

```
        - name: myhelm1
          image1: "radial/busyboxplus:base"

#          imagePullPolicy1: "IfNotPresent"
          imagePullPolicy1: "IfNotPresent"

#          imagePullPolicy2: "IfNotPresent"
          imagePullPolicy2: "IfNotPresent"

          imagePullPolicy3: " .Values.image.pullPolicy "
          imagePullPolicy3: "IfNotPresent"

          imagePullPolicy4:  .Values.image.pullPolicy | upper
          imagePullPolicy4: IFNOTPRESENT

          imagePullPolicy5:  .Values.image.pullPolicy | lower
          imagePullPolicy5: ifnotpresent
```

Комментарии к заметкам не помогают. Интерпретируется синтаксис шаблона внутри комментариев.

Удаление `{{ and }}` всегда работает. Затем вы можете просмотреть исходный код шаблона и результат в следующей строке выходных данных.

Лучший способ показать заголовки того, что было протестировано, - это использовать `{ { and { }` вместо `{{ and }}`.. политики 1 ниже.

`{ and }` также работает в заголовках и очень похож на синтаксис чтения `{{ and }}`

```
          imagePullPolicy1: { { quote .Values.image.pullPolicy } }
          imagePullPolicy1: {{ quote .Values.image.pullPolicy }}

          imagePullPolicy2: { .Values.image.pullPolicy | quote }
          imagePullPolicy2: {{ .Values.image.pullPolicy | quote }}
```

Отобразится как:

```
          imagePullPolicy1: { { quote .Values.image.pullPolicy } }
          imagePullPolicy1: "IfNotPresent"

          imagePullPolicy2: { .Values.image.pullPolicy | quote }
          imagePullPolicy2: "IfNotPresent"
```

```
helm install .\myhelm1\ --set replicaCount={1,2,3}  --name test5 --dry-run --debug | grep -vE 'debug]|NAME|REVIS|RELEA|ART:|OKS:|FEST:'
```

### Обучение на других шаблонах

В официальном репозитории Helm по адресу https://github.com/helm/charts есть почти 300 превосходных примеров диаграмм и шаблонов Helm.

Вы хотите учиться у лучших: у этих людей.

Вы увидите, что всего после этих двух руководств вы уже сможете понять более 80% всего кодирования шаблонов. (Но эти 2 руководства охватывают примерно 10 процентов синтаксиса).

Теперь у вас есть 4 независимых разных способа изучения чартов и шаблонов:

•	официальные справочные документы Helm
•	эти 2 практических урока
•	300 превосходных примеров диаграмм Helm
•	метод быстрого взлома, который вы можете использовать для вырезания и быстрого изучения этих 3 источников с помощью быстрых интерактивных упражнений

Справочная документация Helm сосредоточена на деталях на низком уровне.

Ищите идеи для проектирования структурных диаграмм, просматривая репозиторий диаграмм Helm.

Из https://github.com/helm/charts/blob/master/stable/lamp/templates/NOTES.txt

Посмотрите, как на чарте LAMP отображается справочный текст только для определенных примечаний `.Values.ingress.enabled`

```
{{- if .Values.ingress.enabled }}

INGRESS:
      Please make sure that you have an ingress controller instance {{ if .Values.ingress.ssl }}and a lego instance
      {{- end -}} running
      and that you have configured the A Records of {{ template "lamp.domain" . }} and its
      subdomains to point to your ingress controllers ip address.
{{- else }}
```

Большинство других чартов используют эту идею, поскольку чарты бесконечно гибки: многие функции можно включить или отключить.

Другой пример использования отображения NOTES.txt в зависимости от того, что активировал пользователь:  https://github.com/helm/charts/blob/master/stable/lamp/templates/NOTES.txt

```
1. You can now connect to the following services:

      {{- if not .Values.ingress.enabled }}
      export CHARTIP=$(kubectl get svc {{ template "lamp.fullname" . }} --output=jsonpath={.status.loadBalancer.ingress..ip})
      {{- end }}

      Main Site:
        {{- if .Values.ingress.enabled }}
        http{{ if .Values.ingress.ssl }}s{{ end }}://{{ template "lamp.domain" . }}
        {{- else }}
        http://$CHARTIP
        {{- end }}
      {{- if .Values.phpmyadmin.enabled }}

      PHPMyAdmin:
      {{- if .Values.ingress.enabled }}
        http{{ if .Values.ingress.ssl }}s{{ end }}://{{ .Values.phpmyadmin.subdomain }}.{{ template "lamp.domain" . }}
      {{- else }}
        http://$CHARTIP:{{ .Values.phpmyadmin.port }}
      {{- end }}
      {{- end }}
```

Другой часто используемый метод - предупредить пользователей, если чарт настроен неправильно или небезопасно: https://github.com/helm/charts/blob/master/stable/mongodb/templates/NOTES.txt

```
{{- if contains .Values.service.type "LoadBalancer" }}
{{- if not .Values.mongodbRootPassword }}
-------------------------------------------------------------------------------
 WARNING

    By specifying "service.type=LoadBalancer" and not specifying "mongodbRootPassword"
    you have most  likely exposed the MongoDB service externally without any
    authentication mechanism.

    For security reasons, we strongly suggest that you switch to "ClusterIP" or
    "NodePort". As alternative, you can also specify a valid password on the
    "mongodbRootPassword" parameter.

-------------------------------------------------------------------------------
{{- end }}
{{- end }}
```

Прекрасный пример того, как обращаться с `.Values.service.type` "`NodePort`", "`LoadBalancer`" или "ClusterIP" : https://github.com/helm/charts/blob/master/stable/mongodb/templates/NOTES.txt

```
To connect to your database from outside the cluster execute the following commands:

{{- if contains "NodePort" .Values.service.type }}

    export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
    export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ template "mongodb.fullname" . }})
    mongo --host $NODE_IP --port $NODE_PORT {{- if .Values.usePassword }} --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD{{- end }}

{{- else if contains "LoadBalancer" .Values.service.type }}

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace {{ .Release.Namespace }} -w {{ template "mongodb.fullname" . }}'

    export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ template "mongodb.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
    mongo --host $SERVICE_IP --port {{ .Values.service.nodePort }} {{- if .Values.usePassword }} --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD{{- end }}

{{- else if contains "ClusterIP" .Values.service.type }}

    kubectl port-forward --namespace {{ .Release.Namespace }} svc/{{ template "mongodb.fullname" . }} 27017:27017 &
    mongo --host 127.0.0.1 {{- if .Values.usePassword }} --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD{{- end }}

{{- end }}
```

Благодаря этим примерам вы потратили менее 3 минут на изучение некоторых примеров чартов. Вы можете спокойно провести там день (и оно того стоит ... если вы будете делать заметки).

### Как использовать .Files.Get

На https://docs.helm.sh/chart_template_guide/ ... Доступ к файлам внутри шаблонов ... (нельзя напрямую ссылаться на этот абзац) есть несколько примеров того, как включать файлы в шаблоны.
В репозитории Helm есть 80 примеров использования `.Files.Get.`
https://github.com/helm/charts/search?utf8=%E2%9C%93&q=.Files.Get&type=
В первых 10 результатах я обнаружил 5 различных вариантов использования `.Files.Get.`
Чтобы узнать больше о Helm, посетите https://github.com/helm/charts/tree/master/stable.

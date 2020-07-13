Skydive — это анализатор топологии сети и протоколов с открытым исходным кодом в реальном времени. Он направлен на то, чтобы предоставить исчерпывающий способ понять, что происходит в сетевой инфраструктуре.

Чтобы заинтересовать вас, приведу пару скриншотов про Skydive. Чуть ниже будет пост по введению в Skydive.

![](https://pbs.twimg.com/media/Dq_eDWdWsAAgBAx?format=jpg&name=large)

![](https://pbs.twimg.com/media/DjWnUGjX4AAmBNa?format=png&name=large)

<cut />

Пост «[Введение в skydive.network]( https://habr.com/ru/post/472724/)» на Хабре.

Skydive отображает топологию сети, получая сетевые события от агентов Skydive. Вы когда-нибудь задавались вопросом, как добавить или отобразить на диаграмме топологии сетевые компоненты, которые находятся вне агентской сети Skydive или не сетевые объекты, такие как TOR, хранилище данных и т. д. Больше не стоит беспокоиться об этом, благодаря Node rule API. 

Начиная с версии 0.20, Skydive предоставляет Node rule API, которое может использоваться для создания новых узлов и ребер и для обновления метаданных существующих узлов. Node rule API разделены на два API-интерфейса: API-интерфейс правила узла и API-интерфейс правила ребра. API правила узла используется для создания нового узла и обновления метаданных существующего узла. API правила ребра используется для создания границы между двумя узлами, т.е. связывает два узла.

В этом блоге мы увидим два варианта использования, один из которых является сетевым компонентом,который не является частью сети skydive. Второй вариант - несетевой компонент. Перед этим мы рассмотрим некоторые основные способы использования API правил топологии.

### Создание узла Skydive

Чтобы создать узел, необходимо указать уникальное имя узла и допустимый тип узла. Вы также можете предоставить некоторые дополнительные параметры.

```
skydive client node-rule create --action="create" --node-name="node1" --node-type="fabric" --name="node rule1"
{
  "UUID": "ea21c30f-cfaa-4f2d-693d-95159acb71ed",
  "Name": "node rule1",
  "Description": "",
  "Metadata": {
    "Name": "node1",
    "Type": "fabric"
  },
  "Action": "create",
  "Query": ""
}
```

### Обновление Метаданных Узлов Skydive

Чтобы обновить метаданные существующего узла, необходимо предоставить запрос на языке gremlin для выбора узлов, в которых вы хотите обновить метаданные. В соответствии с вашим запросом вы можете обновить метаданные одного или нескольких узлов с помощью правила одного узла.

```
skydive client node-rule create --action="update" --name="update rule" --query="G.V().Has('Name', 'node1')" --metadata="key1=val1, key2=val2"
{
  "UUID": "3e6c0e15-a863-4583-6345-715053ac47ce",
  "Name": "update rule",
  "Description": "",
  "Metadata": {
    "key1": "val1",
    "key2": "val2"
  },
  "Action": "update",
  "Query": "G.V().Has('Name', 'node1')"
}
```

### Создание ребра Skydive

Для создания ребра необходимо указать исходный и конечный узлы и тип связи ребра, для создания дочернего узла значение типа связи должно быть ownership аналогично для создания связи типа layer2 значение типа связи должно быть layer2. Вы можете создать более одной связи между двумя узлами, но тип связи должен быть разным.

```
skydive client edge-rule create --name="edge" --src="G.v().has('TID', '2f6f9b99-82ef-5507-76b6-cbab28bda9cb')" --dst="G.V().Has('TID', 'd6ec6e2f-362e-51e5-4bb5-6ade37c2ca5c')" --relationtype="both"
{
  "UUID": "50fec124-c6d0-40c7-42a3-2ed8d5fbd410",
  "Name": "edge",
  "Description": "",
  "Src": "G.v().has('TID', '2f6f9b99-82ef-5507-76b6-cbab28bda9cb')",
  "Dst": "G.V().Has('TID', 'd6ec6e2f-362e-51e5-4bb5-6ade37c2ca5c')",
  "Metadata": {
    "RelationType": "both"
  }
}
```

### Первый вариант использования

В этом случае мы рассмотрим, как показать несетевое устройство в топологии skydive. Давайте рассмотрим, что у нас есть хранилище данных, которое должно быть отображено в топологической диаграмме skydive с некоторыми полезными метаданными.

Нам просто нужно создать правило узла, чтобы добавить устройство в топологию. Мы можем добавить метаданные устройства как часть команды create или позже создать одну или несколько команд update node rule.

Выполните приведенную ниже команду правила узла, чтобы добавить запоминающее устройство в схему топологии.

```
skydive client node-rule create --action="create" --node-name="sda" --node-type="persistentvolume" --metadata="DEVNAME=/dev/sda,DEVTYPE=disk,ID.MODEL=SD_MMC, ID.MODEL ID=0316, ID.PATH TAG=pci-0000_00_14_0-usb-0_3_1_0-scsi-0_0_0_0, ID.SERIAL SHORT=20120501030900000, ID.VENDOR=Generic-, ID.VENDOR ID=0bda, MAJOR=8, MINOR=0, SUBSYSTEM=block, USEC_INITIALIZED=104393719727"
```

Выполните команду ниже правила ребра, чтобы связать созданный узел с узлом хоста.

```
skydive client edge-rule create --src="G.V().Has('Name', 'node1')" --dst="G.V().Has('Name', 'sda')" --relationtype="ownership"
```

После приведенных выше команд теперь вы можете увидеть устройство, видимое на схеме топологии skydive с заданными метаданными, как показано на рисунке ниже.

![](http://skydive.network/assets/images/blog/rules4.png)

### Второй вариант использования

В этом случае мы увидим, как добавить сетевое устройство,которое не является частью сети skydive. Давайте рассмотрим этот пример. У нас есть два агента skydive, работающих на двух разных хостах, для подключения этих двух хостов нам нужен коммутатор TOR. Даже если мы можем достичь этого, определив узлы структуры и ссылки в конфигурационном файле, давайте посмотрим, как мы можем сделать то же самое с помощью API правил топологии.

Без коммутатора TOR два агента будут выглядеть как два разных узла без каких-либо ссылок, как показано на рисунке ниже.

![](http://skydive.network/assets/images/blog/rules1.png)

Теперь выполните следующие команды Правила узла, чтобы создать коммутатор TOR и порты.

```
skydive client node-rule create --node-name="TOR" --node-type="fabric" --action="create"
skydive client node-rule create --node-name="port1" --node-type="port" --action="create"
skydive client node-rule create --node-name="port2" --node-type="port" --action="create"
```

Как видите, коммутатор TOR и порты созданы и добавлены в топологию skydive, и теперь топология будет выглядеть так, как показано на рисунке ниже.

![](http://skydive.network/assets/images/blog/rules2.png)

Теперь выполните приведенные ниже команды Edge Rule для создания связи между коммутатором TOR, портом 1 и публичным интерфейсом хоста 1.

```
skydive client edge-rule create --src="G.V().Has('Name', 'TOR')" --dst="G.V().Has('Name', 'port1')" --relationtype="ownership"
skydive client edge-rule create --src="G.V().Has('Name', 'TOR')" --dst="G.V().Has('Name', 'port1')" --relationtype="layer2"
skydive client edge-rule create --src="G.V().Has('TID', '372c254d-bac9-50c2-4ca9-86dcc6ce8a57')" --dst="G.V().Has('Name', 'port1')" --relationtype="layer2"
```

Выполните следующие команды, чтобы создать связь между коммутатором TOR, портом 2 и публичным интерфейсом хоста 2

```
skydive client edge-rule create --src="G.V().Has('Name', 'TOR')" --dst="G.V().Has('Name', 'port2')" --relationtype="layer2"
skydive client edge-rule create --src="G.V().Has('Name', 'TOR')" --dst="G.V().Has('Name', 'port2')" --relationtype="ownership"
skydive client edge-rule create --src="G.V().Has('TID', '50037073-7862-5234-4996-e58cc067c69c')" --dst="G.V().Has('Name', 'port2')" --relationtype="layer2"
```

Теперь создаются связи ownership и layer2 между коммутатором TOR и портом, а также связи layer2 между агентами и портами. Теперь окончательная топология будет выглядеть так, как показано на рисунке ниже.

![](http://skydive.network/assets/images/blog/rules3.png)

Теперь два хоста / агента связаны правильно, и вы можете проверить подключение или создать захват кратчайшего пути между двумя хостами.

P.S. Ссылка на [оригинал поста](http://skydive.network/blog/topology-rules.html)

Ищутся люди, которые могли бы писать посты о других возможностях Skydive.
[Телеграм-чат](https://t.me/skydive_network_ru) по skydive.network.

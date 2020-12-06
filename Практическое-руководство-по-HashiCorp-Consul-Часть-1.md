Практическое руководство по HashiCorp Consul — Часть 1
https://medium.com/velotio-perspectives/a-practical-guide-to-hashicorp-consul-part-1-5ee778a7fcf4

![](https://habrastorage.org/webt/r-/-d/a0/r--da0gbyafieifqbiwxezr48a4.jpeg)

Это часть 1 из серии 2 частей практического руководства по HashiCorp Consul. Эта часть в первую очередь ориентирована на понимание проблем, которые решает Consul и как он их решает. Вторая часть больше ориентирована на практическое применение Consul в реальном примере и будет опубликована на следующей неделе. Давайте начнем.

Как насчет настройки обнаруживаемого, настраиваемого и безопасного service mesh с помощью одного инструмента?

Что, если мы скажем вам, что этот инструмент не зависит от платформы и готов к работе в облаке?

И поставляется в виде одного бинарного файла.

Все это правда. Инструмент, о котором мы говорим, - это [HashiCorp Consul](https://www.consul.io/).

Consul обеспечивает [обнаружение сервисов](https://en.wikipedia.org/wiki/Service_discovery), [проверку работоспособности](https://microservices.io/patterns/observability/health-check-api.html), [балансировку нагрузки](https://en.wikipedia.org/wiki/Load_balancing_(computing)), [граф сервисов](https://medium.com/@marcus.cavalcanti/using-service-graphs-to-reduce-mttr-in-a-http-based-architecture-624d2c9d54c1), [принудительное использование идентификационных данных с помощью TLS](https://www.nevatech.com/blog/post/What-you-need-to-know-about-securing-APIs-with-mutual-certificates) и [управление конфигурацией распределенных сервисов](https://www.pluralsight.com/guides/role-of-configuration-management-in-devops).

Давайте подробнее познакомимся с Consul ниже и посмотрим, как он решает эти сложные задачи и облегчает жизнь оператору распределенной системы.

### Вступление

[Микросервисы](https://martinfowler.com/articles/microservices.html) и [другие распределенные системы](https://medium.com/microservices-learning/the-evolution-of-distributed-systems-fec4d35beffd) могут обеспечить более быструю и простую разработку программного обеспечения. Но есть компромисс, приводящий к большей сложности работы в области межсервисного взаимодействия, управления конфигурацией и сегментации сети.

![](https://habrastorage.org/webt/yv/ej/qz/yvejqzhsqrydqvh7ltyvej9h6fq.png)

*Монолитное приложение (репрезентативное) — с различными подсистемами A, B, C и D*

![](https://habrastorage.org/webt/7f/4u/j5/7f4uj5igx5ds4ci31dvygyjmswq.png)

*Распределенное приложение (репрезентативное) — с различными сервисами A, B, C и D*

[HashiCorp Consul – это инструмент с открытым исходным кодом](https://github.com/hashicorp/consul), который решает эти новые сложности, предоставляя обнаружение сервисов, проверку работоспособности, балансировку нагрузки, график сервисов, взаимное применение идентификации TLS и хранилище конфигурационных ключей. Эти особенности делают Consul идеальным control plane для [service mesh](https://www.nginx.com/blog/what-is-a-service-mesh/).

![](https://habrastorage.org/webt/6o/8w/dp/6o8wdplmguyscheyzmezf8zqqhu.png)

*HashiCorp Consul поддерживает обнаружение сервисов, конфигурацию сервисов и сегментацию сервисов*

[HashiCorp объявил Consul в апреле 2014 года](https://www.hashicorp.com/blog/consul-announcement), и с тех пор он получила хорошее признание сообщества.

Это руководство предназначено для обсуждения некоторых из этих важнейших проблем и изучения различных решений, предлагаемых Consul HashiCorp для решения этих проблем.

Давайте кратко рассмотрим темы, которые мы собираемся охватить в этом руководстве. Темы написаны так, чтобы быть самодостаточными. Вы можете перейти непосредственно к конкретной теме, если хотите.

### Краткая справка о Монолитных и Сервис-ориентированных архитектурах (SOA)

Рассматривая традиционные архитектуры [доставки приложений](https://www.nginx.com/resources/glossary/application-delivery/), мы находим [классический монолит](https://en.wikipedia.org/wiki/Monolithic_application). Когда мы говорим о Монолитных архитектурах, у нас есть [развертывание одного приложения](http://www.codingthearchitecture.com/2014/11/19/what_is_a_monolith.html).

Даже если это одно приложение, обычно оно имеет несколько различных субкомпонентов.

Один из примеров, который технический директор HashiCorp Армон Дадгар привел во время своего [вступительного видео для Consul](https://www.youtube.com/watch?v=mxeMdl0KvBI&t=), был о поставке настольного банковского приложения. Он имеет дискретный набор субкомпонентов-например, аутентификацию (скажем, подсистема а), управление счетами (подсистема в), перевод средств (подсистема С) и обмен валюты (подсистема D).

Теперь, хотя это независимые функции — аутентификация системы А против перевода средств системы С, — мы развертываем его как единое монолитное приложение.

За последние несколько лет мы наблюдаем [тенденцию отхода от такого рода архитектуры](https://medium.com/swlh/moving-away-from-monolithic-architecture-8a19def7f7f9). Есть несколько причин для этого сдвига.

Задача с монолитной архитектурой такова: предположим, что в одной из подсистем, системе A, есть ошибка, связанная с аутентификацией.

![](https://habrastorage.org/webt/d3/tp/b1/d3tpb1cnifenhuup2e376hmdtke.png)

*Репрезентативная ошибка в подсистеме А в нашем монолитном приложении*

Мы не можем просто исправить его в системе А и обновить его в производстве.

![](https://habrastorage.org/webt/r4/75/c2/r475c21szlngnl3hysbfmtzy-_i.png)

Исправление репрезентативной ошибки в подсистеме А в нашем монолитном приложении

Мы должны обновить систему А и выполнить повторное развертывание всего приложения, для чего нам также необходимо развертывание подсистем В, С и D.

![](https://habrastorage.org/webt/u7/o_/w7/u7o_w79kb26rqxpondnd0rc_l9g.png)

*Исправление ошибок в одной подсистеме приводит к перераспределению всего монолитного приложения*

Вся эта передислокация не идеальна. Вместо этого мы хотели бы сделать развертывание отдельных сервисов.

Одно и то же монолитное приложение поставляется в виде набора отдельных, дискретных сервисов.

![](https://habrastorage.org/webt/7f/4u/j5/7f4uj5igx5ds4ci31dvygyjmswq.png)

*Разделение монолитного приложения на отдельные сервисы*

Итак, если есть исправление ошибки в одном из наших сервисов:

![](https://habrastorage.org/webt/er/lu/7b/erlu7buheypao98dg9nae-szrlu.png)

*Репрезентативная ошибка в одном из сервисов, в данном случае сервисе А нашего SOA-приложения*

И мы исправляем эту ошибку:

![](https://habrastorage.org/webt/wh/vb/za/whvbza8dkvxyo9jytwwfklylbam.png)

*Исправление репрезентативной ошибки в сервисе A нашего SOA-приложения*

Мы можем осуществить передислокацию этого сервиса, не согласовывая ее развертывание с другими сервисами. По сути, речь идет об одной из форм микросервисов.

![](https://habrastorage.org/webt/gp/u7/km/gpu7kmph2hcpnr1mqhx2-duj2by.png)

*Исправление ошибки приведет к передеплою только сервиса А во всем нашем приложении*

Это дает большой толчок к нашей [гибкости разработки](https://jaxenter.com/most-important-benefit-microservices-is-agility-129472.html). Нам не нужно координировать наши усилия по разработке между различными командами разработчиков или даже системами. У нас будет свобода [разработки и деплоя независимо](https://www.nginx.com/blog/deploying-microservices/). Одна услуга предоставляется еженедельно, а другая-ежеквартально. Это будет большим преимуществом для команд разработчиков.

Но такого понятия, как бесплатный обед, не существует.

Эффективность развития, которую мы получили, вводит свой [собственный набор оперативных проблем](https://docs.aws.amazon.com/aws-technical-content/latest/microservices-on-aws/challenges-of-microservices.html). Давайте посмотрим на некоторые из них.

### Обнаружение сервиса в монолитной архитектуре, его проблемы в распределенной системе и решение Consul

#### Монолитные приложения

Предположим, что два сервиса в одном приложении хотят общаться друг с другом. Один из способов-раскрыть метод - сделать его общедоступным и разрешить другим сервисам вызывать его. В монолитном приложении это одно приложение, в котором сервисы будут предоставлять общедоступные функции посредством вызова функций между сервисами.

![](https://habrastorage.org/webt/sf/fp/9g/sffp9gc0_2jkpwirztap5wjnrm4.png)

*Подсистемы общаются друг с другом через вызов функции в нашем монолитном приложении*

Поскольку это вызов функции внутри процесса, он произошел в памяти. Это происходит быстро, поэтому нам не нужно беспокоиться о том, как были перемещены наши данные и были ли они безопасны.

#### Распределенные системы

В распределенном мире сервис A больше не поставляется как то же приложение, что и сервис B. Итак, как сервис A находит сервис B, если она хочет поговорить с B?

![](https://habrastorage.org/webt/ft/tv/2u/fttv2utldiku-kqshw1alwm7qwg.png)

*Сервис A пытается найти сервис В для установления связи*

Сервис A может даже не находиться на той же машине, что и сервис B. Таким образом, в поиске сервиса присутствует сеть. Но это происходит не так быстро, так как есть задержка, которую мы можем измерить на линиях миллисекунд, по сравнению с наносекундами простого вызова функции.

#### Проблемы в распределенных системах

Как мы уже знаем, две сервисы в распределенной системе должны обнаружить друг друга, чтобы взаимодействовать. Одним из традиционных способов решения этой проблемы является использование [балансировщиков нагрузки](https://www.citrix.com/glossary/load-balancing.html).

![](https://habrastorage.org/webt/js/ge/hb/jsgehbnlautw-bqr9sw7jr6fwwq.png)

*Балансировщик нагрузки находится между сервисами, чтобы они могли общаться друг с другом*

Балансировщики нагрузки будут сидеть перед каждым сервисом со статическим IP-адресом, известным всем другим сервисам.

![](https://habrastorage.org/webt/7s/in/hv/7sinhvboqheb7oibcf3sn_lgy6w.png)

*Балансировщик нагрузки между двумя сервисами позволяет осуществлять двусторонний трафик*

Это дает возможность добавить несколько экземпляров одним и том же сервисом за балансировщиком нагрузки, и он будет направлять трафик соответствующим образом. Но этот IP-адрес балансировщика нагрузки статичен и жестко закодирован во всех других сервисах, поэтому сервисы могут пропустить обнаружение.

![](https://habrastorage.org/webt/il/mr/wn/ilmrwng8ndsniy2ghkq44fphoza.png)

*Балансировщики нагрузки позволяют осуществлять связь между несколькими экземплярами одного и того же сервиса*

Теперь задача состоит в том, чтобы поддерживать набор балансировщиков нагрузки для каждого отдельного сервиса. И мы можем с уверенностью предположить, что изначально существовал балансировщик нагрузки и для всего приложения. Затраты и усилия на поддержание этих балансировщиков нагрузки возросли.

С балансировщиками нагрузки перед сервисами они представляют собой единую точку отказа. Даже если у нас есть несколько экземпляров сервисов за балансировщиком нагрузки, если он не работает, то наши сервисы не работает. Независимо от того, сколько экземпляров этих сервисов запущено.

Балансировщики нагрузки также [увеличивают](https://blog.buoyant.io/2016/03/16/beyond-round-robin-load-balancing-for-latency/) [задержку](https://en.wikipedia.org/wiki/Latency_(engineering)) межсервисного взаимодействия. Если сервисов A хочет поговорить со сервисом B, запрос от A должен сначала поговорить с балансировщиком нагрузки сервиса B, а затем достичь B. Ответ от B также должен будет пройти через тот же самый маршрут.

![](https://habrastorage.org/webt/th/s8/kg/ths8kgysqqthqiq5i8wd8bhvbug.png)

*Поддержание записи экземпляров сервисов в балансировщике нагрузки для всего приложения.*

И по своей природе [балансировщики нагрузки в большинстве случаев управляются вручную](https://dzone.com/articles/load-balancers-and-high-volume-traffic-management-1). Если мы добавим еще один экземпляр сервиса, он будет недоступен. Нам нужно будет зарегистрировать этот сервис в балансировщике нагрузки, чтобы сделать его доступной для всего мира. Это потребует ручных усилий и времени.

#### Решения Consul

Решение Consul для решения проблемы обнаружения сервисов в распределенных системах - это [центральный реестр сервисов](https://auth0.com/blog/an-introduction-to-microservices-part-3-the-service-registry/).

Consul ведет центральный реестр, содержащий записи для всех вышестоящих сервисов. При запуске экземпляра сервиса он регистрируется в центральном реестре. Реестр заполняется всеми вышестоящими экземплярами сервисов.

![](https://habrastorage.org/webt/if/zg/0u/ifzg0u53cfwybg-pc2o8eilxfxy.png)

*Реестр услуг Consul помогает сервису A найти сервис Б и установить связь*

Когда сервис A хочет поговорить со сервисом B, он обнаруживает и связывается с B, запрашивая реестр о вышестоящих экземплярах сервиса B. Таким образом, вместо взаимодействия с балансировщиком нагрузки сервис может напрямую связаться с желаемым конечным экземпляром сервиса.

Consul также обеспечивает проверку работоспособности (health-checks) этих экземпляров сервисов. Если один из экземпляров сервисов или сам сервисов не работает или не проходит проверку работоспособности (health-check), реестр будет знать об этом и будет избегать возврата адреса сервиса. Работа, которую будет выполнять балансировщик нагрузки, в этом случае выполняется реестром.

Кроме того, если существует несколько экземпляров одного и того же сервиса, Consul будет посылать трафик случайным образом в разные экземпляры. Таким образом, выравнивая нагрузку между различными экземплярами.

Consul справился с нашими задачами обнаружения сбоев и распределения нагрузки между несколькими экземплярами сервисов без необходимости развертывания централизованного балансировщика нагрузки.

Здесь решается традиционная проблема медленных и управляемых вручную балансировщиков нагрузки. Consul программно управляет реестром, который обновляется, когда любой новый сервис регистрируется и становится доступным для приема трафика.

Это помогает с легкостью масштабировать сервисы.

### Управление конфигурацией в монолите, его проблемы в распределенной среде и решение Consul

#### Монолитные Приложения

Когда мы смотрим на конфигурацию для монолитного приложения, они, как правило, находятся где-то на уровне гигантских файлов [YAML, XML или JSON](https://en.wikipedia.org/wiki/Configuration_file). Предполагается, что эта конфигурация настроит все приложение.

![](https://habrastorage.org/webt/k8/kk/yq/k8kkyqqaya4aztvwed4sxmfmol8.png)

*Единый конфигурационный файл, общий для различных частей нашего монолитного приложения*

Учитывая один файл, все наши подсистемы в нашем монолитном приложении теперь будут потреблять конфигурацию из одного и того же файла. Таким образом, создается согласованное представление обо всех наших подсистемах или сервисах.

Если мы хотим изменить состояние приложения с помощью обновления конфигурации, оно будет легко доступно для всех подсистем. Новая конфигурация одновременно потребляется всеми компонентами нашего приложения.

#### Распределенные системы

В отличие от монолитной системы, распределенные сервисы не будут иметь общего представления о конфигурации. Конфигурация теперь распределена, и там каждый отдельный сервис должна быть настроен отдельно.

![](https://habrastorage.org/webt/72/tq/cj/72tqcj5gw6bxf14od0aqkasn2ky.png)

*Копия конфигурации приложения распределяется между различными сервисами*

#### Проблемы в распределенных системах

- Конфигурация должна быть распределена между различными сервисами. Поддержание согласованности между конфигурациями в различных сервисах после каждого обновления является сложной задачей.

- Кроме того, проблема возрастает, когда мы ожидаем, что конфигурация будет обновляться динамически.


#### Решения Консула

Решение Consul для управления конфигурацией в распределенной среде - это центральное хранилище ключей и значений.

![](https://habrastorage.org/webt/yy/lf/c9/yylfc94cawk5t9c1fmbeqhv0ps0.png)

*Набор Consul’s KV дает возможность плавного отображения конфигурации на каждый сервис*

Consul решает эту задачу уникальным способом. Вместо того чтобы распределять конфигурацию между различными распределенными сервисами в виде частей конфигурации, он передает всю конфигурацию всем сервисами и динамически настраивает их в распределенной системе.

Возьмем пример изменения состояния в конфигурации. Измененное состояние передается через все сервисы в режиме реального времени. Конфигурация последовательно присутствует во всех сервисах.

### Сегментация сети в монолите, ее проблемы в распределенных системах и решения Consul

#### Монолитные приложения

При рассмотрении классической монолитной архитектуры, сеть обычно делится на три различные зоны.

**Первая зона** в нашей сети является общедоступной. Трафик, поступает в наше приложение через интернет и достигает наших балансировщиков нагрузки.

**Вторая зона** - это трафик от наших балансировщиков нагрузки к нашему приложению. В основном это внутренняя сетевая зона без прямого публичного доступа.

**Третья зона** - это закрытая сетевая зона, предназначенная в первую очередь для передачи данных. Это считается изолированной зоной.

![](https://habrastorage.org/webt/i_/go/a_/i_goa_fpbr5g_o2iqmeq85goe4e.png)

*Различные сетевые зоны в типичном приложении*

Только зона балансировщиков нагрузки может попасть в зону приложения, и только зона приложения может попасть в зону данных. Это простая система зонирования, простая в реализации и управлении.

#### Распределенные системы

Для распределенных сервисов картина кардинально меняется.

![](https://habrastorage.org/webt/gl/hz/1x/glhz1xpwm7wpnsesc7wsqhqibhk.png)

*Сложная структура сетевого трафика и маршрутов между различными сервисами*

В самой зоне нашей прикладной сети существует несколько сервисов. Каждый из этих сервисов взаимодействует с другими внутри этой сети, что делает ее сложной структурой трафика.

#### Проблемы в распределенных системах

- Основная проблема заключается в том, что [трафик не находится в каком-либо последовательном потоке](https://dzone.com/articles/eastwest-is-the-new-northsouth). В отличие от монолитной архитектуры, где поток был определен от балансировщиков нагрузки к приложению и от приложения к данным.

- В зависимости от модели доступа, которую мы хотим поддерживать, трафик может поступать с разных конечных точек и достигать разных сервисов.![](https://habrastorage.org/webt/jj/ir/yq/jjiryq-l8bx4fq3khh9gszj0kn0.png)

*Клиент по существу общается с каждым сервисным в приложением прямо или косвенно*

- Учитывая наличие нескольких сервисов и возможность [поддержки нескольких конечных точек](https://www.mulesoft.com/resources/api/microservices-and-apis), мы можем развернуть несколько потребителей и поставщиков сервисов.

- Из-за природы системы [безопасность является нашей следующей задачей](https://www.owasp.org/images/2/20/Microservice_Security.pdf). Сервисы должны быть в состоянии определить, что трафик, который они получают, исходит от [проверенного и доверенного объекта в сети](https://medium.com/tech-tajawal/microservice-authentication-and-authorization-solutions-e0e5e74b248a).


![](https://habrastorage.org/webt/9g/mi/zz/9gmizzm2vgackmfe3fkupxq2thy.png)

*SOA требует контроля над надежными и ненадежными источниками трафика*

- Управление потоком трафика и сегментирование сети на группы или блоки станет более серьезной проблемой. Кроме того, очень важно убедиться, что у нас есть строгие правила, которыми мы руководствуемся при [разделении сети](https://en.wikipedia.org/wiki/Network_partition) на основе того, кто должен иметь право взаимодействовать с кем и наоборот.


#### Решения консула

Решение Consul общей проблемы сегментации сети в распределенных системах заключается в реализации сервисных графов и [взаимного TLS](https://medium.com/sitewards/the-magic-of-tls-x509-and-mutual-authentication-explained-b2162dec4401).

![](https://habrastorage.org/webt/zq/n1/_h/zqn1_hhmipx_wy6zy3mn8hhmcbi.png)

*Применение политик уровня обслуживания для определения модели трафика и сегментации с помощью Consul*

Consul решает проблему сегментации сети, [централизованно управляя определением того, кто с кем может взаимодействовать](https://www.consul.io/docs/agent/acl-system.html). У Consul есть специальная функция для этого под названием [Consul Connect](https://learn.hashicorp.com/consul/getting-started/connect).

Consul Connect регистрирует эти политики межсервисного взаимодействия, которые мы желаем зарегистрировать, и реализует его в рамках графика обслуживания. Таким образом, политика может сказать, что сервис A может взаимодействовать со сервисом B, но B не может взаимодействовать с C.

Большее преимущество этого заключается в том, что он не ограничен IP-адресом. Скорее это уровень обслуживания. Это делает его масштабируемым. Эта политика будет применяться ко всем экземплярам сервиса, и не будет никакого жесткого правила брандмауэра, специфичного для IP-адреса сервиса. Это делает нас независимыми от масштаба нашей дистрибьюторской сети.

Consul Connect также обрабатывает идентификационные данные сервисов, используя популярный [протокол TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security). Он распространяет сертификат TLS, связанный со сервисом.

Эти сертификаты помогают другим сервисам надежно идентифицировать друг друга. TLS также помогает обеспечить безопасную связь между сервисами. Это обеспечивает надежную сетевую реализацию.

Consul применяет TLS с помощью [прокси на основе агента](https://www.consul.io/docs/connect/proxies.html), подключенного к каждому экземпляру сервиса. Этот прокси действует как [Sidecar](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar). Использование прокси-сервера в этом случае позволяет нам не вносить какие-либо изменения в код исходного сервиса.

Это позволяет получить преимущество более высокого уровня применения шифрования для данных в состоянии покоя и данных в процессе передачи. Кроме того, он будет способствовать соблюдению требований законодательства о конфиденциальности и идентификации пользователей.

### Базовая архитектура Consul

Consul - это распределенная и [высокодоступная система](https://en.wikipedia.org/wiki/High_availability).

Consul поставляется в виде одной бинарной загрузки для всех популярных платформ. Исполняемый файл может работать как клиент, так и сервер.

Каждый узел, предусматривающий службу Consul, запускает агента Consul. Каждый из этих агентов взаимодействует с одним или несколькими серверами Consul.

![](https://habrastorage.org/webt/hp/tx/r1/hptxr1vcepjax5pop9h4oijun5m.png)

Агент Consul отвечает за проверку работоспособности сервисов на узле, как и за проверку работоспособности самого узла. Он не несет ответственности за обнаружение сервисов или поддержание данных ключ/значение.

Серверы Consul - это место, где хранятся и реплицируются данные.

Consul может работать с одним сервером, но HashiCorp рекомендует запускать набор из 3-5 серверов, чтобы избежать сбоев. Поскольку все данные хранятся на стороне сервера Consul. С одним сервером, сбой может привести к потере данных.

В кластере с несколькими серверами они выбирают лидера между собой. HashiCorp также рекомендует иметь кластер серверов для каждого центра обработки данных.

Во время процесса обнаружения любой сервис в поисках других сервисов может запросить серверы Consul или даже агентов консула. Агенты Consul автоматически пересылают запросы на серверы консула.

![](https://habrastorage.org/webt/bs/yv/pg/bsyvpg67auykppnptt_emg2nola.png)

Агент Consul сидит на узле и общается с другими агентами в сети, синхронизируя всю информацию об уровне обслуживания

Если запрос является перекрестным центром обработки данных, то запросы пересылаются сервером Consul на удаленные серверы Consul. Результаты с удаленных серверов Consul возвращаются на исходный сервер Consul.

### Начало работы с Consul

Этот раздел посвящен пристальному рассмотрению Consul как инструмента, имеющего некоторый практический опыт.

#### Скачивание и Установка

Как обсуждалось выше, Consul поставляется в виде одного двоичного файла, загруженного с веб-сайта HashiCorps или из раздела релизов Consul на GitHub.

Один бинарный файл может работать как сервер Consul или даже как агент клиента Консула.

Вы можете скачать Consul отсюда — [Страница загрузки Consul](https://www.consul.io/downloads.html).

![](https://habrastorage.org/webt/ct/q5/j1/ctq5j1dtpss-bt0xqsg_u_ps8-0.png)

*Различные варианты загрузки Consul на разных операционных системах*

Мы скачаем Consul в командной строке по ссылке со страницы загрузки.

```
$ wget https://releases.hashicorp.com/consul/1.4.3/consul_1.4.3_linux_amd64.zip -O consul.zip

--2019-03-10 00:14:07--  https://releases.hashicorp.com/consul/1.4.3/consul_1.4.3_linux_amd64.zip
Resolving releases.hashicorp.com (releases.hashicorp.com)... 151.101.37.183, 2a04:4e42:9::439
Connecting to releases.hashicorp.com (releases.hashicorp.com)|151.101.37.183|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 34777003 (33M) [application/zip]
Saving to: ‘consul.zip’

consul.zip             100%[============================>]  33.17M  4.46MB/s    in 9.2s    

2019-03-10 00:14:17 (3.60 MB/s) - ‘consul.zip’ saved [34777003/34777003]
```

Распакуйте загруженный zip-файл.

```
$ unzip consul.zip

Archive:  consul.zip
  inflating: consul
```

Добавьте его в PATH.

```
$ export PATH="$PATH:/path/to/consul"
```

#### Использование Consul

Как только вы распакуете сжатый файл и поместите бинарный файл под свой путь, вы можете запустить его следующим образом.

```
$ consul agent -dev

==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.4.2'
           Node ID: 'ef46ebb7-3496-346f-f67a-30117cfec0ad'
         Node name: 'devcube'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2019/03/04 00:38:01 [DEBUG] agent: Using random ID "ef46ebb7-3496-346f-f67a-30117cfec0ad" as node ID
    2019/03/04 00:38:01 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:ef46ebb7-3496-346f-f67a-30117cfec0ad Address:127.0.0.1:8300}]
    2019/03/04 00:38:01 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2019/03/04 00:38:01 [INFO] serf: EventMemberJoin: devcube.dc1 127.0.0.1
    2019/03/04 00:38:01 [INFO] serf: EventMemberJoin: devcube 127.0.0.1
    2019/03/04 00:38:01 [INFO] consul: Adding LAN server devcube (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2019/03/04 00:38:01 [INFO] consul: Handled member-join event for server "devcube.dc1" in area "wan"
    2019/03/04 00:38:01 [DEBUG] agent/proxy: managed Connect proxy manager started
    2019/03/04 00:38:01 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2019/03/04 00:38:01 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2019/03/04 00:38:01 [DEBUG] raft: Votes needed: 1
    2019/03/04 00:38:01 [DEBUG] raft: Vote granted from ef46ebb7-3496-346f-f67a-30117cfec0ad in term 2. Tally: 1
    2019/03/04 00:38:01 [INFO] raft: Election won. Tally: 1
    2019/03/04 00:38:01 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2019/03/04 00:38:01 [INFO] consul: cluster leadership acquired
    2019/03/04 00:38:01 [INFO] consul: New leader elected: devcube
    2019/03/04 00:38:01 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2019/03/04 00:38:01 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2019/03/04 00:38:01 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
    2019/03/04 00:38:01 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
    2019/03/04 00:38:01 [INFO] agent: started state syncer
    2019/03/04 00:38:01 [INFO] connect: initialized primary datacenter CA with provider "consul"
    2019/03/04 00:38:01 [DEBUG] consul: Skipping self join check for "devcube" since the cluster is too small
    2019/03/04 00:38:01 [INFO] consul: member 'devcube' joined, marking health alive
    2019/03/04 00:38:01 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/03/04 00:38:01 [INFO] agent: Synced node info
    2019/03/04 00:38:01 [DEBUG] agent: Node info in sync
    2019/03/04 00:38:01 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/03/04 00:38:01 [DEBUG] agent: Node info in sync
```

Это позволит запустить агента в режиме разработки.

#### Участники Consul

Пока выполняется приведенная выше команда, вы можете проверить наличие всех участников в Сети Consul.

```
$ consul members

Node     Address         Status  Type    Build  Protocol  DC   Segment
devcube  127.0.0.1:8301  alive   server  1.4.0  2         dc1  <all>
```

Учитывая, что у нас работает только один узел, он по умолчанию рассматривается как сервер. Вы можете назначить агента в качестве сервера, указав сервер в качестве параметра командной строки или сервер в качестве параметра конфигурации в конфигурации Consul.

Выходные данные приведенной выше команды основаны на протоколе gossip и в конечном итоге согласованы.

#### Consul HTTP API

Для строго согласованного представления агентурной сети Consul мы можем использовать HTTP API, предоставленный консулом из коробки.

```
$ curl localhost:8500/v1/catalog/nodes

[
    {
        "ID": "ef46ebb7-3496-346f-f67a-30117cfec0ad",
        "Node": "devcube",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "Meta": {
            "consul-network-segment": ""
        },
        "CreateIndex": 9,
        "ModifyIndex": 10
    }
]
```

#### Интерфейс DNS Consul

Consul также предоставляет DNS-интерфейс для запросов узлов. По умолчанию он обслуживает DNS на порту 8600. Этот порт настраивается.

```
$ dig @127.0.0.1 -p 8600 devcube.node.consul

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> @127.0.0.1 -p 8600 devcube.node.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42215
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;devcube.node.consul.		IN	A

;; ANSWER SECTION:
devcube.node.consul.	0	IN	A	127.0.0.1

;; ADDITIONAL SECTION:
devcube.node.consul.	0	IN	TXT	"consul-network-segment="

;; Query time: 19 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Mar 04 00:45:44 IST 2019
;; MSG SIZE  rcvd: 100
```

Регистрация сервиса на Consul может быть достигнута либо путем написания определения Сервиса, либо путем отправки запроса через соответствующий HTTP API.

#### Определение сервиса Consul

Определение сервиса - это один из популярных способов регистрации сервиса. Давайте рассмотрим один из таких примеров определения сервиса.

Для размещения наших определений сервисов мы добавим каталог конфигурации, условно именуемый consul.d — ‘.d’ означает, что в этом каталоге имеется набор конфигурационных файлов, а не один конфиг под именем consul.

```
$ mkdir ./consul.d
```

Напишите определение сервиса для фиктивного веб-приложения Django, работающего на порту 80 на локальном хосте.

```
$ echo '{"service": {"name": "web", "tags": ["django"], "port": 80}}' \
    > ./consul.d/web.json
```

Чтобы наш агент-консул знал об этом определении сервиса, мы можем предоставить ему каталог конфигурации.

```
$ consul agent -dev -config-dir=./consul.d

==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.4.2'
           Node ID: '810f4804-dbce-03b1-056a-a81269ca90c1'
         Node name: 'devcube'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2019/03/04 00:55:28 [DEBUG] agent: Using random ID "810f4804-dbce-03b1-056a-a81269ca90c1" as node ID
    2019/03/04 00:55:28 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:810f4804-dbce-03b1-056a-a81269ca90c1 Address:127.0.0.1:8300}]
    2019/03/04 00:55:28 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2019/03/04 00:55:28 [INFO] serf: EventMemberJoin: devcube.dc1 127.0.0.1
    2019/03/04 00:55:28 [INFO] serf: EventMemberJoin: devcube 127.0.0.1
    2019/03/04 00:55:28 [INFO] consul: Adding LAN server devcube (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2019/03/04 00:55:28 [DEBUG] agent/proxy: managed Connect proxy manager started
    2019/03/04 00:55:28 [INFO] consul: Handled member-join event for server "devcube.dc1" in area "wan"
    2019/03/04 00:55:28 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2019/03/04 00:55:28 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2019/03/04 00:55:28 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
    2019/03/04 00:55:28 [INFO] agent: started state syncer
    2019/03/04 00:55:28 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
    2019/03/04 00:55:28 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2019/03/04 00:55:28 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2019/03/04 00:55:28 [DEBUG] raft: Votes needed: 1
    2019/03/04 00:55:28 [DEBUG] raft: Vote granted from 810f4804-dbce-03b1-056a-a81269ca90c1 in term 2. Tally: 1
    2019/03/04 00:55:28 [INFO] raft: Election won. Tally: 1
    2019/03/04 00:55:28 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2019/03/04 00:55:28 [INFO] consul: cluster leadership acquired
    2019/03/04 00:55:28 [INFO] consul: New leader elected: devcube
    2019/03/04 00:55:28 [INFO] connect: initialized primary datacenter CA with provider "consul"
    2019/03/04 00:55:28 [DEBUG] consul: Skipping self join check for "devcube" since the cluster is too small
    2019/03/04 00:55:28 [INFO] consul: member 'devcube' joined, marking health alive
    2019/03/04 00:55:28 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/03/04 00:55:28 [INFO] agent: Synced service "web"
    2019/03/04 00:55:28 [DEBUG] agent: Node info in sync
    2019/03/04 00:55:29 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2019/03/04 00:55:29 [DEBUG] agent: Service "web" in sync
    2019/03/04 00:55:29 [DEBUG] agent: Node info in sync
```

Соответствующая информация в журнале здесь - это инструкции синхронизации, относящиеся к сервису `web`. Агент Consul принял нашу конфигурацию и синхронизировал ее по всем узлам. В данном случае один узел.

#### Запрос сервиса DNS Consul

Мы можем запросить сервис с помощью DNS, как это было сделано с узлом. Вот так:

```
$ dig @127.0.0.1 -p 8600 web.service.consul

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51488
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.		IN	A

;; ANSWER SECTION:
web.service.consul.	0	IN	A	127.0.0.1

;; ADDITIONAL SECTION:
web.service.consul.	0	IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Mar 04 00:59:32 IST 2019
;; MSG SIZE  rcvd: 99
```

Мы также можем запросить DNS для служебных записей, которые дают нам больше информации о специфике сервиса, такой как порт и узел.

```
$ dig @127.0.0.1 -p 8600 web.service.consul SRV

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 712
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.		IN	SRV

;; ANSWER SECTION:
web.service.consul.	0	IN	SRV	1 1 80 devcube.node.dc1.consul.

;; ADDITIONAL SECTION:
devcube.node.dc1.consul. 0	IN	A	127.0.0.1
devcube.node.dc1.consul. 0	IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Mar 04 00:59:43 IST 2019
;; MSG SIZE  rcvd: 142
```

Вы также можете использовать TAG, предоставленный нами в определении сервиса, для запроса определенного тега:

```
$ dig @127.0.0.1 -p 8600 django.web.service.consul

; <<>> DiG 9.11.3-1ubuntu1.5-Ubuntu <<>> @127.0.0.1 -p 8600 django.web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12278
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;django.web.service.consul.	IN	A

;; ANSWER SECTION:
django.web.service.consul. 0	IN	A	127.0.0.1

;; ADDITIONAL SECTION:
django.web.service.consul. 0	IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Mar 04 01:01:17 IST 2019
;; MSG SIZE  rcvd: 106
```

#### Каталог сервисов Consul через HTTP API

Сервис также может быть запрошен с помощью HTTP API:

```
$ curl http://localhost:8500/v1/catalog/service/web

[
    {
        "ID": "810f4804-dbce-03b1-056a-a81269ca90c1",
        "Node": "devcube",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceKind": "",
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "django"
        ],
        "ServiceAddress": "",
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "ServiceMeta": {},
        "ServicePort": 80,
        "ServiceEnableTagOverride": false,
        "ServiceProxyDestination": "",
        "ServiceProxy": {},
        "ServiceConnect": {},
        "CreateIndex": 10,
        "ModifyIndex": 10
    }
]
```

Мы можем фильтровать сервисы на основе проверок работоспособности по HTTP API:

```
$ curl http://localhost:8500/v1/catalog/service/web?passing

[
    {
        "ID": "810f4804-dbce-03b1-056a-a81269ca90c1",
        "Node": "devcube",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceKind": "",
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "django"
        ],
        "ServiceAddress": "",
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "ServiceMeta": {},
        "ServicePort": 80,
        "ServiceEnableTagOverride": false,
        "ServiceProxyDestination": "",
        "ServiceProxy": {},
        "ServiceConnect": {},
        "CreateIndex": 10,
        "ModifyIndex": 10
    }
]
```

#### Обновление определения сервиса Consul

Если вы хотите обновить определение сервиса на работающем агенте Consul, это очень просто.

Есть три способа достичь этого. Вы можете отправить сигнал SIGHUP процессу, перезагрузить Consul, который внутренне отправляет SIGHUP на узел, или вы можете вызвать HTTP API, посвященный обновлениям определений сервисов, которые будут внутренне перезагружать конфигурацию агента.

```
$ ps aux | grep [c]onsul

pranav   21289  2.4  0.3 177012 54924 pts/2    Sl+  00:55   0:22 consul agent -dev -config-dir=./consul.d
```

Отправляем SIGHUP на номер 21289

```
$ kill -SIGHUP 21289
```

Или Перезагружаем Consul

```
$ consul reload
```

Конфигурация перезагрузки срабатывает.

Вы должны увидеть это в своем журнале Consul.

```
...
    2019/03/04 01:10:46 [INFO] agent: Caught signal:  hangup
    2019/03/04 01:10:46 [INFO] agent: Reloading configuration...
    2019/03/04 01:10:46 [DEBUG] agent: removed service "web"
    2019/03/04 01:10:46 [INFO] agent: Synced service "web"
    2019/03/04 01:10:46 [DEBUG] agent: Node info in sync
...
```

#### Веб-интерфейс Consul.

Consul предоставляет красивый веб-пользовательский интерфейс из коробки. Вы можете получить доступ к нему через [порт 8500](https://www.consul.io/docs/agent/options.html).

В этом случае http://localhost:8500. Давайте посмотрим на некоторые экраны.

Домашняя страница для сервисов пользовательского интерфейса Consul со всей соответствующей информацией, связанной с агентом Consul и проверкой веб-сервисов.

![](https://habrastorage.org/webt/28/t5/1b/28t51bpkwuf_4nkrq6i-duww73s.png)

*Изучение (Exploring) определенных сервисов в веб-интерфейсе Consul*

Переходя к более подробной информации о данной сервисе, мы получаем панель мониторинга сервисов со всеми узлами и их работоспособностью для этого сервиса.

![](https://habrastorage.org/webt/qi/pj/qe/qipjqetv710f5hlnsghvmodycis.png)

*Изучение (Exploring) информации на уровне узлов для каждого сервиса в веб-интерфейсе Consul*

На каждом отдельном узле мы можем посмотреть проверки работоспособности, сервисы и сессии.

![](https://habrastorage.org/webt/dr/ap/1z/drap1zxeceqnwted9kjvghoqvbu.png)

*Изучение информации о проверке работоспособности конкретного узла, информации об услугах и сессиях в веб-интерфейсе Consul.*

В целом, Consul Web UI действительно впечатляет и является отличным компаньоном для инструментов командной строки, которые предоставляет Consul.

### Чем Consul отличается от Zookeeper, doozerd и etcd?

Consul имеет первоклассную поддержку для обнаружения сервисов, проверки работоспособности (health-check), хранения ключ-значение, нескольких дата-центров.

[Zookeeper](https://zookeeper.apache.org/), [doozerd](https://github.com/ha/doozerd) и [etcd](https://github.com/etcd-io/etcd) в основном основаны на механизме хранения ключей и значений. Чтобы достичь чего-то сверх хранения ключей и значений, магазину нужны дополнительные инструменты, библиотеки и пользовательские разработки вокруг них.

Все эти инструменты, включая Consul, используют серверные узлы, которые требуют кворума узлов для работы и строго согласованы.

Более или менее, все они имеют сходную семантику для управления хранилищем ключей/значений.

Эта семантика привлекательна для построения систем обнаружения сервисов. Consul имеет готовую поддержку для обнаружения сервисов, которой нет в других системах.

Система обнаружения сервисов также требует способа выполнения проверок работоспособности. Так же важно проверить работоспособность сервиса, прежде чем позволить другим обнаружить его. Некоторые системы используют heartbeats с периодическими обновлениями и TTL. Работа по этим health checks растет с масштабом и требует фиксированной инфра-информации. Окно обнаружения сбоев имеет длину не менее длинны TTL.

В отличие от Zookeeper, у Consul есть агенты-клиенты, сидящие на каждом узле кластера, разговаривающие друг с другом в пуле gossip. Это позволяет клиентам быть тонкими, дает лучшую возможность проверки работоспособности, снижает сложность на стороне клиента и решает проблемы отладки.

Кроме того, Consul предоставляет встроенную поддержку интерфейсов [HTTP](https://www.consul.io/api/index.html) или [DNS](https://www.consul.io/docs/agent/dns.html) для выполнения общесистемных, узловых или сервисных операций. Другие системы нуждаются в тех, которые разрабатываются вокруг открытых примитивов.

Веб-сайт Consul дает хороший комментарий о сравнении между Consul и другими инструментами.

### Инструменты С Открытым Исходным Кодом для HashiCorp Consul

HashiCorp и сообщество построили несколько инструментов вокруг Consul.

Эти инструменты Consul создаются и управляются инженерами HashiCorp:

[Consul Template](https://github.com/hashicorp/consul-template) — стандартный рендеринг шаблона и уведомления с Consul. Рендеринг шаблонов, уведомитель и супервизор для данных HashiCorp Consul и Vault. Он обеспечивает удобный способ заполнения значений из Consul в файловую систему с помощью демона consul-template.

[Envconsul](https://github.com/hashicorp/envconsul) — считывает и устанавливает переменные среды для процессов из Consul. Envconsul предоставляет удобный способ запуска подпроцесса с переменными окружения, заполненными из HashiCorp Consul и Vault.

[Consul Replicate](https://github.com/hashicorp/consul-replicate) — Демон Consul cross-DC KV репликации. Этот проект предоставляет удобный способ репликации значений из одного центра обработки данных Consul в другой с помощью демона consul-replicate.

[Consul Migrate](https://github.com/hashicorp/consul-migrate) — средство переноса данных для обработки обновления Consul в Consul  0.5.1+.

Сообщество Consul также создало несколько инструментов для помощи в регистрации сервисов и управлении конфигурацией сервисов, я хотел бы упомянуть некоторые из популярных и хорошо поддерживаемых.

[Confd](https://github.com/kelseyhightower/confd) — управление локальными конфигурационными файлами приложений с помощью шаблонов и данных из etcd или consul.

[Fabio](https://github.com/fabiolb/fabio) — Fabio - это быстрый, современный маршрутизатор HTTP(S) и TCP с нулевой конфигурацией балансировки для развертывания приложений, управляемых Consul. Зарегистрируйте свои сервисы в consul, проведите проверку работоспособности, и Fabio начнет направлять на них трафик. Никакой конфигурации не требуется.

[Registrator](https://github.com/gliderlabs/registrator) — Мост реестра сервисов для Docker с подключаемыми адаптерами. Регистратор автоматически регистрирует и отменяет регистрацию сервисов для любого контейнера Docker, проверяя контейнеры по мере их поступления в сеть.

[Hashi-UI](https://github.com/jippi/hashi-ui) — современный пользовательский интерфейс для HashiCorp Consul & Nomad.

[Git2consul](https://github.com/breser/git2consul) — отражает содержимое репозитория git в Consul KVs. git2consul берет один или несколько репозиториев git и отражает их в Consul KVs. Цель состоит в том, чтобы организации любого размера использовали git в качестве резервного хранилища, контрольного журнала и механизма контроля доступа для изменений конфигурации, а Consul-в качестве механизма доставки.

[Spring-cloud-consul](https://github.com/spring-cloud/spring-cloud-consul) — этот проект обеспечивает интеграцию Consul для приложений Spring Boot посредством автоконфигурации и привязки к среде Spring и другим идиомам модели программирования Spring. С помощью нескольких простых аннотаций вы можете быстро включить и настроить общие шаблоны внутри вашего приложения и построить большие распределенные системы с компонентами на основе Consul.

[Crypt](https://github.com/xordataexchange/crypt) — храните и извлекайте зашифрованные конфигурации из etcd или consul.

[Mesos-Consul](https://github.com/mantl/mesos-consul) — Mesos to Consul bridge для открытия сервиса. Mesos-Consul автоматически регистрирует / отменяет регистрацию сервисов, запускаемых как задачи Mesos.

[Consul-cli](https://github.com/mantl/consul-cli) — Интерфейс командной строки для Consul HTTP API.

### Вывод

Распределенные системы не так просто построить и настроить. Поддержание их в рабочем состоянии - это совсем другая работа. HashiCorp Consul облегчает жизнь инженерам, сталкивающимся с такими проблемами.

По мере того, как мы изучали различные аспекты Consul, мы узнали, насколько простым для нас станет разработка и развертывание приложения с распределенной архитектурой или архитектурой микросервисов.

Простота использования, отличная документация, надежный готовый к производству код и поддержка сообщества позволяют довольно легко адаптировать и внедрить HashiCorp Consul в наш технологический стек.

Мы надеемся, что это была познавательное изучение Consul. Наше путешествие еще не закончилось, это была только первая половина. Мы снова встретимся с вами во второй части этой статьи, которая проведет нас через практические примеры, близкие к реальным приложениям.

Дайте нам знать, что вы хотели бы услышать от нас больше или если у вас есть какие-либо вопросы по этой теме, мы будем более чем рады ответить на них.

### Ссылки

[HashiCorp Consul](https://www.consul.io/) и [его РЕПО на GitHub](https://github.com/hashicorp/consul).

[HashiCorp Consul Guides](https://www.consul.io/docs/guides/index.html) and [Code](https://github.com/hashicorp/consul-guides)

[Микросервисы, как объяснил Мартин Фаулер и др.](https://martinfowler.com/articles/microservices.html)

[Статьи в блоге HashiCorp о компании Consul](https://www.hashicorp.com/blog/category/consul)

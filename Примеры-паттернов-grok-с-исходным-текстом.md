Если вы используете стек Elastic (ELK) и заинтересованы в сопоставлении пользовательских журналов Logstash с Elasticsearch, то этот пост для вас.

Эта статья посвящена Grok, которая является функцией в Logstash, которая может преобразовать ваши журналы

В этом посте приводятся примеры GROK паттернов с исходными текстами, для которых этот паттерн подходит. GROK паттерны и исходные тексты взяты из проекта [logstash-patterns-core](https://github.com/logstash-plugins/logstash-patterns-core).

![](https://habrastorage.org/getpro/habr/post_images/304/56f/8b5/30456f8b5cf3e9c91a68ebecfd20d31d.png)

Паттерн `%{SYSLOGBASE2} %{GREEDYDATA:message}`. Парсит логи syslog. 

```
Mar 16 00:01:25 evita postfix/smtpd[1713]: connect from camomile.cloud9.net[168.100.1.3]
```

Парсит он их так:

![](https://habrastorage.org/webt/pp/le/d-/ppled-fahvy9yinmxqdsb8jlwuy.jpeg)

Разбор остальные grok паттернов будет в текстовом виде.


Если вы используете стек Elastic (ELK) и заинтересованы в сопоставлении пользовательских журналов Logstash с Elasticsearch, то этот пост для вас.

Эта статья посвящена [fluent-plugin-grok-parser](https://github.com/fluent/fluent-plugin-grok-parser) - плагину для [td-agent](https://www.fluentd.org/download), которая может преобразовать ваши журналы на том же сервере где и собираются логи с помощью паттернов GROK.

В этом посте приводятся примеры GROK паттернов с исходными текстами, для которых этот паттерн подходит. А также минимальная настройка.

Пример парсинга syslog c помощью паттерна `%{SYSLOGLINE}`, ввклюенного в базову поставку fluent-plugin-grok-parser.

![](https://habrastorage.org/webt/pp/le/d-/ppled-fahvy9yinmxqdsb8jlwuy.jpeg)



Для теста сделаем 2 севрера: PostgreSQL и ELK:

![](https://habrastorage.org/webt/wt/al/o1/wtalo1sfhlomvzrwtdykirnl1kc.png)



На сервере PostgreSQL (Centos 7) установим PostgreSQL и td-agent 3.

Активируем логирование запросов и после изменения конфига запустим PostgreSQL.

```
log_statement = 'all'
```

Логи pgbench примерно такие

```
< 2020-08-14 04:06:01.744 UTC > LOG:  statement: insert into pgbench_tellers(tid,bid,tbalance) values (1500,150,0)
```



После установки td-agent 3 необходимо установить плагин.

```
wget https://rubygems.org/downloads/fluent-plugin-grok-parser-2.6.1.gem
td-agent-gem install --local  fluent-plugin-grok-parser-2.6.1.gem
```

Добавим дополнительне паттерны для PostgreSQL из поста [Мониторинг ошибок и событий в журнале PostgreSQL (grok_exporter)](https://habr.com/ru/post/487302/).







Паттерн `%{SYSLOGBASE2} %{GREEDYDATA:message}`. Парсит логи syslog. 

```
Mar 16 00:01:25 evita postfix/smtpd[1713]: connect from camomile.cloud9.net[168.100.1.3]
```

GROK паттерны и исходные тексты взяты из проекта [logstash-patterns-core](https://github.com/logstash-plugins/logstash-patterns-core).



Разбор остальные grok паттернов будет в текстовом виде.


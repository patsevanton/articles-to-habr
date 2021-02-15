

------------------

# Лабораторная работа 7a

## Упражнения по защите Hadoop 

В этой лабораторной работе мы увидим, как взаимодействовать с компонентами Hadoop (HDFS, Hive, Hbase, Sqoop), работающими в керборизованном кластере, и создавать соответствующие политики авторизации Ranger для доступа.

- Мы настроим политики Ranger для:
  - Защиты каталога /sales HDFS - доступ к нему имеет только группа продаж
  - Защиты таблицы продаж - чтобы только группа продаж имела к ней доступ
  - Защиты таблицу продаж HBase, чтобы только группа продаж имела к ней доступ

#### Защищенный доступ к HDFS

- Цель: создать каталог / sales в HDFS и обеспечить доступ только пользователям, принадлежащим к группе продаж (и администраторам).


- Войдите в Ranger (используя admin / admin) и убедитесь, что репозиторий HDFS был правильно настроен в Ranger.


- - В Ranger> Under Service Manager > HDFS > щелкните значок Edit (рядом со значком корзины), чтобы отредактировать репозиторий HDFS.
  - Нажмите "Test connection".
  - Если не удается проверить соединение, повторно введите поля ниже и повторите попытку:
    - Username: `rangeradmin@LAB.HORTONWORKS.NET`
    - Password: BadPass#1
    - RPC Protection type: Authentication
  - После успешного прохождения теста нажмите Save  

- Создать каталог /sales в HDFS как hadoopadmin
```
#аутентификация
sudo -u hadoopadmin kinit
# введите пароль: BadPass#1

#создайте каталог и установите разрешения на 000
sudo -u hadoopadmin hdfs dfs -mkdir /sales
sudo -u hadoopadmin hdfs dfs -chmod 000 /sales
```

- Теперь войдите в систему как sales1 и попытайтесь получить к нему доступ перед добавлением любой политики Ranger HDFS
```
sudo su - sales1

hdfs dfs -ls /sales
```
- Ошибка `GSSException: No valid credentials provided`появляется, потому что кластер керберизован, а мы еще не прошли аутентификацию.

- Аутентифицируйтесь как пользователь sales1 и проверьте билет
```
kinit
# введите пароль: BadPass#1

klist
## Принципал по умолчанию: sales1@LAB.HORTONWORKS.NET
```
- Теперь попробуйте снова получить доступ к каталогу как sales1
```
hdfs dfs -ls /sales
```
- На этот раз происходит сбой с ошибкой авторизации: 
  - `Permission denied: user=sales1, access=READ_EXECUTE, inode="/sales":hadoopadmin:hdfs:d---------`

- Войдите в пользовательский интерфейс Ranger, например в http://RANGER_HOST_PUBLIC_IP:6080/index.html как admin/admin

- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать: 
  - Service Type: `HDFS`
  - User: `sales1`
  
- Обратите внимание, что Ranger зафиксировал попытку доступа, и, поскольку в настоящее время нет политики, разрешающей доступ, она была запрещена («Denied»)
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HDFS-denied.png)

- Чтобы создать политику HDFS в Ranger, выполните следующие действия.:
  - На вкладке "Access Manager" нажмите HDFS > (clustername)_hadoop
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HDFS-policy.png)
  - Откроется список политик HDFS
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HDFS-edit-policy.png)
  - Нажмите кнопку 'Add New Policy' чтобы создать новую, разрешающую пользователям группы `sales` доступ к каталогу `/sales` dir:
    - Policy Name: `sales dir`
    - Resource Path: `/sales`
    - Group: `sales`
    - Permissions : `Execute Read Write`
    - Add
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HDFS-create-policy.png)

- Подождите 30 секунд, чтобы политика вступила в силу
  
- Теперь попробуйте снова получить доступ к каталогу как sales1, и теперь ошибки нет
```
hdfs dfs -ls /sales
```

- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать:
  - Service Type: HDFS
  - User: sales1
- Обратите внимание, что Ranger зафиксировал попытку доступа, и поскольку на этот раз существует политика, разрешающая доступ, она была разрешена `Allowed`
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HDFS-allowed.png)  
- Вы также можете увидеть детали, которые были получены для каждого запроса:
    - Политика, разрешившая доступ
    - Время
    - Запрашивающий пользователь
    - Тип службы (например, hdfs, hive, hbase и т. д.)
    - Название ресурса
    - Тип доступа (например, чтение, запись, выполнение)
    - Результат (например, разрешен или запрещен)
    - Enforcer доступы (т. е. Использовались ли собственные acl или контрольные адреса рейнджера)
    - Клиентский IP
    - Количество событий
- Обратите внимание, что для любых разрешенных запросов вы можете быстро проверить сведения о политике, которая разрешила доступ, щелкнув номер политики в столбце «Policy ID».
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-policy-details.png)  
- Теперь давайте проверим, могут ли некоммерческие пользователи получить доступ к каталогу.
- Выйдите из системы как sales1 и войдите снова как hr1
```
kdestroy
#выйдите как sales1
logout

#войдите как hr1 и аутинтефицируйтесь
sudo su - hr1

kinit
# введите пароль: BadPass#1

klist
## Принципал по умолчанию: hr1@LAB.HORTONWORKS.NET
```
- Попробуйте получить доступ к тому же каталогу, что и hr1, и заметьте, что это не удается
```
hdfs dfs -ls /sales
## ls: Permission denied: user=hr1, access=READ_EXECUTE, inode="/sales":hadoopadmin:hdfs:d---------
```

- В Ranger нажмите «Audit», чтобы открыть страницу аудита, и на этот раз отфильтруйте его по «Resource Name»
  - Service Type: `HDFS`
  - Resource Name: `/sales`
  
- Обратите внимание, что вы можете видеть историю / детали всех запросов, сделанных для каталога / sales:
  - создано hadoopadmin
  - первоначальный запрос пользователя sales1 был отклонен
  - разрешен последующий запрос от пользователя sales1 (после создания политики)
  - запрос пользователя hr1 был отклонен![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HDFS-summary.png)  

- выйдите из hr1
```
kdestroy
logout
```
- Мы успешно настроили каталог HDFS, который доступен только группе продаж (и администраторам).

#### Защищенный доступ к Hive

- Цель: настроить политики авторизации Hive, чтобы продавцы имели доступ только к коду, столбцы описания в default.sample_07
- Включите Hive на tez, установив параметры ниже и перезапустив Hive. 
  - Ambari > Hive > Configs  	
    - Execution Engine = Tez
- Убедитесь, что репозиторий HIVE был правильно настроен в Ranger
  - В Ranger > Service Manager > HIVE > Нажмите на иконку Edit (рядом с иконкой корзины) чтобы редактировать репозиторий HIVE
  - Нажмите на 'Test connection' 
  - если это не удается, повторно введите поля ниже и повторите попытку:
    - Username: `rangeradmin@LAB.HORTONWORKS.NET`
    - Password: BadPass#1
  - Когда все успешно пройдет, нажмите Save  
- Теперь выполните эти шаги с узла, на котором установлен Hive (или клиент).
- Войдите в систему как sales1 и попытайтесь подключиться к базе данных по умолчанию в Hive через beeline и получить доступ к таблице sample_07
- Обратите внимание, что в строке подключения JDBC для подключения к защищенному Hive, когда он работает в транспортном режиме по умолчанию (то есть в двоичном) :
  - порт остается 10000
  - *теперь необходимо передать принципала Kerberos*
- Войдите в систему как sales1 без тикета Kerberos и попробуйте открыть соединение beeline:
```
sudo su - sales1
kdestroy
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@LAB.HORTONWORKS.NET"
```
- Это не сработает с `GSS initiate failed` потому что кластер керберизован, и мы еще не прошли аутентификацию.

- Выйдите из beeline:
```
!q
```
- Аутентифицируйтесь как пользователь sales1 и проверьте тикет
```
kinit
# введите пароль: BadPass#1

klist
## Default principal: sales1@LAB.HORTONWORKS.NET
```
- Теперь попробуйте подключиться к Hive через beeline как sales1
```
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@LAB.HORTONWORKS.NET"
```

- Если вы получаете указанную ниже ошибку, это потому, что вы не добавили hive в глобальную политику KMS на более раннем этапе (вместе с nn, hadoopadmin). Вернитесь и добавьте его
```
org.apache.hadoop.security.authorize.AuthorizationException: User:hive not allowed to do 'GET_METADATA' on 'testkey'
```

- На этот раз он подключается. Теперь попробуйте запустить запрос
```
beeline> select code, description from sample_07;
```
- Теперь не получается с ошибкой авторизации: 
  - `HiveAccessControlException Permission denied: user [sales1] does not have [SELECT] privilege on [default/sample_07]`
- Войдите в пользовательский интерфейс Ranger, например в http://RANGER_HOST_PUBLIC_IP:6080/index.html как admin/admin
- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать по ниже. 
  - Service Type: `Hive`
  - User: `sales1`
- Обратите внимание, что Ranger зафиксировал попытку доступа, и, поскольку в настоящее время нет политики, разрешающей доступ, она была отклонена `Denied`
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HIVE-denied.png)
- Чтобы создать политику HIVE в Ranger, выполните следующие действия.:
  - Во вкладке 'Access Manager' нажмите HIVE > (clustername)_hive
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HIVE-policy.png)
  - Это откроет список политик HIVE
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HIVE-edit-policy.png)
  - Нажмите кнопку «Add New Policy», чтобы создать новую, разрешающую пользователям группы `sales` доступ к столбцам `code`, `description` and `total_emp` в каталоге `sample_07`:
    - Policy Name: `sample_07`
    - Hive Database: `default`
    - table: `sample_07`
    - Hive Column: `code` `description` `total_emp`
    - Group: `sales`
    - Permissions : `select`
    - Add
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HIVE-create-policy.png)
- Обратите внимание: когда вы вводили имя базы данных и таблицы, Ranger смог найти их и автозаполнить
  -  Это было сделано с использованием принципа rangeradmin, который мы предоставили во время установки Ranger.
- Также обратите внимание, что разрешения настраиваются только для разрешения доступа, и вы не можете явно запретить пользователю / группе доступ к ресурсу, если вы не включили условия запрета во время установки Ranger (шаг 8).
- Подождите 30 секунд, чтобы принять новую политику

- Теперь попробуйте снова получить доступ к столбцам, и теперь запрос работает
```
beeline> select code, description, total_emp from sample_07;
```

- Однако обратите внимание, что если вместо этого вы попытаетесь описать таблицу или запросить все столбцы, это будет отклонено, потому что мы предоставили продавцам доступ только к двум столбцам в таблице
  - `beeline> desc sample_07;`
  - `beeline> select * from sample_07;`
  
- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать:
  - Service Type: HIVE
  - User: sales1
  
- Обратите внимание, что Ranger зафиксировал попытку доступа, и поскольку на этот раз существует политика, разрешающая доступ, она была разрешена `Allowed`
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HIVE-allowed.png)  
- Вы также можете увидеть детали, которые были получены для каждого запроса:
    - Policy that allowed the access
    - Time
    - Requesting user
    - Service type (e.g. hdfs, hive, hbase etc)
    - Resource name 
    - Access type (e.g. read, write, execute)
    - Result (e.g. allowed or denied)
    - Access enforcer (i.e. whether native acl or ranger acls were used)
    - Client IP
    - Event count
  
- Обратите внимание, что для любых разрешенных запросов вы можете быстро проверить сведения о политике, которая разрешила доступ, щелкнув номер политики в столбце «Policy ID».
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HIVE-policy-details.png)  


- Мы также можем ограничить доступ sales1 только подмножеством данных с помощью фильтра на уровне строк. Предположим, мы хотим разрешить группе продаж доступ только к данным, для которых total_emp меньше 5000.
- На странице Hive Policies выберите вкладку «Row Level Filter» и нажмите «Add New Policy».![Image](/screenshots/Ranger-HIVE-select-row-level-filter-tab.png)  
    - Обратите внимание, что для применения политики фильтрации на уровне строк у пользователя / группы уже должны быть разрешения на выбор в таблице. 
- Создайте политику, ограничивающую доступ только к строкам, где `total_emp` меньше 5000:
    - Policy Name: `sample_07_filter_total_emp`
    - Hive Database: `default`
    - table: `sample_07`
    - Group: `sales`
    - Permissions : `select`
    - Row Level Filter : `total_emp<5000`
    	- Синтаксис фильтра аналогичен тому, что вы написали бы после предложения WHERE в запросе SQL.
    - Add
  ![Image](/screenshots/Ranger-HIVE-create-row-level-filter-policy.png)
- Подождите 30 секунд, чтобы принять новую политику
- Теперь попробуйте снова получить доступ к столбцам и обратите внимание, как отображаются только строки, соответствующие критериям фильтра.
```
beeline> select code, description, total_emp from sample_07;
```

- Вернитесь на страницу Ranger Audits и обратите внимание, как политика фильтрации была применена к запросу.


- Предположим, теперь мы хотим замаскировать столбец total_emp из sales1. Это отличается от запрета доступа тем, что пользователь может запрашивать столбец, но не может видеть фактические данные.

- На странице политик Hive выберите вкладку «Masking» и нажмите «Add New Policy».

- ![Image](/screenshots/Ranger-HIVE-select-masking-tab.png)  


- - Обратите внимание, что для того, чтобы замаскировать столбец, у пользователя / группы уже должны быть права на «выбор» для этого столбца. Создание политики маскирования для столбца, к которому у пользователя нет доступа, запретит пользователю доступ
- Создайте политику, маскирующую столбец  `total_emp` для пользователей группы `sales`:
    - Policy Name: `sample_07_total_emp`
    - Hive Database: `default`
    - table: `sample_07`
    - Hive Column: `total_emp`
    - Group: `sales`
    - Permissions : `select`
    - Masking Option : `redact`
    	- Обратите внимание на различные доступные варианты маскировки
    	- Параметр маскирования Custom может использовать любой UDF Hive, если он возвращает тот же тип данных, что и столбец. 

    - Add
  ![Image](/screenshots/Ranger-HIVE-create-masking-policy.png)
- Подождите 30 секунд, чтобы принять новую политику
- Теперь попробуйте снова получить доступ к столбцам и обратите внимание, как маскируются результаты для столбца `total_emp`
```
beeline> select code, description, total_emp from sample_07;
```

- Вернитесь на страницу Ranger Audits и обратите внимание, как к запросу была применена политика маскирования.
- Выйдите из beeline
```
!q
```
- Now let's check whether non-sales users can access the table

- Logout as sales1 and log back in as hr1
```
kdestroy
#выйдите как sales1
logout

#войдите как hr1 и аутентифицируйтесь
sudo su - hr1

kinit
# введите пароль: BadPass#1

klist
## Default principal: hr1@LAB.HORTONWORKS.NET
```
- Попытайтесь получить доступ к той же таблице, что и hr1, и заметьте, что это не удается
```
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@LAB.HORTONWORKS.NET"
```
```
beeline> select code, description from sample_07;
```
- В Ranger нажмите «Audit», чтобы открыть страницу аудита, и отфильтруйте 'Service Type' = 'Hive'
  - Service Type: `HIVE`

- Здесь вы можете увидеть, что запрос от sales1 был разрешен, но hr1 был отклонен

![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HIVE-summary.png)  

- Выйдите из beeline
```
!q
```
- Выйдите из hr1
```
logout
```



- Мы настроили политики авторизации Hive, чтобы только продавцы имели доступ к коду, столбцы описания в default.sample_07


#### Защищенный доступ HBase

- Цель: создать таблицу под названием "продажи" в HBase и настроить политики авторизации, чтобы гарантировать, что только пользователи отдела продаж имеют доступ к таблице.
- Выполните эти шаги с любого узла, на котором установлены службы Hbase Master или RegionServer.

- Войти как sales1
```
sudo su - sales1
```
-  Запустите оболочку hbase
```
hbase shell
```
- Список таблиц в базе данных по умолчанию
```
hbase> list 'default'
```
- Ошибка GSSException: не предоставлены действительные учетные данные, потому что кластер керберизован, а мы еще не прошли аутентификацию.
- Чтобы выйти из оболочки hbase:
```
exit
```
- Аутентифицируйтесь как пользователь sales1 и проверьте тикет
```
kinit
# введите пароль: BadPass#1

klist
## Принципал по умолчанию: sales1@LAB.HORTONWORKS.NET
```
- Теперь попробуйте подключиться к оболочке Hbase и перечислить таблицы как sales1.	
```
hbase shell
hbase> list 'default'
```
- На этот раз все работает. Теперь попробуйте создать таблицу под названием `sales` с семейством столбцов под названием`cf`
```
hbase> create 'sales', 'cf'
```
- Теперь не получается с ошибкой авторизации: 
  - `org.apache.hadoop.hbase.security.AccessDeniedException: Insufficient permissions for user 'sales1@LAB.HORTONWORKS.NET' (action=create)`
  - Примечание: сверху будет много вывода. Ошибка будет в строке сразу после вашей команды создания
- Войдите в пользовательский интерфейс Ranger, например в http://RANGER_HOST_PUBLIC_IP:6080/index.html как admin/admin
- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать по ниже. 
  - Service Type: `Hbase`
  - User: `sales1`
- Обратите внимание, что Ranger зафиксировал попытку доступа, и, поскольку в настоящее время нет политики, разрешающей доступ, она была отклонена  `Denied`
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HBASE-denied.png)
- Чтобы создать политику HBASE в Ranger, выполните следующие действия:
  - На вкладке "Access Manager" щелкните HBASE> (имя кластера) _hbase.
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HBASE-policy.png)
  - Откроется список политик HBASE
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HBASE-edit-policy.png)
  - Нажмите кнопку «Add New Policy», чтобы создать новую политику, разрешающую пользователям группы «sales» доступ к таблице «продажи» в HBase:
    - Policy Name: `sales`
    - Hbase Table: `sales`
    - Hbase Column Family: `*`
    - Hbase Column: `*`
    - Group : `sales`    
    - Permissions : `Admin` `Create` `Read` `Write`
    - Add
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HBASE-create-policy.png)
- Подождите 30 секунд, чтобы политика вступила в силу
- Теперь попробуйте создать таблицу, и теперь она работает
```
hbase> create 'sales', 'cf'
```

- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать по ниже:
  - Service Type: HBASE
  - User: sales1
  
- Обратите внимание, что Ranger зафиксировал попытку доступа, и поскольку на этот раз существует политика, разрешающая доступ, она была одобрена `Allowed`
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HBASE-allowed.png)  
- Вы также можете увидеть детали, которые были получены для каждого запроса:
    - Policy that allowed the access
    - Time
    - Requesting user
    - Service type (e.g. hdfs, hive, hbase etc)
    - Resource name 
    - Access type (e.g. read, write, execute)
    - Result (e.g. allowed or denied)
    - Access enforcer (i.e. whether native acl or ranger acls were used)
    - Client IP
    - Event count
  
- Обратите внимание, что для любых разрешенных запросов вы можете быстро проверить сведения о политике, которая разрешила доступ, щелкнув номер политики в столбце «Policy ID».
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HBASE-policy-details.png)  

- Выйти из оболочки hbase
```
hbase> exit
```

- Теперь проверим, есть ли доступ к таблице для непродавцов.
- Выйдите из системы как sales1 и войдите снова как hr1
```
kdestroy
#выйдите из sales1
logout

#войдите как hr1 
sudo su - hr1

kinit
# введите пароль: BadPass#1

klist
## Default principal: hr1@LAB.HORTONWORKS.NET
```
- Попробуйте получить доступ к тому же каталогу, что и hr1, и обратите внимание, что этот пользователь даже не видит таблицу
```
hbase shell
hbase> describe 'sales'
hbase> list 'default'
```

![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-hbase-sales.png)

- Попробуйте создать таблицу как hr1. Она не работает с `org.apache.hadoop.hbase.security.AccessDeniedException: Insufficient permissions`
```
hbase> create 'sales', 'cf'
```
- В Ranger нажмите «Audit», чтобы открыть страницу аудита и отфильтровать по:
  - Service Type: `HBASE`
  - Resource Name: `sales`

- Здесь вы можете увидеть, что запрос от sales1 был разрешен, но hr1 был отклонен

![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-HBASE-summary.png)  

- Выйти из оболочки hbase
```
hbase> exit
```

- Выйти из hr1
```
kdestroy
logout
```
- Мы успешно создали таблицу под названием "продажи" в HBase и настроили политики авторизации, чтобы  только продавцы имели доступ к таблице.
- Это показывает, как вы можете взаимодействовать с компонентами Hadoop в керберизованном кластере и использовать Ranger для управления политиками авторизации и аудита.

<!---
- **TODO: fix for 2.5. Skip for now ** На этом этапе ваша панель аудита Silk / Banana должна отображать данные аудита из нескольких компонентов Hadoop, например http://54.68.246.157:6083/solr/banana/index.html#/dashboard

![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-banana.png)  
--->

#### (Необязательно) Используйте Sqoop для импорта

- Если Sqoop еще не установлен, установите его через Ambari на том же узле, где установлен Mysql / Hive:
  - Admin > Stacks and Versions > Sqoop > Add service > выберите узел, на котором установлен Mysql / Hive, примите все значения по умолчанию и, наконец, нажмите «Proceed Anyway»
  - Вам будет предложено ввести имя администратора / пароль:
    - `hadoopadmin@LAB.HORTONWORKS.NET`
    - BadPass#1
  
- * *На хосте, на котором запущен Mysql*: измените пользователя на root, загрузите образец csv и войдите в Mysql
```
sudo su - 
wget https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/labdata/PII_data_small.csv
mysql -u root -pBadPass#1
```

- Это позволит вам: 
  - создать таблицу в Mysql
  - дать доступ к sales1
  - импортировать данные из csv
  - проверить, что таблица была создана
```
create database people;
use people;
create table persons (people_id INT PRIMARY KEY, sex text, bdate DATE, firstname text, lastname text, addresslineone text, addresslinetwo text, city text, postalcode text, ssn text, id2 text, email text, id3 text);
GRANT ALL PRIVILEGES ON people.* to 'sales1'@'%' IDENTIFIED BY 'BadPass#1';
LOAD DATA LOCAL INFILE '~/PII_data_small.csv' REPLACE INTO TABLE persons FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';

select people_id, firstname, lastname, city from persons where lastname='SMITH';
exit
```

- выйти из системы root
```
logout
```

- Создайте политику Ranger, чтобы разрешить группе `sales`  `все разрешения` на панели`persons` в Hive
  - Access Manager > Hive > (cluster)_hive > Add new policy
  - Создайте новую политику, как показано ниже, и нажмите Add:
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HIVE-create-policy-persons.png) 

- Создайте политику Ranger, чтобы разрешить группе `sales` все разрешения` в каталоге` / ranger / audit / kms` в HDFS
  - Access Manager > HDFS > (cluster)_hdfs > Add new policy
  - Создайте новую политику, как показано ниже, и нажмите:
  **TODO: добавить скриншот**

  - Выйти из Ranger
  
- Создайте политику Ranger, чтобы разрешить группе `sales` получить метаданные, генерировать ЕЕК, расшифровывать ЕЕК ( `Get Metadata` `GenerateEEK` `DecryptEEK`) для «testkey» (т. е. Ключа, используемого для шифрования каталогов хранилища Hive)
  - Войдите в Ranger http://RANGER_PUBLIC_IP:6080 с помощью keyadmin / keyadmin
  - Access Manager > KMS > (cluster)_KMS > Add new policy
  - Создайте новую политику, как показано ниже, и нажмите Add:
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-KMS-create-policy-testkey.png)  
  - Выйдите из Ranger и повторно войдите как admin / admin

- Войдите как sales1
```
sudo su - sales1
```

- В качестве пользователя sales1 выполните задание sqoop для создания таблицы лиц в Hive (в формате ORC) и импорта данных из MySQL. Ниже приведены подробности аргументов, переданных в:
  - Table: MySQL table name
  - username: Mysql username
  - password: Mysql password
  - hcatalog-table: Hive table name
  - create-hcatalog-table: hive table should be created first
  - driver: classname for Mysql driver
  - m: number of mappers
  
```
kinit
## введите пароль BadPass#1

sqoop import --verbose --connect "jdbc:mysql://$(hostname -f)/people" --table persons --username sales1 --password BadPass#1 --hcatalog-table persons --hcatalog-storage-stanza "stored as orc" -m 1 --create-hcatalog-table  --driver com.mysql.jdbc.Driver
```
- Это запустит задание mapreduce для импорта данных из Mysql в Hive в формате ORC.
- Примечание: если задание mapreduce не выполняется с помощью приведенного ниже, скорее всего, вы не предоставили группе продаж все разрешения, необходимые для EK, используемого для шифрования каталогов Hive. 
```
 java.lang.RuntimeException: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
```

- Также обратите внимание: если задание mapreduce завершается неудачно из-за того, что пользователь отдела продаж не имеет доступа на запись в / apps / hive / inventory, вам необходимо создать политику HDFS, разрешающую доступ пользователя sales1 и hive к /apps/hive/warehouse 
- Войдите в beeline
```
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@LAB.HORTONWORKS.NET"
```

- Таблица запросов лиц в билайн
```
beeline> select * from persons;
```
- Поскольку существует политика авторизации, запрос должен работать
- Аудит рейнджеров должен показать, что запрос был разрешен:
  - Under Ranger > Audit > query for
    - Service type: HIVE
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-HIVE-audit-persons.png)


##### Отбрасываем зашифрованные таблицы Hive 

- Из билайна попробуйте сбросить таблицу лиц. 
```
beeline> drop table persons;
```
- Вы получите ошибку, аналогичную приведенной ниже
```
message:Unable to drop default.persons because it is in an encryption zone and trash is enabled.  Use PURGE option to skip trash.
```

- Чтобы удалить таблицу Hive (когда каталоги Hive расположены в EncryptionZone), вам необходимо включить `purge`, как показано ниже.:
```
beeline> drop table persons purge;
```

- Удалите тикет и выйдите из системы как sales1
```
kdestroy
logout
```

- На этом лабораторная работа завершена. Теперь вы взаимодействовали с компонентами Hadoop в защищенном режиме и использовали Ranger для управления политиками авторизации и аудита.

------------------

# Lab 7b

------------------

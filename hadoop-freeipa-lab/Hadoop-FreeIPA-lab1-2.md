### Инструкции для проведения работ в IPA. Перевод https://github.com/HortonworksUniversity/Security_Labs/blob/master/HDP-3.1-IPA.md

### Предварительные требования 

- HDP 3.x / Ambari 2.7.1 cluster
- Доступ к IPA-серверу, который был настроен, как описано в [документации Hortonworks](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/authentication-with-kerberos/content/kerberos_optional_use_an_existing_ipa.html). Смотрите образец [скрипта](https://github.com/HortonworksUniversity/Security_Labs/blob/master/extras/ipa.md) 

**Lab Topics**

0. Доступ к вашему кластеру
1. [Регистрация узлов кластера в качестве клиентов IPA](#section-1)
2. [Защита Ambari с помощью настройки безопасности ambari-сервера](#section-2)
3. [Включение Kerberos для служб кластера](#section-3)
4. [Включение LDAP для Ambari](#section-4)


# Lab 1

## Доступ к вашему кластеру

Учетные данные будут предоставлены оператором для этих сервисов:

* SSH
* Ambari

## Использование Вашего Кластера

### Подключение с помощью Putty через Windows

- Щелкните правой кнопкой мыши, чтобы загрузить [этот ключ ppk](https://github.com/HortonworksUniversity/Security_Labs/raw/master/training-keypair.ppk) > Сохранить ссылку как> Сохранить в папку загрузок
- Используйте putty для подключения к вашему узлу с помощью ключа ppk:
  - Connection > SSH > Auth > Private key for authentication > Browse... > Select training-keypair.ppk

  ![](https://habrastorage.org/webt/ln/1x/-x/ln1x-xyufuwki5ify7rkt-e4hsq.png)
  
- Обязательно нажмите "Save" на странице сеанса перед входом в систему.
- При подключении вам будет предложено ввести имя пользователя. Введите `centos`

### Подключение с Linux/MacOSX 

- SSH в узел Ambari вашего кластера, используя следующие шаги:
- Щелкните правой кнопкой мыши, чтобы загрузить [этот ключ pem](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/training-keypair.pem)  > Сохранить ссылку как> Сохранить в папку загрузок
  - Скопируйте ключ pem в каталог ~ / .ssh и исправьте разрешения
  ```
  cp ~/Downloads/training-keypair.pem ~/.ssh/
  chmod 400 ~/.ssh/training-keypair.pem
  ```
 - Войдите в узел Ambari, который вам был назначен, заменив IP_ADDRESS_OF_AMBARI_NODE ниже на IP-адрес узла Ambari (ваш оператор предоставит это)   
  ```
  ssh -i  ~/.ssh/training-keypair.pem centos@IP_ADDRESS_OF_AMBARI_NODE
  ```
  - Чтобы изменить пользователя на root, вы можете использовать команду:
  ```
  sudo su -
  ```

- Аналогичным образом войдите через SSH на каждый из других узлов в вашем кластере, поскольку вам нужно будет запускать команды на каждом узле в будущей работе.

- Совет: поскольку в следующих работах вам потребуется выполнить *один и тот же набор команд* на каждом из узлов кластера, сейчас самое время настроить для этого ваш любимый инструмент: примеры [здесь](https://www.reddit.com/r/sysadmin/comments/3d8aou/running_linux_commands_on_multiple_servers/)
  - В OSX простой способ сделать это - использовать [iTerm](https://www.iterm2.com/): открыть несколько вкладок / разделов, а затем использовать функцию 'Broadcast input' (в разделе Shell -> Broadcast)
  - Если вы еще не знакомы с таким инструментом, вы также можете просто запускать команды в кластере, по одному хосту за раз.

#### Вход в Ambari

- Войдите в веб-интерфейс Ambari, открыв http://AMBARI_PUBLIC_IP:8080 и авторизуйтесь с admin/BadPass#1

- В левой части страницы вы увидите список компонентов Hadoop, работающих в вашем кластере.
  - Все они должны отображаться зеленым цветом (т.е. запущены). Если нет, запустите их с помощью Ambari через меню «Действия службы» для этой службы.

#### Поиск внутренних / внешних хостов

- Ниже приведены полезные методы, которые вы можете использовать в будущих работах, чтобы найти детали вашего кластера:

  - Как я могу найти имя кластера с терминала SSH?
  ```
  #запуск на узле Ambari, чтобы получить имя кластера через API Ambari
  PASSWORD=BadPass#1
  output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://localhost:8080/api/v1/clusters`
  cluster=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`
  echo $cluster
  ```
  - Как с терминала SSH найти внутреннее имя хоста (также называемое FQDN) узла, в который я вошел?
  ```
  $ hostname -f
  ip-172-30-0-186.us-west-2.compute.internal  
  ```
  
  - Как проверить имя кластера в Ambari?
    
    - Он отображается в левом верхнем углу панели инструментов Ambari рядом с логотипом Ambari. Если имя кажется усеченным, вы можете навести на него курсор, чтобы отобразить диалоговое окно справочного текста с полным именем.![](https://habrastorage.org/webt/ej/bq/jo/ejbqjo-fkj9vkaabbutnwj4r1am.png)
    
  - Как в Ambari найти внешнее имя хоста узла, на котором установлен компонент (например, Resource Manager или Hive)?
    - Щелкните на родительский сервис (например, YARN) и *наведите указатель мыши на* имя компонента. Появится внешнее имя хоста.
  
    ![](https://habrastorage.org/webt/8e/v0/jh/8ev0jhz9zsib4os7jzrfl80j0vy.png)
    
  - Как в Ambari найти внутреннее имя хоста узла, на котором установлен компонент (например, Resource Manager или Hive)?
    - Щелкните родительскую службу (например, YARN) и *нажмите на* имя компонента. Вы попадете на страницу хостов этого узла, где отобразится внутреннее имя хоста вверху.
    
    ![](https://habrastorage.org/webt/bz/dm/dc/bzdmdc6g-nd-4su0yfdpjheqplm.png)
    
  - В будущих labs вам может потребоваться предоставить частное или общедоступное имя хоста узлов, на которых запущен конкретный компонент (например, YARN RM, Mysql или HiveServer).
  
  
#### Импорт образцов данных в Hive 


- Запуститесь *на узле, где установлен HiveServer2* чтобы загрузить данные и импортировать их в таблицу Hive для последующих лабораторных работ.
  - Вы можете найти узел с помощью Ambari, как описано в Лабораторной 1.
  - Скачайте и импортируйте данные
  ```
  cd /tmp
  wget https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/labdata/sample_07.csv
  wget https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/labdata/sample_08.csv
  ```
  - Создайте каталог пользователей для admin, sales1 и hr1
  ```
   sudo -u hdfs hdfs dfs  -mkdir /user/admin
   sudo -u hdfs hdfs dfs  -chown admin:hadoop /user/admin

   sudo -u hdfs hdfs dfs  -mkdir /user/sales1
   sudo -u hdfs hdfs dfs  -chown sales1:hadoop /user/sales1
   
   sudo -u hdfs hdfs dfs  -mkdir /user/hr1
   sudo -u hdfs hdfs dfs  -chown hr1:hadoop /user/hr1   
  ```
  - Скопируйте csv в HDFS
  ```
  sudo -u hdfs hdfs dfs -mkdir -p /hive_data/salary
  sudo -u hdfs hdfs dfs -put /tmp/sample*  /hive_data/salary
  sudo -u hdfs hdfs dfs -chown -R hive:hive /hive_data/
  ```
  - Теперь создайте таблицу Hive в базе данных по умолчанию с помощью данной процедуры 
    - Запуск beeline shell с узла, на котором установлен Hive: 
```
beeline -n hive -u "jdbc:hive2://localhost:10000/default"
```

  - В отклике beeline, заупстите:
    
```
CREATE EXTERNAL TABLE sample_07 (
code string ,
description string ,  
total_emp int ,  
salary int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile
LOCATION '/hive_data/salary';
```
```
CREATE EXTERNAL TABLE sample_08 (
code string ,
description string ,  
total_emp int ,  
salary int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile
LOCATION '/hive_data/salary';
```
```
!q
```

- Обратите внимание, что в строке подключения JDBC для подключения к незащищенному Hive, когда он работает в транспортном режиме по умолчанию (то есть в двоичном):
  - port - 10000
  - Принцип Kerberos не нужен 

- Это изменится после того, как мы:
  - включим kerberos
  - настроим Hive на транспортный режим http (чтобы пройти через Knox)
    
### Зачем нужна безопасность?


##### Доступ HDFS в незащищенном кластере

- На вашем незащищенном кластере попробуйте получить доступ к ограниченному каталогу в HDFS
```
hdfs dfs -ls /tmp/hive   
## this should fail with Permission Denied
```

- Теперь попробуйте еще раз после установки HADOOP_USER_NAME env var
```
export HADOOP_USER_NAME=hdfs
hdfs dfs -ls /tmp/hive   
## this shows the file listing!
```
- Отключите env var, и он снова потерпит неудачу
```
unset HADOOP_USER_NAME
hdfs dfs -ls /tmp/hive  
```

##### Доступ к WebHDFS на незащищенном кластере

- С *узла на котором запущен NameNode*, сделайте запрос WebHDFS, используя следующую команду:
```
curl -sk -L "http://$(hostname -f):50070/webhdfs/v1/user/?op=LISTSTATUS"
```

- Обратите внимание, что при отсутствии Knox он переходит через HTTP (не HTTPS) на порт 50070 без требования учетных данных.

##### Доступ к Web UI на незащищенном кластере

- Из уведомления Ambari вы можете открывать веб-интерфейсы без аутентификации.
  - HDFS > Quicklinks (Быстрые ссылки)> NameNode UI
  - Mapreduce > Quicklinks (Быстрые ссылки > JobHistory UI
  - YARN > Quicklinks (Быстрые ссылки > ResourceManager UI
  
- Все это лишь напоминает, почему Kerberos (и другие средства защиты) необходимы в Hadoop :)


-----------------------------

# Лабораторная работа 2

### Обзор юзкейса 

Пример использования: у клиента есть существующий кластер и он хочет, чтобы вы для него защитили.

- Текущая настройка:
  - У клиента есть несколько организационных групп (например, отдел продаж, отдела кадров, юридический отдел), в которые входят бизнес-пользователи (sales1, hr1, legal1 и т. д.) и hadoopadmin.
  - Эти группы и пользователи определены в Active Directory (AD) в рамках собственной организационной единицы (OU), называемой CorpUsers. 
  - В AD созданы пустые организационные единицы для хранения участников / узлов hadoop (HadoopServices, HadoopNodes).
  - У пользователя Hadoopadmin есть учетные данные администратора с делегированным контролем на «Создание, удаление и управление учетными записями пользователей» в вышеуказанных подразделениях.
  - Кластер Hadoop с HDP уже настроен с использованием Ambari (включая HDFS, YARN, Hive, Hbase, Solr, Zookeeper)
  
- Задачи:
  - Интегрировать Ambari с AD, чтобы hadoopadmin мог администрировать кластер
  - Интегрировать ОС узлов Hadoop с AD, чтобы бизнес-пользователи узнавали и могли отправлять задания Hadoop
  - Включить kerberos - для защиты кластера и включения аутентификации.
  - Установить Ranger и включить плагины Hadoop, чтобы администратор мог настраивать политики авторизации и проверять аудит компонентов Hadoop.
  - Установить Ranger KMS и включить шифрование HDFS, чтобы иметь возможность создавать зоны шифрования.
  - Зашифровать резервные каталоги Hive для защиты таблиц Hive
  - Настроить политики Ranger для:
    - Чтобы к директории Protect / sales HDFS доступ имела только группа продаж
    - Защитить таблицу продаж Hive - чтобы только группа продаж имела к ней доступ
      - Детализированный доступ: продавцы должны иметь доступ только к коду, столбцам описания в default.sample_07, но только к строкам, где total_emp <5000. Также следует замаскировать столбец total_emp
    - Защитить таблицу продаж HBase - чтобы только группа продаж имела к ней доступ
  - Установить Knox и интегрировать с AD для защиты периметра и предоставления клиентам доступа к API без работы с Kerberos
  - Включить обозреватель Ambari для работы в защищенном кластере

Мы проведем серию лабораторных работ и шаг за шагом достигнем всех вышеперечисленных целей.



## 1. Регистрация узлов кластера как клиента IPA
- Выполните на *всех узлах кластера HDP* (заменить $INTERNAL_IP_OF_IPA)

```
echo "$INTERNAL_IP_OF_IPA ipa.us-west-2.compute.internal ipa" >> /etc/hosts
```

- Установите пакеты yum
```
sudo yum install -y ipa-client
```

- Обновите /etc/resolv.conf (заменить INTERNAL_IP_OF_IPA)
```
mv /etc/resolv.conf /etc/resolv.conf.bak 
echo "search us-west-2.compute.internal" > /etc/resolv.conf
echo "nameserver $INTERNAL_IP_OF_IPA" >> /etc/resolv.conf
```
- Установите клиент IPA

  ```	
	sudo ipa-client-install \
	--server=ipa.us-west-2.compute.internal \
	--realm=US-WEST-2.COMPUTE.INTERNAL \
	--domain=us-west-2.compute.internal \
	--mkhomedir \
	--principal=admin -w BadPass#1 \
	--unattended
  ```
Примечание: иногда требуется перезапуск dbus `service dbus restart`

- Убедитесь, что вы не видите ниже сообщение из вывода предыдущей команды
```
Missing A/AAAA record(s) for host xxxxxxxxx
```

- Если вы это сделали, удалите и попробуйте снова:
```
service dbus restart
sudo ipa-client-install --uninstall
```

- Обратите внимание, что при изменении DNS, возможно, узел не сможет подключиться к общедоступному Интернету. Когда вам нужно это сделать то (например, для установки yum) вы можете временно вернуть обратно /etc/resolv.conf.bak


### Проверка

- При регистрации в качестве клиента IPA-сервера SSSD настраивается автоматически. Итак, теперь хост распознает пользователей, определенных в IPA.
```
id hadoopadmin
```

- Вы также можете пройти аутентификацию и получить билет Kerberos (пароль - BadPass # 1)
```
kinit -V hadoopadmin
```

---



# 2. Защита Ambari с помощью настройки безопасности ambari-сервера

Давайте использовать сертификат, созданный FreeIPA для вариантов 1 и 4 в `ambari-server setup-security`
	
  ```
Security setup options...
===========================================================================
Choose one of the following options:
  *[1] Enable HTTPS for Ambari server.
  *[2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  *[4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
  ```

**Подготовка:** Создайте сертификаты на всех хостах ipa-client (запустите это на каждом узле)

Убедитесь, что SELinux не в принудительном режиме, иначе запрос сертификата от имени пользователя root с билетом Kerberos администратора будет отклонен системой, и сертификат не будет создан. 

```
getenforce
# Если результат «Enforcing», выполните следующее
sudo su
setenforce 0
```

Получите билет kerberos как **admin**(или привилегированный пользователь IPA) и запросите пару сертификатов x509, сохраненную как «host.key» и «host.crt» на каждом хосте. 

```
echo BadPass#1 | kinit admin 
mkdir /etc/security/certificates/
cd /etc/security/certificates/
ipa-getcert request -v -f /etc/security/certificates/host.crt -k /etc/security/certificates/host.key
```

Перечислите каталог для проверки создания сертификатов. 

```
[root@demo certificates]# ls -ltr /etc/security/certificates/
total 8
-rw------- 1 root root 1704 Sep 30 04:56 host.key
-rw------- 1 root root 1724 Sep 30 04:56 host.crt
```


### 2.1 Включение HTTPS для сервера Ambari
Если вы используете Knox на этом хосте (что крайне не рекомендуется), изменение порта по умолчанию с 8443 позволит избежать конфликта портов. 

```
[root@demo ~]$ambari-server setup-security
 
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================

# Включение SSL
Enter choice, (1-5): 1
Do you want to configure HTTPS [y/n] (y)? y
SSL port [8443] ? 8444
Enter path to Certificate: /etc/security/certificates/host.crt
Enter path to Private Key: /etc/security/certificates/host.key
Please enter password for Private Key: changeit
```

### Проверка
Перезагрузите амбари-сервера. Заверните Ambari на новом https-порту **без** указания флага «-k».
```
[root@demo ~]$ curl -u admin:"password" https://`hostname -f`:8444/api/v1/clusters
```

### 2.2 Зашифровка паролей, хранящихся в файле ambari.properties.
Этот шаг необходим для мастера Kerberos для сохранения учетных данных KDC (`hadoopadmin`). Это также необходимо для сохранения пароля `ldapbind` без которого включение ldaps в Ambari 2.7.1, похоже, вызывает некоторые проблемы. 

```
[root@demo ~]# ambari-server setup-security
Using python  /usr/bin/python
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
Enter choice, (1-5): 2
Please provide master key for locking the credential store:
Re-enter master key:
Do you want to persist master key. If you choose not to persist, you need to provide the Master Key while starting the ambari server as an env variable named AMBARI_SECURITY_MASTER_KEY or the start will prompt for the master key. Persist [y/n] (y)? y
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup-security' completed successfully.
```


### 2.3 Настройка truststore.

Предварительная настройка доверенного хранилища и перезапуск Ambari, похоже, делают интеграцию ldap более гармоничной. 
Ambari может использовать хранилище доверенных сертификатов `/etc/pki/java/cacerts` управляемое клиентами IPA на хостах. Это хранилище доверенных сертификатов содержит общедоступные CA вместе с CA IPA, которые должны быть единственными необходимыми сертификатами.

```
# Пример имени хоста ipa: ipa.us-west-2.compute.internal

[root@demo ~]# /usr/java/default/bin/keytool -list \
-keystore /etc/pki/java/cacerts \
-v -storepass changeit | grep ipa

Alias name: hortonworks.comipaca
   accessLocation: URIName: http://ipa-ca.us-west-2.compute.internal/ca/ocsp
```


```
[root@demo certificates]# ambari-server setup-security
Using python  /usr/bin/python
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
Enter choice, (1-5): 4
Do you want to configure a truststore [y/n] (y)? y
TrustStore type [jks/jceks/pkcs12] (jks):
Path to TrustStore file :/etc/pki/java/cacerts
Password for TrustStore: changeit
Re-enter password: changeit
Ambari Server 'setup-security' completed successfully.
```


### 2.4 Перезапуск Ambari, чтобы изменения вступили в силу.

```
ambari-server restart
```

 

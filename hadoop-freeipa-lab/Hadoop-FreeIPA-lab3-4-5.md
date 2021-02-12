# 3. Активация Kerberos в кластере

Включите Kerberos для служб кластера с помощью мастера в Ambari, расположенного в меню Cluster Admin в нижней левой панели навигации. https://demo.us-west-2.compute.internal:8444/#/main/admin/kerberos

![](https://habrastorage.org/webt/e9/fe/c-/e9fec-yf8hqr4cz-nowrvhctv-i.png)

На этом этапе все необходимые требования выполнены. Группа участников, управляемая амбари, не требуется, а политики истечения срока действия паролей не влияют на служебные ключевые вкладки, поскольку им не были присвоены пароли. Срок действия пароля пользователей `hadoopadmin` и `ldapbind` истечет, и его необходимо будет изменить через 90 дней (вместе с остальными пользователями). Смотрите документацию для объяснения https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/authentication-with-kerberos/content/kerberos_optional_use_an_existing_ipa.html 


- KDC хост: `ipa.us-west-2.compute.internal`
- Имя области: `US-WEST-2.COMPUTE.INTERNAL`
- Домен: `us-west-2.compute.internal`

- Kadmin хост: `ipa.us-west-2.compute.internal`
- Главный Admin: `hadoopadmin`
- Пароль Admin: `BadPass#1`
- Сохранить учетные данные администратора: true

![](https://habrastorage.org/webt/tt/j0/2r/ttj02rhztb3vfpusevqlomysnqe.png)

![](https://habrastorage.org/webt/gi/tj/dy/gitjdyspxs0ual1rvkcrb5_rm-o.png)

Полезный интерфейс командной строки для проверки вновь созданных Принципов обслуживания:

	#Usage: ipa service-show <principal>
	[root@demo ~]# ipa service-show spark/demo.us-west-2.compute.internal@US-WEST-2.COMPUTE.INTERNAL
	  Principal name: spark/demo.us-west-2.compute.internal@US-WEST-2.COMPUTE.INTERNAL
	  Principal alias: spark/demo.us-west-2.compute.internal@US-WEST-2.COMPUTE.INTERNAL
	  Keytab: True
	  Managed by: demo.us-west-2.compute.internal

---


# 4. Активация LDAP для Ambari

#### Советы FreeIPA по определению свойств поиска LDAP

- Клиенты IPA содержатся в `/etc/ipa/default.conf` с различными свойствами сервера ldap

		[root@demo ~]# cat /etc/ipa/default.conf 
		basedn = dc=us-west-2,dc=compute,dc=internal
		realm = US-WEST-2.COMPUTE.INTERNAL
		domain = us-west-2.compute.internal
		server = ipa.us-west-2.compute.internal

- Определение допустимых атрибутов **user**  (posixaccount, uid, и так далее):
		
	
		ipa user-show hadoopadmin --raw --all
	
- Определение допустимых атрибутов **group** (posixgroup, member, memberUid, etc)

		ipa group-show admins --raw --all
	
- Проверка учетной записи ldapbind и поисковой базы через `ldapsearch`

		[root@demo ~]# yum install -y openldap-clients 
		
		# Проверка свойств привязки ldap
		AM_LDAP_SEARCHBASE="cn=accounts,dc=us-west-2,dc=compute,dc=internal"
		AM_LDAP_BINDDN="uid=ldapbind,cn=users,cn=accounts,dc=us-west-2,dc=compute,dc=internal"
		AM_LDAP_BINDDN_PW="BadPass#1"
		AM_LDAP_URL=ldaps://ipa.us-west-2.compute.internal:636
		
		# Найдите действительный uid и убедитесь, что searchbase, bind dn и ldapurl разрешаются правильно
		[root@demo ~]# ldapsearch -D ${AM_LDAP_BINDDN} \
		-w ${AM_LDAP_BINDDN_PW} \
		-b ${AM_LDAP_SEARCHBASE} \
		-H ${AM_LDAP_URL} uid=hadoopadmin
		
		# Хвостовые результаты действительного ldapsearch для одного uid:
		numResponses: 2
		numEntries: 1


### 4.1 Активация LDAP для сервера Ambari

Ambari 2.7.1 предлагает опцию CLI в `ambari-server setup-ldap` для выбора типа ldap в качестве IPA, который пытается установить некоторые из значений по умолчанию, необходимых для интеграции. Кажется, по-прежнему есть несколько проблем, поэтому нужно изменить некоторые из значений по умолчанию.

На хосте амбари-сервера:

```
[root@demo certificates]# ambari-server setup-ldap
Currently 'no auth method' is configured, do you wish to use LDAP instead [y/n] (y)?  
Please select the type of LDAP you want to use (AD, IPA, Generic LDAP):IPA
Primary LDAP Host (ipa.ambari.apache.org): ipa.us-west-2.compute.internal
Primary LDAP Port (636):
Secondary LDAP Host <Optional>:
Secondary LDAP Port <Optional>:
Use SSL [true/false] (true):
Do you want to provide custom TrustStore for Ambari [y/n] (y)?
TrustStore type [jks/jceks/pkcs12] (jks):
Path to TrustStore file (/etc/pki/java/cacerts):
Password for TrustStore: changeit
Re-enter password: changeit
User object class (posixUser): posixaccount
User ID attribute (uid):
Group object class (posixGroup):
Group name attribute (cn):
Group member attribute (memberUid): member
Distinguished name attribute (dn):
Search Base (dc=ambari,dc=apache,dc=org): cn=accounts,dc=us-west-2,dc=compute,dc=internal
Referral method [follow/ignore] (follow):
Bind anonymously [true/false] (false):
Bind DN (uid=ldapbind,cn=users,cn=accounts,dc=ambari,dc=apache,dc=org): uid=ldapbind,cn=users,cn=accounts,dc=us-west-2,dc=compute,dc=internal
Enter Bind DN Password: BadPass#1
Confirm Bind DN Password: BadPass#1
Handling behavior for username collisions [convert/skip] for LDAP sync (skip):
Force lower-case user names [true/false]:
Results from LDAP are paginated when requested [true/false]:
```
- Затем введите учетные данные Ambari (admin / BadPass # 1)

### 4.2 Синхронизация пользователей
Пользователи LDAP должны быть синхронизированы путем вызова команды на хосте сервера Ambari. Добавления пользователей и групповые ассоциации, созданные на сервере LDAP, не будут автоматически передаваться в Ambari, только при вызове этой команды. 

```
[root@demo ~]# ambari-server sync-ldap --all
Using python  /usr/bin/python
Syncing with LDAP...
Enter Ambari Admin login: admin
Enter Ambari Admin password:

Fetching LDAP configuration from DB.
Syncing all...

Completed LDAP Sync.
Summary:
  memberships:
    removed = 0
    created = 16
  users:
    skipped = 1
    removed = 0
    updated = 0
    created = 15
  groups:
    updated = 0
    removed = 0
    created = 26

Ambari Server 'sync-ldap' completed successfully.
```

- Теперь перезапустите амбари-сервер

### 4.2.1 Проверка объединения групп пользователей в Ambari

Войдите в Ambari как администратор и перейдите в Manage Ambari> Users. Пример пользователя / группы из этой лабораторной работы:

![](https://habrastorage.org/webt/zz/bw/mp/zzbwmpmut_-tgzdup6mehwp1e6g.png)


### Включение аутентификации SPNEGO для Hadoop

- Это необходимо для защиты webUIs компонентов Hadoop  (e.g. Namenode UI, JobHistory UI, Yarn ResourceManager UI etc...)
- Начиная с HDP 3.0, об этом заботятся как часть настройки Kerberos с помощью мастера безопасности Ambari.
- Теперь, когда вы попытаетесь открыть любой из веб-интерфейсов, как показано ниже, вы получите `401: Authentication required`
  - HDFS: Namenode UI
  - Mapreduce: Job history UI
  - YARN: Resource Manager UI

------------------

# Лабораторная работа 5

## Установка Ranger 

Цель: в этой лабораторной работе мы установим Apache Ranger через Ambari и настроим плагины Ranger для компонентов Hadoop: HDFS, Hive, Hbase, YARN, Knox. Мы также включим аудит Ranger для Solr и HDFS.

### Требования к Ranger 

##### Создать и подтвердить пользователя MySQL root

Подготовить базу данных MySQL для использования Ranger.

- Выполните эти шаги на узле, где расположен MySQL / Hive. Чтобы найти это, вы можете :
  - использовать Ambari UI или
  - просто запустите `mysql`на каждом узле: если он выдает `mysql: command not found`, переходите к следующему узлу.

- `sudo mysql`
- Выполните следующее в оболочке MySQL, чтобы создать «пользователя root Ranger DB» в MySQL. Ambari будет использовать этого пользователя для создания пользователя rangeradmin.
```sql
CREATE USER 'root'@'%';
GRANT ALL PRIVILEGES ON *.* to 'root'@'%' WITH GRANT OPTION;
SET PASSWORD FOR 'root'@'%' = PASSWORD('BadPass#1');
SET PASSWORD = PASSWORD('BadPass#1');
FLUSH PRIVILEGES;
exit
```

- Подтвердите пользователя MySQL: `mysql -u root -h $(hostname -f) -p -e "select count(user) from mysql.user;"`
  - Вывод должен быть простым подсчетом. 
  - В случае появления ошибок проверьте предыдущий шаг на наличие ошибок. 
  - Если вы столкнулись с ошибкой ниже, модифицируйте /etc/my.conf, удалив `skip-grant-tables` а затем перезапустите службу с помощью by `service mysqld restart`
  

`ERROR 1290 (HY000): The MySQL server is running with the --skip-grant-tables option so it cannot execute this statement`

  - Если это не помогло, попробуйте вместо этого создать пользователя admin. Если вы это сделаете, не забудьте ввести admin insted of root при появлении запроса «Ranger DB root User» в Ambari.

##### Подготовьте Ambari для MySQL
- Запустите его на ноде Ambari
- Добавьте MySQL JAR в Ambari:
  - `sudo ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar`
    - Если файл отсутствует, он доступен в RHEL / CentOS: `sudo yum -y install mysql-connector-java`




###### Установка Solr для аудита Ranger 

- Начиная с HDP 2.5, если вы развернули службу Ambari Infra, ее можно использовать для аудита Ranger.
- **Перед началом установки Ranger убедитесь, что служба Ambari Infra установлена и запущена**

- *TODO*: добавить шаги для установки / настройки панели управления Banana для Ranger Audits

## Установка Ranger 

##### Установите Ranger

- Запустите мастер Ambari 'Add Service' и выберите Ranger.

- Когда будет предложено, где его установить, выберите удобное место.

- Во всплывающих окнах требований Ranger вы можете установить флажок и продолжить, поскольку мы уже выполнили необходимые шаги.

- На странице мастера настройки служб есть ряд вкладок, которые необходимо настроить, как показано ниже.

- Пройдите по каждой вкладке конфигурации Ranger, внося указанные изменения ниже:

1. Вкладка Ranger Admin:
  - Хост Ranger DB = полное доменное имя хоста, на котором работает Mysql (например, ip-172-30-0-242.us-west-2.compute.internal)

  - Введите пароль: BadPass#1

  - Нажмите кнопку "Test Connection".

  ![](https://habrastorage.org/webt/kj/ct/yu/kjctyuwqvpuycyenurhlpljrmm0.png)

  

  - ![](https://habrastorage.org/webt/hb/ec/sj/hbecsjirs1novfhasazbhh10tqm.png)

2. Вкладка Ranger User info 
  - 'Sync Source' = LDAP/AD 
  - Подвкладка User configs
    - Введите пароль: BadPass#1


3. Вкладка Ranger User info  
  - Подвкладка User configs 
    - База поиска пользователей = `cn=accounts,dc=us-west-2,dc=compute,dc=internal`
    - Фильтр поиска пользователей = `(objectcategory=posixaccount)`


4. Вкладка Ranger User info  
  - Подвкладка Group configs 
    - Убедитесь, что групповая синхронизация (Group sync) отключена
  ![](https://habrastorage.org/webt/lo/-r/ds/lo-rdsr_2bwe6y91874dur3iqv8.png)
    
      

5. Вкладка Ranger plugins 
  - Включите все плагины
![](https://habrastorage.org/webt/8v/r_/cq/8vr_cq0uzys7dombvcfptclwtns.png)

6. Вкладка Ranger Audits 
  - SolrCloud = ON
![](https://habrastorage.org/webt/si/ki/jr/sikijrmymuukmnha1yxqg7ulge0.png)
![](https://habrastorage.org/webt/0z/am/po/0zampovnmeonrhg8m_n-mac6t6g.png)

7. Вкладка Advanced 

  - Никаких изменений не требуется (пока не нужно настраивать аутентификацию Ranger в AD)
![](https://habrastorage.org/webt/w-/vg/uw/w-vguwvp4k_nfotvvatfl2b1kku.png)

- Перейдите по ссылке Next > Proceed Anyway 
  
- При появлении запроса на странице Configure Identities, вам, возможно, придется ввести учетные данные администратора AD.
  - Главный Admin: `hadoopadmin@LAB.HORTONWORKS.NET`
  - Пароль Admin: BadPass#1
  - Обратите внимание, что теперь вы можете сохранить учетные данные администратора. Отметьте и этот флажок
  ![](https://habrastorage.org/webt/sa/ev/kv/saevkvp5wguhc3s6uzmtr_vuhm4.png)
  
- Перейдите в Next > Deploy чтобы установить Ranger

- После установки перезапустите компоненты, требующие перезапуска. (например, HDFS, YARN, Hive и.т.д.)

- (Необязательно) В случае сбоя (обычно вызванного неправильным вводом полного доменного имени узлов Mysql в конфигурации выше) удалите службу Ranger из Ambari и повторите попытку.


8 -(Необязательно) Активация условий запрета в Ranger 

Условие отказа в политиках по умолчанию является необязательным и должно быть включено для использования.

- В Ambari>Ranger>Configs>Advanced>Custom ranger-admin-site, добавьте : 
`ranger.servicedef.enableDenyAndExceptionsInPolicies=true`

- Перезапустите Ranger

https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.1/bk_security/content/about_ranger_policies.html


##### Проверка Ranger

- Откройте интерфейс Ranger в http://RANGERHOST_PUBLIC_IP:6080 используя admin/admin
- Убедитесь, что репозитории для HDFS, YARN, Hive, HBase, Knox отображаются на вкладке Access Manager.'
![](https://habrastorage.org/webt/ay/zr/7p/ayzr7pqr3wddnpqi9haos6cogh4.png)

- Убедитесь, что аудит отображается на вкладке 'Audit' > 'Access' 
![](https://habrastorage.org/webt/qf/-r/qc/qf-rqcuazbh8swnnalphqc52rfw.png)
- Если аудит здесь не отображается, возможно, вам потребуется перезапустить Ambari Infra Solr из Ambari
  - В случае, если аудиты все еще не появляются, а Ranger жалуется, что коллекция аудита не найдена: попробуйте [эти шаги](https://community.hortonworks.com/articles/96618/how-to-clean-up-recreate-collections-on-ambari-inf.html)
  
- Убедитесь, что плагины для HDFS, YARN, Hive и т. д. отображаются на вкладке 'Audit' > 'Plugins'  
![](https://habrastorage.org/webt/kg/dl/et/kgdletub-kmhq8o5_qfttau9tvc.png)

- Убедитесь, что синхронизация пользователей / групп из AD в Ranger работает, щелкнув 'Settings' > 'Users/Groups tab' в пользовательском интерфейсе Ranger и убедитесь, что пользователи / группы AD присутствуют.
![](https://habrastorage.org/webt/ha/be/xw/habexwklllvgkeslba4lstpffwc.png)

- Подтвердите работу аудита HDFS, запросив каталог аудита в HDFS:

```
#### 1 аутентификацияя
используйтесь PASSWORD=BadPass#1

#определения имени кластера
output=`curl -u hadoopadmin:$PASSWORD -k -i -H 'X-Requested-By: ambari'  https://localhost:8443/api/v1/clusters`
cluster=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

echo $cluster
## эта команда должна показать имя вашего кластера

## если нет, вы можете вручную установить его, как показано ниже
## cluster=Security-HWX-LabTesting-XXXX

#затем kinit как hdfs, используя headless keytab и основное имя
sudo -u hdfs kinit -kt /etc/security/keytabs/hdfs.headless.keytab "hdfs-${cluster,,}"
    
#### 2 чтение аудит-директории hdfs 
sudo -u hdfs hdfs dfs -cat /ranger/audit/hdfs/*/*
```

- Подтвердите работу аудита Solr, запросив Solr REST API *с любого узла solr* - SKIP 
```
curl "http://localhost:6083/solr/ranger_audits/select?q=*%3A*&df=id&wt=csv"
```

- Подтвердите, что панель управления Banana начала показывать аудит HDFS - SKIP
http://PUBLIC_IP_OF_SOLRLEADER_NODE:6083/solr/banana/index.html#/dashboard

![](https://habrastorage.org/webt/f-/hb/yb/f-hbybffhfrlbbp49trxjwzzwsc.png)


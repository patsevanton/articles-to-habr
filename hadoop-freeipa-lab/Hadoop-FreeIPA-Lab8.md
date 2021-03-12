# Лабораторная работа 8

## Knox 

Цель: в этой лабораторной работе мы настроим Apache Knox для аутентификации AD и будем делать запросы WebHDFS, Hive через Knox (после установки соответствующих политик авторизации Ranger для доступа).

### Конфигурация Knox  

#### Конфигурация Knox для аутентификации AD

- Выполняйте эти шаги на узле, на котором ранее был установлен Knox.

- Чтобы настроить Knox для аутентификации AD, нам нужно ввести свойства, связанные с AD, в топологию xml через Ambari.

- Проблема в том, что нам необходимо ввести пароль привязки LDAP, но мы не хотим, чтобы он отображался в виде простого текста в конфигурациях Ambari.

- Какое решение? Создайте псевдоним хранилища ключей для пользователя ldap manager (который вы позже передадите в топологию через свойство systemUsername)

   - Прочтите пароль для использования в следующей команде (вам будет предложено ввести пароль и сохранить его в переменной среды knoxpass). Введите BadPass # 1:
   ```
   read -s -p "Password: " knoxpass
   ```
   - Это удобный способ установить переменную env без сохранения команды в истории
   - Создайте псевдоним пароля для Knox под названием knoxLdapSystemPassword.
    ```
    sudo -u knox /usr/hdp/current/knox-server/bin/knoxcli.sh create-alias knoxLdapSystemPassword --cluster default --value ${knoxpass}
    unset knoxpass
    ```

- Теперь давайте настроим Knox на использование нашей AD для аутентификации. Замените содержимое ниже в Ambari> Knox> Config> Advanced topology. 
  - Как узнать, какие конфиги были изменены по сравнению с настройками по умолчанию? 
    - Конфигурации по умолчанию остаются с отступом ниже
    - Конфигурации, которые были добавлены / изменены, не имеют отступа
```
<topology>
 
            <gateway>
 
                <provider>
                    <role>authentication</role>
                    <name>ShiroProvider</name>
                    <enabled>true</enabled>
                    <param>
                        <name>sessionTimeout</name>
                        <value>30</value>
                    </param>
                    <param>
                        <name>main.ldapRealm</name>
                        <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value>
                    </param>
 
<!-- changes for AD/user sync -->
 
<param>
    <name>main.ldapContextFactory</name>
    <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory</value>
</param>
 
<!-- main.ldapRealm.contextFactory needs to be placed before other main.ldapRealm.contextFactory* entries  -->
<param>
    <name>main.ldapRealm.contextFactory</name>
    <value>$ldapContextFactory</value>
</param>
 
<!-- IPA url -->
<param>
    <name>main.ldapRealm.contextFactory.url</name>
    <value>ldap://ipa.us-west-2.compute.internal:389</value>
</param>
 
<!-- system user -->
<param>
    <name>main.ldapRealm.contextFactory.systemUsername</name>
    <value>uid=hadoopadmin,cn=users,cn=accounts,dc=us-west-2,dc=compute,dc=internal</value>
</param>
 
<!-- pass in the password using the alias created earlier -->
<param>
    <name>main.ldapRealm.contextFactory.systemPassword</name>
    <value>${ALIAS=knoxLdapSystemPassword}</value>
</param>
 
                    <param>
                        <name>main.ldapRealm.contextFactory.authenticationMechanism</name>
                        <value>simple</value>
                    </param>
                    <param>
                        <name>urls./**</name>
                        <value>authcBasic</value>
                    </param>
 
<!--  AD groups of users to allow -->
<param>
    <name>main.ldapRealm.searchBase</name>
    <value>cn=accounts,dc=us-west-2,dc=compute,dc=internal</value>
</param>
<param>
    <name>main.ldapRealm.userObjectClass</name>
    <value>posixAccount</value>
</param>
<param>
    <name>main.ldapRealm.userSearchAttributeName</name>
    <value>uid</value>
</param>
 
<!-- changes needed for group sync-->
<param>
    <name>main.ldapRealm.authorizationEnabled</name>
    <value>true</value>
</param>
<param>
    <name>main.ldapRealm.groupSearchBase</name>
    <value>cn=accounts,dc=us-west-2,dc=compute,dc=internal</value>
</param>
<param>
    <name>main.ldapRealm.groupObjectClass</name>
    <value>posixGroup</value>
</param>
<param>
    <name>main.ldapRealm.groupIdAttribute</name>
    <value>cn</value>
</param>
 
 
                </provider>
 
                <provider>
                    <role>identity-assertion</role>
                    <name>Default</name>
                    <enabled>true</enabled>
                </provider>
 
                <provider>
                    <role>authorization</role>
                    <name>XASecurePDPKnox</name>
                    <enabled>true</enabled>
                </provider>
 
<!--
  Knox HaProvider for Hadoop services
  -->
<provider>
     <role>ha</role>
     <name>HaProvider</name>
     <enabled>true</enabled>
     <param>
         <name>OOZIE</name>
         <value>maxFailoverAttempts=3;failoverSleep=1000;enabled=true</value>
     </param>
     <param>
         <name>HBASE</name>
         <value>maxFailoverAttempts=3;failoverSleep=1000;enabled=true</value>
     </param>
     <param>
         <name>WEBHCAT</name>
         <value>maxFailoverAttempts=3;failoverSleep=1000;enabled=true</value>
     </param>
     <param>
         <name>WEBHDFS</name>
         <value>maxFailoverAttempts=3;failoverSleep=1000;maxRetryAttempts=300;retrySleep=1000;enabled=true</value>
     </param>
     <param>
        <name>HIVE</name>
        <value>maxFailoverAttempts=3;failoverSleep=1000;enabled=true;zookeeperEnsemble=machine1:2181,machine2:2181,machine3:2181;
       zookeeperNamespace=hiveserver2</value>
     </param>
</provider>
<!--
  END Knox HaProvider for Hadoop services
  -->
 
 
            </gateway>
 
            <service>
                <role>NAMENODE</role>
                <url>hdfs://{{namenode_host}}:{{namenode_rpc_port}}</url>
            </service>
 
            <service>
                <role>JOBTRACKER</role>
                <url>rpc://{{rm_host}}:{{jt_rpc_port}}</url>
            </service>
 
            <service>
                <role>WEBHDFS</role>
                <url>http://{{namenode_host}}:{{namenode_http_port}}/webhdfs</url>
            </service>
 
            <service>
                <role>WEBHCAT</role>
                <url>http://{{webhcat_server_host}}:{{templeton_port}}/templeton</url>
            </service>
 
            <service>
                <role>OOZIE</role>
                <url>http://{{oozie_server_host}}:{{oozie_server_port}}/oozie</url>
            </service>
 
            <service>
                <role>WEBHBASE</role>
                <url>http://{{hbase_master_host}}:{{hbase_master_port}}</url>
            </service>
 
            <service>
                <role>HIVE</role>
                <url>http://{{hive_server_host}}:{{hive_http_port}}/{{hive_http_path}}</url>
            </service>
 
            <service>
                <role>RESOURCEMANAGER</role>
                <url>http://{{rm_host}}:{{rm_port}}/ws</url>
            </service>
        </topology>
```

- Затем перезапустите Knox через Ambari.

#### Конфигурация HDFS для Knox

-  - Задайте Hadoop разрешить нашим пользователям доступ к Knox с любого узла кластера. Измените указанные ниже свойства в Ambari> HDFS> Config> Custom core-site (группа пользователей уже должна быть частью групп, поэтому просто добавьте остальные)
  - adoop.proxyuser.knox.groups=users,hadoop-admins,sales,hr,legal
  - hadoop.proxyuser.knox.hosts=*
    - (лучше было бы поставить разделенный запятыми список полных доменных имен хостов)
  - Теперь перезапустите HDFS
  - Без этого шага вы увидите ошибку, как показано ниже, когда вы позже запустите запрос WebHDFS:
  ```
   org.apache.hadoop.security.authorize.AuthorizationException: User: knox is not allowed to impersonate sales1"
  ```


#### Конфигурация Ranger для WebHDFS через Knox

- Настройте политику Knox для группы продаж для WEBHDFS:

- Войдите в Ranger> Access Manager> KNOX> щелкните ссылку имени кластера> Add new policy

  - Policy name: webhdfs
  - Topology name: default
  - Service name: WEBHDFS
  - Group permissions: sales 
  - Permission: check Allow
  - Нажмите Add

  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-knox-webhdfs-policy.png)

#### Упражнения WebHDFS поверх Knox 

- Теперь мы можем отправить несколько запросов в WebHDFS через Knox, чтобы проверить его работу. Мы будем использовать curl со следующими аргументами:
  - i (также известный как –include): используется для вывода информации заголовка ответа HTTP. Это будет важно, когда содержимое заголовка HTTP Location требуется для последующих запросов.
  - -k (также известный как –insecure) используется, чтобы избежать проблем, связанных с использованием демонстрационных сертификатов SSL.
  - -u (также известный как –user) используется для предоставления учетных данных, которые будут использоваться, когда шлюз запрашивает клиента.
  - Обратите внимание, что в большинстве примеров для простоты не используются функции cookie cURL. Поэтому мы будем передавать учетные данные пользователя с каждым запросом curl для аутентификации.

- *С хоста, на котором запущен Knox*, отправьте приведенный ниже запрос curl на порт 8443, на котором запущен Knox, чтобы запустить команду ls в каталоге / / dir в HDFS:
```
curl -ik -u sales1:BadPass#1 https://localhost:8443/gateway/default/webhdfs/v1/?op=LISTSTATUS
```
  - Это должно вернуть объект json, содержащий список каталогов / файлов, расположенных в корневом каталоге, и их атрибуты
- Чтобы избежать передачи пароля в командной строке, вы можете передать только имя пользователя (чтобы пароль не сохранялся в истории оболочки). В этом случае вам будет предложено ввести пароль.  
```
curl -ik -u sales1 https://localhost:8443/gateway/default/webhdfs/v1/?op=LISTSTATUS

## enter BadPass#1
```

- Для остальных примеров ниже, для простоты, мы передаем пароль в командной строке, но не стесняйтесь удалять пароль и вводить его вручную при появлении запроса.
- Попробуйте тот же запрос, что и hr1, и обратите внимание, что он не выполняется с `ошибкой 403 Forbidden` :
  - Это было ожидаемо, поскольку в приведенной выше политике мы разрешили группе продаж только доступ к WebHDFS через Knox.
```
curl -ik -u hr1:BadPass#1 https://localhost:8443/gateway/default/webhdfs/v1/?op=LISTSTATUS
```

- Обратите внимание, что для выполнения запросов через Knox билет Kerberos не требуется - пользователь аутентифицируется, передавая учетные данные AD / LDAP.

- Проверьте в Ranger Audits, чтобы подтвердить, что запросы были проверены.:

  - Ranger > Audit > Service type: KNOX

  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-knox-webhdfs-audit.png)


- Другие возможности доступа к WebHDFS с помощью Knox

  - A. Используйте cookie, чтобы сделать запрос без передачи учетных данных
    - Когда вы запускали предыдущий запрос curl, он перечислял заголовки HTTP как часть вывода. Один из заголовков будет «Set Cookie».
    - Например `Set-Cookie: JSESSIONID=xxxxxxxxxxxxxxx;Path=/gateway/default;Secure;HttpOnly`
    - Вы можете передать значение из своей настройки и сделать запрос, не передавая учетные данные:
      - Убедитесь, что вы скопировали JSESSIONID из запроса, который работал (то есть из sales1, а не hr1)
  ```
  curl -ik --cookie "JSESSIONID=xxxxxxxxxxxxxxx;Path=/gateway/default;Secure;HttpOnly" -X GET https://localhost:8443/gateway/default/webhdfs/v1/?op=LISTSTATUS
  ```
  
  - Б. Открыть файл через WebHDFS
    - Пример команды для вывода списка файлов в / tmp:
    ```
    curl -ik -u sales1:BadPass#1 https://localhost:8443/gateway/default/webhdfs/v1/tmp?op=LISTSTATUS
    ```
      - Вы можете запустить команду ниже, чтобы создать тестовый файл в / tmp
    
      ```
      echo "Test file" > /tmp/testfile.txt
      sudo -u sales1 kinit
      ## enter BadPass#1
      sudo -u sales1 hdfs dfs -put /tmp/testfile.txt /tmp
      sudo -u sales1 kdestroy
      ```
    
    - Откройте этот файл через WebHDFS 
    ```
    curl -ik -u sales1:BadPass#1 -X GET https://localhost:8443/gateway/default/webhdfs/v1/tmp/testfile.txt?op=OPEN
    ```
      - Посмотрите на значение заголовка Location. Это будет содержать длинный URL 
        ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/knox-location.png)
        
    - Получите доступ к содержимому файла /tmp/testfile.txt, передав значение из указанного выше заголовка Location.
    ```
    curl -ik -u sales1:BadPass#1 -X GET '{https://localhost:8443/gateway/default/webhdfs/data/v1/webhdfs/v1/tmp/testfile.txt?_=AAAACAAAABAAAAEwvyZNDLGGNwahMYZKvaHHaxymBy1YEoe4UCQOqLC7o8fg0z6845kTvMQN_uULGUYGoINYhH5qafY_HjozUseNfkxyrEo313-Fwq8ISt6MKEvLqas1VEwC07-ihmK65Uac8wT-Cmj2BDab5b7EZx9QXv29BONUuzStCGzBYCqD_OIgesHLkhAM6VNOlkgpumr6EBTuTnPTt2mYN6YqBSTX6cc6OhX73WWE6atHy-lv7aSCJ2I98z2btp8XLWWHQDmwKWSmEvtQW6Aj-JGInJQzoDAMnU2eNosdcXaiYH856zC16IfEucdb7SA_mqAymZuhm8lUCvL25hd-bd8p6mn1AZlOn92VySGp2TaaVYGwX-6L9by73bC6sIdi9iKPl3Iv13GEQZEKsTm1a96Bh6ilScmrctk3zmY4vBYp2SjHG9JRJvQgr2XzgA}'
    ```
    
  - В. Используйте отличные скрипты для доступа к WebHDFS
    - Отредактируйте отличный скрипт, чтобы установить:
      - gateway = "https://localhost:8443/gateway/default"
    ```
    sudo vi /usr/hdp/current/knox-server/samples/ExampleWebHdfsLs.groovy
    ```
    - Запустите сценарий и введите учетные данные при появлении запроса имя пользователя: sales1 и пароль: BadPass # 1
    ```
    sudo java -jar /usr/hdp/current/knox-server/bin/shell.jar /usr/hdp/current/knox-server/samples/ExampleWebHdfsLs.groovy
    ```
    - Вывод уведомления показывает список каталогов в HDFS
    ```
    [app-logs, apps, ats, hdp, mapred, mr-history, ranger, tmp, user, zone_encr]
    ```
    
  - D. Доступ через браузер 
    - Возьмите тот же URL-адрес, который мы использовали через curl, и замените localhost на общедоступный IP-адрес узла Knox (не забудьте использовать https!), Например. **https**://PUBLIC_IP_OF_KNOX_HOST:8443/gateway/default/webhdfs/v1?op=LISTSTATUS
    - Откройте URL-адрес через браузер
    - Войдите в систему как sales1/BadPass#1
    
     ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/knox-webhdfs-browser1.png)
     ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/knox-webhdfs-browser2.png)
     ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/knox-webhdfs-browser3.png)
    

- Мы показали, как можно использовать Knox, чтобы конечный пользователь не знал о внутренних деталях кластера.
  - независимо от того, керберизован он или нет
  - какова топология кластера (например, на каком узле был запущен WebHDFS)


#### Hive в Knox 

##### Настройка Hive для Knox

- В Ambari в разделе Hive> Configs> установите указанные ниже параметры и перезапустите компонент Hive.
  - hive.server2.transport.mode = http
- Предоставьте пользователям доступ к файлу jks.
  - Это только для тестирования, поскольку мы используем самозаверяющий сертификат.
  - Это открывает только хранилище доверенных сертификатов, но не ключи.
```
sudo chmod o+x /usr/hdp/current/knox-server /usr/hdp/current/knox-server/data /usr/hdp/current/knox-server/data/security /usr/hdp/current/knox-server/data/security/keystores
sudo chmod o+r /usr/hdp/current/knox-server/data/security/keystores/gateway.jks
```

##### Конфигурация Ranger для Hive для Knox

- Настройте политику Knox для группы продаж HIVE путем:
- Зайдите Ranger > Access Manager > KNOX > щелкните ссылку имени кластера > Add new policy
  - Policy name: hive
  - Topology name: default
  - Service name: HIVE
  - Group permissions: sales 
  - Permission: check Allow
  - Нажмите Add

  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-knox-hive-policy.png)


##### Используйте Hive для Knox

- По умолчанию Knox будет использовать самозаверяющий (ненадежный) сертификат. Доверять сертификату:
  
    - Сначала на узле Knox создайте /tmp/knox.crt certificate

```
knoxserver=$(hostname -f)
openssl s_client -connect ${knoxserver}:8443 <<<'' | openssl x509 -out /tmp/knox.crt
```
  - На узле, с которого будет запускаться beeline (например, узел Hive):
      - скопируйте /tmp/knox.crt
        - Самый простой вариант - просто открыть его в `vi` и скопировать / вставить содержимое поверх:
        `vi /tmp/knox.crt`
      - доверьте сертификату, выполнив команду ниже      

```
sudo keytool -import -trustcacerts -keystore /etc/pki/java/cacerts -storepass changeit -noprompt -alias knox -file /tmp/knox.crt
```

  - Теперь подключитесь через beeline, не забудьте сначала заменить KnoxserverInternalHostName ниже:

```
beeline -u "jdbc:hive2://<KnoxserverInternalHostName>:8443/;ssl=true;transportMode=http;httpPath=gateway/default/hive" -n sales1 -p BadPass#1
```

- Обратите внимание, что в строке подключения JDBC для подключения к защищенному Hive, работающему в транспортном режиме http:
  - *порт меняется на порт 8443 Knox*
  - *трафик между клиентом и Knox превышает HTTPS*
  - *принципала Kerberos больше не нужно передавать*


- Протестируйте этих пользователей:
  - sales1/BadPass#1 should work
  - hr1/BadPass#1 should *not* work
    - Will fail with:
    ```
    Could not create http connection to jdbc:hive2://<hostname>:8443/;ssl=true;transportMode=http;httpPath=gateway/default/hive. HTTP Response code: 403 (state=08S01,code=0)
    ```

- Проверьте Ranger Audits, чтобы убедиться, что запросы были проверены:
  - Ranger > Audit > Service type: KNOX
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-audit-KNOX-hive-summary.png)


- Это показывает, как Knox помогает конечным пользователям безопасно получать доступ к Hive через HTTPS, используя Ranger для установки политик авторизации и аудита

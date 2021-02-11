# Лабораторная работа 6a

## Настройка шифрования KMS/Data в Ranger 


- Цель: в этой лабораторной работе мы установим Ranger KMS через Ambari. Затем мы создадим несколько ключей шифрования и будем использовать их для создания зон шифрования (EZ) и копирования в них файлов. Справка: [Документы](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.4/bk_Ranger_KMS_Admin_Guide/content/ch_ranger_kms_overview.html)

- В этом разделе нам нужно будет настроить прокси-пользователей. Это сделано для включения *имперсонализации* при котором суперпользователь может отправлять задания или обращаться к hdfs от имени другого пользователя (например, потому что у суперпользователя есть учетные данные Kerberos, а у пользователя joe их нет)
  - Подробнее об этом см. [документ](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/Superusers.html)

- Перед началом установки KMS найдите и запишите приведенную ниже информацию. Данные будут использоваться во время установки KMS.
  - Найдите внутреннее имя хоста, на котором запущен *Mysql* и запишите его.
    - В Ambari> Hive> Mysql> щелкните гиперссылку «Mysql Server». Внутреннее имя хоста должно появиться в верхнем левом углу страницы.

- Откройте Ambari > запустите мастера 'Add service' > выберите 'Ranger KMS'.
- Выберите удобное место для установки
- Сохраните конфигурации по умолчанию, кроме 
  - в Ambari > Ranger KMS > вкладка Settings  :
    - Ranger KMS DB host: <FQDN of Mysql>
    - Ranger KMS DB password: `BadPass#1` 
    - DBA password: `BadPass#1`
    - KMS master secret password: `BadPass#1`
     ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ambari-KMS-enhancedconfig1.png) 
     ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ambari-KMS-enhancedconfig2.png) 
    
  - - В разделе Advanced> Custom kms-site введите следующие конфигурации (Совет: чтобы не добавлять по одному, вы можете использовать режим массового добавления):
      - hadoop.kms.proxyuser.oozie.users=*
      - hadoop.kms.proxyuser.ambari.users=*
      - hadoop.kms.proxyuser.oozie.hosts=*
      - hadoop.kms.proxyuser.ambari.hosts=*
      - hadoop.kms.proxyuser.keyadmin.groups=*
      - hadoop.kms.proxyuser.keyadmin.hosts=*
      - hadoop.kms.proxyuser.keyadmin.users=*     
        ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ambari-KMS-proxy.png) 
  
- Нажмите Next > Proceed Anyway чтобы продолжить работу с мастером.

- При появлении запроса на странице «Configure Identities» вам, возможно, придется ввести учетные данные администратора AD:
  - Admin principal: `hadoopadmin@LAB.HORTONWORKS.NET`
  - Admin password: BadPass#1
  - Установите флажок "Save admin credentials" 
  
- Нажмите Next > Deploy чтобы установить RangerKMS
  
- Подтвердите, что эти свойства были заполнены kms://http@(kmshostname):9292/kms
  - HDFS > Configs > Advanced core-site:
    - hadoop.security.key.provider.path
  - HDFS > Configs > Advanced hdfs-site:
    - dfs.encryption.key.provider.uri  
  
- ерезапустите службы, которые требуют этого, например HDFS, Mapreduce, YARN через Actions > Restart All Required

- Перезапустите сервисы Ranger и RangerKMS.

- (Необязательно) Добавьте еще один KMS:
  - Ambari > Ranger KMS > Service Actions > Add Ranger KMS Server > Pick any host
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ambari-add-KMS.png) 
  - После установки вы можете запустить его:
    - Ambari > Ranger KMS > Service Actions > Start
    
  - После запуска вы увидите несколько серверов KMS, работающих в Ambari:  
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ambari-multiple-KMS.png) 

------------------

# Лабораторная работа 6b

## Упражнение по шифрованию KMS/Data в Ranger

- Прежде чем мы сможем начать использовать шифрование HDFS, нам нужно будет установить:
  - политику для доступа hadoopadmin к HDFS
  - политику для доступа hadoopadmin к Hive
  - политику доступа hadoopadmin к созданным нами KMS-ключам

  - добавить пользователя hadoopadmin в глобальные политики Ranger HDFS. 
    - Access Manager > HDFS > (имя кластера)_hdfs   
    - Откроется список политик HDFS.
     ![Image](screenshots/Ranger-KMS-HDFS-list.png) 
    - Отредактируйте глобальную политику all-path и добавьте hadoopadmin в глобальную политику HDFS и сохраните 
    ![Image](screenshots/Ranger-KMS-HDFS-add-hadoopadmin.png) 
    - Теперь ваша политика включает hadoopadmin
    ![Image](screenshots/Ranger-KMS-HDFS-list-after.png) 
  - Добавьте пользователя hadoopadmin в глобальные политики Ranger Hive. (У Hive есть две глобальные политики: одна для таблиц Hive и одна для UDF Hive)
    - Access Manager > HIVE > (имя кластера)_hive   
    - Это откроет список политик HIVE.
    [рисунок](screenshots/Ranger-KMS-HIVE-list.png) 
    - Отредактируйте глобальную политику «all - database, table, column» и добавьте hadoopadmin в глобальную политику HIVE и сохраните  
    ![Image](screenshots/Ranger-KMS-HIVE-add-hadoopadmin-table.png) 
    - Отредактируйте глобальную политику all - database, udf и добавьте hadoopadmin в глобальную политику HIVE и сохраните 
    ![Image](screenshots/Ranger-KMS-HIVE-add-hadoopadmin-udf.png) 
    - Теперь ваша политика включает hadoopadmin
     ![Image](screenshots/Ranger-KMS-HIVE-list-after.png) 
  - Предоставьте ключевому администратору разрешение на просмотр экрана аудита в Ranger:
    - Перейдите в Settings tab > Permissions
     ![Image](screenshots/Ranger-user-permissions.png)
    - Нажмите «Audit», чтобы изменить пользователей, у которых есть доступ к экрану аудита.
    - В разделе 'Select User', добавьте пользователя 'keyadmin' 
     ![Image](screenshots/Ranger-user-permissions-audits.png)
    - Сохраните
  
- Выйдите из системы Ranger
  - В правом верхнем углу admin > Logout      
- Войдите в Ranger как keyadmin/keyadmin
- Убедитесь, что репозиторий KMS настроен правильно
  - В разделе Service Manager> KMS> щелкните значок Edit (рядом со значком корзины), чтобы изменить репозиторий KMS.
  ![Image](screenshots/Ranger-KMS-edit-repo.png) 
  - Нажмите 'Test connection' и подтвердите изменения

- Создайте ключ под названием testkey - для справки: см. [документ](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.5.0/bk_security/content/use_ranger_kms.html)
  - Выберите Encryption > Key Management
  - Выберите KMS service > выберите свой kms > Add new Key
    - если возникает ошибка, вернитесь и проверьте соединение, как описано в предыдущем шаге
  - Создайте ключ под названием `testkey` > Save
  ![Image](screenshots/Ranger-KMS-createkey.png)

- Точно так же создайте еще один ключ с именем `testkey2`
  - Выберите Encryption > Key Management
  - Выберите KMS service > выберите ваш kms > Add new Key
  - Создайте ключ под названием `testkey2` > Save  

- Добавьте пользователя `hadoopadmin` в политику ключей KMS по умолчанию
  - Нажмите на вкладку Access Manager
  - Перейдите в Service Manager > KMS > (имя кластера)_kms 
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-KMS-policy.png)

  - Измените политику по умолчанию
  ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-KMS-edit-policy.png)
  
  - В разделе «Select User» добавьте пользователя «hadoopadmin» и нажмите «Save».
   ![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-KMS-policy-add-nn.png)
  - Обратите внимание, что:
      - Пользователю`hdfs` необходимы привилегии `GetMetaData` и `GenerateEEK`- HDP 2.5
      - Пользователю`nn` необходимы привилегии`GetMetaData` и `GenerateEEK`  - HDP 2.4
      - Пользователю`hive` необходимы привилегии `GetMetaData` и `DecryptEEK` 
  
- Выполните указанные ниже действия, чтобы создать зону использования ключа и выполните упражнения по базовому ключу и зоне шифрования (EZ). 
  - Создайте EZ с помощью ключей
  - Скопируйте файл в EZs
  - Удалите файл из EZ
  - Просмотрите содержимое необработанного файла
  - Запретите доступ к необработанному файлу
  - Скопируйте файл через EZs
  - Переместите каталог hive-хранилища в EZ
  
```
#запуск на узле Ambari

export PASSWORD=BadPass#1

#определение имени кластера
output=`curl -u hadoopadmin:$PASSWORD -k -i -H 'X-Requested-By: ambari'  https://localhost:8443/api/v1/clusters`
cluster=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

echo $cluster
## это должно показать имя вашего кластера

## если нет, вы можете вручную установить это, как показано ниже
## cluster=Security-HWX-LabTesting-XXXX

#сначала мы запустим логин 3 разных пользователей: hdfs, hadoopadmin, sales1

#kinit,  hadoopadmin и sales используют BadPass#1 
sudo -u hadoopadmin kinit
## введите BadPass#1
sudo -u sales1 kinit
## войдите BadPass#1

#после этого kinit как hdfs использует headless keytab и the principal name
sudo -u hdfs kinit -kt /etc/security/keytabs/hdfs.headless.keytab "hdfs-${cluster,,}"

#hadoopadmin перечисляет ключи и их метаданные
sudo -u hadoopadmin hadoop key list -metadata

#hadoopadmin создает каталоги для EZ
sudo -u hadoopadmin hdfs dfs -mkdir /zone_encr
sudo -u hadoopadmin hdfs dfs -mkdir /zone_encr2

#hdfs создает 2 EZ с помощью 2 ключей
sudo -u hdfs hdfs crypto -createZone -keyName testkey -path /zone_encr
sudo -u hdfs hdfs crypto -createZone -keyName testkey2 -path /zone_encr2
# если вы получаете ошибку RemoteException, это означает, что вы не предоставили права пользователя namenode для testkey, создав политику для KMS в Ranger

#проверьте, созданы ли EZ
sudo -u hdfs hdfs crypto -listZones  

#создайте тестовые файлы
sudo -u hadoopadmin echo "My test file1" > /tmp/test1.log
sudo -u hadoopadmin echo "My test file2" > /tmp/test2.log

#скопируйте файлы в EZs
sudo -u hadoopadmin hdfs dfs -copyFromLocal /tmp/test1.log /zone_encr
sudo -u hadoopadmin hdfs dfs -copyFromLocal /tmp/test2.log /zone_encr

sudo -u hadoopadmin hdfs dfs -copyFromLocal /tmp/test2.log /zone_encr2

#Обратите внимание, что hadoopadmin разрешил расшифровывать EEK, но не пользователь sales (поскольку это не разрешено политикой Ranger)
sudo -u hadoopadmin hdfs dfs -cat /zone_encr/test1.log
sudo -u hadoopadmin hdfs dfs -cat /zone_encr2/test2.log
#это должно сработать

sudo -u sales1      hdfs dfs -cat /zone_encr/test1.log
## это должно выдать вам ошибку ниже
## cat: User:sales1 not allowed to do 'DECRYPT_EEK' on 'testkey'
```

- Проверьте страницу Ranger> Audit и обратите внимание, что запрос от hadoopadmin был разрешен, но запрос от sales1 был отклонен.
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/Ranger-KMS-audit.png)

- Теперь давайте протестируем удаление и копирование файлов между EZs- ([справочный документ](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.4/bk_hdfs_admin_tools/content/copy-to-from-encr-zone.html))
```
#попробуйте удалить файл из EZ обычным способом -rm command 
sudo -u hadoopadmin hdfs dfs -rm /zone_encr/test2.log
## Это работает, потому что с версии HDP2.4.3 параметр -skipTrash больше не нужно указывать

#подтвердите, что test2.log был удален и что zone_encr содержит только test1.log
sudo -u hadoopadmin hdfs dfs -ls  /zone_encr/
 
#скопируйте файл между EZ, используя distcp с параметром -skipcrccheck
sudo -u hadoopadmin hadoop distcp -skipcrccheck -update /zone_encr2/test2.log /zone_encr/
```
- Теперь посмотрим на содержимое необработанного файла.
```
#Просмотр содержимого необработанного файла в зашифрованной зоне от имени суперпользователя hdfs. 
Это должно показать некоторые зашифрованные символы
sudo -u hdfs hdfs dfs -cat /.reserved/raw/zone_encr/test1.log

#Запретите пользователю hdfs читать файл, установив security.hdfs.unreadable.by.superuser attribute. Обратите внимание, что этот атрибут может быть установлен только для файлов и никогда не может быть удален.
sudo -u hdfs hdfs dfs -setfattr -n security.hdfs.unreadable.by.superuser  /.reserved/raw/zone_encr/test1.log

# Теперь, будучи суперпользователем hdfs, попробуйте прочитать файлы или содержимое необработанного файла.
sudo -u hdfs hdfs dfs -cat /.reserved/raw/zone_encr/test1.log

## Вы должны получить ошибку ниже
##cat: Access is denied for hdfs since the superuser is not allowed to perform this operation.

```

- Настройте Hive для шифрования HDFS с помощью testkey. [Справка](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.4/bk_hdfs_admin_tools/content/hive-access-encr.html)
```
sudo -u hadoopadmin hdfs dfs -mv /apps/hive /apps/hive-old
sudo -u hadoopadmin hdfs dfs -mkdir /apps/hive
sudo -u hdfs hdfs crypto -createZone -keyName testkey -path /apps/hive
sudo -u hadoopadmin hadoop distcp -skipcrccheck -update /apps/hive-old/warehouse /apps/hive/warehouse
```

- Чтобы настроить рабочий каталог Hive (hive.exec.scratchdir) так, чтобы он располагался внутри зоны шифрования, выполните следующее:
  - Ambari > Hive > Configs > Advanced 
    - hive.exec.scratchdir = /apps/hive/tmp
  - Перезапустите Hive
  
- Убедитесь, что разрешения для / apps / hive / tmp установлены на 1777
```
sudo -u hdfs hdfs dfs -chmod -R 1777 /apps/hive/tmp
```

- Подтвердите разрешения, войдя в каталог с именем sales1.
```
sudo -u sales1 hdfs dfs -ls /apps/hive/tmp
## это должно предоставить список
```

- Удалите запись  sales1
```
sudo -u sales1 kdestroy
```

- Выйдите из системы Ranger как пользователь keyadmin

------------------


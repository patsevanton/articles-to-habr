https://habr.com/ru/topic/edit/539620/

Apache HDFS (Hadoop Distributed File System) — файловая система, предназначенная для хранения файлов больших размеров, поблочно распределённых между узлами вычислительного кластера.
Apache Hive — система управления базами данных на основе платформы Hadoop.
Apache HBase — СУБД класса NoSQL с открытым исходным кодом, проект экосистемы Hadoop.
Apache KNOX — REST API и шлюз приложений для компонентов экосистемы Apache Hadoop, обеспечивает единую точку доступа для всех HTTP соединений с кластерами Apache Hadoop и систему единой аутентификации Single Sign On (SSO) для сервисов и пользовательского интерфейса компонент Apache Hadoop.
Apache Ranger – это инфраструктура для обеспечения, мониторинга и управления комплексной безопасностью данных на платформе Hadoop

Перевод поста [Securing hdfs hive hbase knox ranger](http://saptak.in/writing/2015/09/08/securing-hdfs-hive-hbase-knox-ranger) 2015 года. Получше и поновее поста не нашел. 

<cut />

Введение

Apache Ranger обеспечивает комплексный подход к безопасности кластера Hadoop. Он обеспечивает централизованное администрирование политик безопасности по основным требованиям безопасности предприятия, включая авторизацию, учет и защиту данных.

Apache Ranger расширяет базовые функции для скоординированного применения в рамках рабочих нагрузок Hadoop, включая пакетный, интерактивный SQL и Hadoop в реальном времени.

В этом руководстве мы рассмотрим использование Apache Ranger для HDP 2.3 для защиты вашей среды Hadoop. Мы рассмотрим следующие темы:

1. Поддержка авторизации Knox и аудита

2. Политики командной строки в Hive

3. Политики командной строки в HBase

4. API REST для диспетчера политик

**Предпосылки**

Единственное предварительное условие для этого урока - наличие [Hortonworks Sandbox](http://hortonworks.com/sandbox).

Как только у вас появится Hortonworks Sandbox, войдите через SSH:

![](https://habrastorage.org/webt/x_/9u/xz/x_9uxzjjm-imo92qojs3bk4cuio.png)

**Запуск службы Knox и демонстрационной службы LDAP**

В консоли Ambari по адресу http://localhost:8080/ (имя пользователя и пароль - admin и admin соответственно) выберите Knox из списка служб в левой части страницы.

![](https://habrastorage.org/webt/u4/do/gw/u4dogw6cf9dyx-2iswwzjjjbdqs.png)

Затем нажмите `Service Actions` в верхнем правом углу страницы и нажмите `Start`.

![](https://habrastorage.org/webt/mk/9p/si/mk9psiz6ofyuq6svydltc86gwqy.png)

Далее вы можете отследить запуск службы Knox до его завершения:

![](https://habrastorage.org/webt/kp/br/jn/kpbrjnsbe22yyznniok-pvl6mme.png)

Затем вернитесь к кнопке `Service Actions` в службе `Knox` и нажмите `Start Demo LDAP`.

![](https://habrastorage.org/webt/hl/qf/pf/hlqfpfhmsdf8ohwhdkwk2azglp8.png)

Вы можете отследить запуск демо-службы LDAP на следующем экране:

![](https://habrastorage.org/webt/hc/kl/wa/hcklwabfxohgowiibjsqk_ogfre.png)

**Сценарии доступа Knox**

Проверьте, запущена ли консоль администратора Ranger по адресу http://localhost:6080/ с вашего хост-компьютера. Имя пользователя - `admin`, пароль - `admin`.

![](https://habrastorage.org/webt/co/vm/5e/covm5emwmmgbzlvrhpw7htdbzys.png)

Если он не запущен, вы можете запустить его из командной строки с помощью команды

```
sudo service ranger-admin start
```

![](https://habrastorage.org/webt/yp/fz/9k/ypfz9kn0qtl7fvgctjyazey2e-i.png)

Щелкните ссылку sandbox_knox в разделе Knox на главном экране Ranger Administration Portal

![](https://habrastorage.org/webt/fj/vu/pa/fjvupamf-kkyh9yhwr0bxy9x0ni.png)

Вы можете просмотреть подробные сведения о политике, щелкнув название политики.

![](https://habrastorage.org/webt/hi/iq/2c/hiiq2csabtxms294cdf5emqo88u.png)

Чтобы начать тестирование политик Knox, нам нужно отключить политику `global knox allow`.

![](https://habrastorage.org/webt/y8/qq/ak/y8qqak31og2e6h-9hp40hj57u-s.png)

Найдите политику `Sandbox for Guest` в консоли Ranger Admin и отредактируйте политику.

![](https://habrastorage.org/webt/7n/ag/w9/7nagw9pdafyrerb0-0krvci8eqk.png)

и включите политику `Sandbox for Guest`

![](https://habrastorage.org/webt/m9/uq/4b/m9uq4bwhm-yw83enxkdpd7oyjpw.png)

Из локального терминала SSHd (непосредственно в Sandbox) запустите эту команду CURL для доступа к WebHDFS.

```
curl -k -u admin:admin-password 'https://127.0.0.1:8443/gateway/knox_sample/webhdfs/v1?op=LISTSTATUS'
```

![](https://habrastorage.org/webt/xm/vv/ew/xmvvewxnkbmmwxtu0snx1qbkuoq.png)

Перейдите в инструмент Ranger Policy Manager → Audit screen и проверьте, проверяется ли доступ (отказ) к Knox.

![](https://habrastorage.org/webt/q1/2b/po/q12bpov9kpvgy9vaeib4afcxz68.png)

Теперь давайте попробуем ту же команду CURL, используя учетные данные `guest` из терминала.

```
curl -k -u guest:guest-password 'https://127.0.0.1:8443/gateway/knox_sample/webhdfs/v1?op=LISTSTATUS'
```

![](https://habrastorage.org/webt/jd/es/ed/jdesed3qhwjnyc0wkmyzmskdojm.png)

```
{"FileStatuses":{"FileStatus":[{"accessTime":0,"blockSize":0,"childrenNum":0,"fileId":16393,"group":"hadoop","length":0,"modificationTime":1439987528048,"owner":"yarn","pathSuffix":"app-logs","permission":"777","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":4,"fileId":16389,"group":"hdfs","length":0,"modificationTime":1439987809562,"owner":"hdfs","pathSuffix":"apps","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":17000,"group":"hdfs","length":0,"modificationTime":1439989173392,"owner":"hdfs","pathSuffix":"demo","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16398,"group":"hdfs","length":0,"modificationTime":1439987529660,"owner":"hdfs","pathSuffix":"hdp","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16394,"group":"hdfs","length":0,"modificationTime":1439987528532,"owner":"mapred","pathSuffix":"mapred","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":2,"fileId":16396,"group":"hadoop","length":0,"modificationTime":1439987538099,"owner":"mapred","pathSuffix":"mr-history","permission":"777","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16954,"group":"hdfs","length":0,"modificationTime":1439988741413,"owner":"hdfs","pathSuffix":"ranger","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":3,"fileId":16386,"group":"hdfs","length":0,"modificationTime":1440165443820,"owner":"hdfs","pathSuffix":"tmp","permission":"777","replication":0,"storagePolicy":0,"type":"DIRECTORY"},{"accessTime":0,"blockSize":0,"childrenNum":8,"fileId":16387,"group":"hdfs","length":0,"modificationTime":1439988397561,"owner":"hdfs","pathSuffix":"user","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"}]}}
```

Мы можем проверить аудит в Ranger Policy Manager → Audit screen.

![](https://habrastorage.org/webt/q9/hs/tv/q9hstvxvb5iq_wd5nuqpe8qum4m.png)

Плагин Ranger для Knox перехватывает любые запросы, отправленные к Knox, и применяет политики, полученные с портала администрирования Ranger.

Вы можете настроить политики Knox в Ranger, чтобы ограничить их определенной службой (WebHDFS, WebHCAT и т. д.) и конкретным пользователем или группой, и вы даже можете привязать пользователя / группу к IP-адресу

![](https://habrastorage.org/webt/94/3f/at/943fatz35xaxgapfenqfyy6bsjg.png)

**Разрешения grant/revoke в Hive**

Ranger может поддерживать импорт политик `grant`/`revoke`, установленных через командную строку или `Hue for Hive`. Ranger может централизованно хранить эти политики вместе с политиками, созданными на портале администрирования, и применять их в Hive с помощью своего плагина.

![](https://habrastorage.org/webt/xh/nh/c3/xhnhc3i0-36tib6oot9_lwktwks.png)

Первым делом отключите глобальную политику доступа для Hive на портале администрирования Ranger.

![](https://habrastorage.org/webt/_b/lh/p-/_blhp-7826ifhrahzl4rmsyaslu.png)

Давайте попробуем запустить операцию `Grant`, используя пользовательский `Hive` из командной строки. Войдите в инструмент beeline, используя следующую команду

```
beeline -u "jdbc:hive2://sandbox.hortonworks.com:10000/default" -n it1 -p it1-d org.apache.hive.jdbc.HiveDriver
```

![](https://habrastorage.org/webt/vt/sg/z-/vtsgz-fpaapkgctkwqzm2ydj9dq.png)

Затем введите команду `GRANT`

```
grant select, update on table xademo.customer_details to user network1;
```

Вы должны увидеть следующую ошибку:

![](https://habrastorage.org/webt/ld/w9/7s/ldw97stv8ndpjitozm08x4fuwru.png)

Давайте проверим журнал аудита в Ranger Administration Portal → Audit

![](https://habrastorage.org/webt/dq/sn/mj/dqsnmjyva_hcbqqvgxpkrdjouxo.png)

Вы можете видеть, что доступ был запрещен для операции администратора для пользователя `it1`.

Мы можем создать политику в Ranger для пользователя `it1` в качестве администратора. Создайте новую политику в консоли администратора Ranger и убедитесь, что конфигурация соответствует приведенной ниже иллюстрации.

![](https://habrastorage.org/webt/hz/ew/k0/hzewk0m3izxa79_cto3-e1kzmou.png)

Мы можем попробовать команду beeline еще раз, как только политика будет сохранена.

```
GRANT select, update on table xademo.customer_details to user network1;
```

![](https://habrastorage.org/webt/5h/ne/--/5hne--3cz4eu5e92_zd_jzcaxb8.png)

Если команда прошла успешно, вы увидите, что политика создана / обновлена в Ranger Admin Portal → Policy Manager. Он проверяет, существует ли существующая соответствующая политика для обновления, в противном случае он создает новую.

![](https://habrastorage.org/webt/ib/xm/ii/ibxmiiwqz4uwsrxmddmfk0h27fg.png)

Что здесь происходит?

Плагин Ranger перехватывает команды `GRANT/REVOKE` в `Hive` и создает соответствующие политики на портале администратора. Затем плагин использует эти политики для принудительной авторизации `Hive` (`Hiveserver2`).

Пользователи могут запускать дополнительные команды `GRANT` для обновления разрешений и команды `REVOKE` для отмены разрешений.

**Разрешения grant/revoke в HBase**

Ranger может поддерживать импорт политик `grant/revoke`, установленных через командную строку в `Hbase`. Подобно Hive, Ranger может хранить эти политики как часть диспетчера политик и применять их в Hbase с помощью своего плагина.

Прежде чем продолжить, убедитесь, что HBase запущен из Ambari - http://127.0.0.1:8080 (имя пользователя и пароль `admin`).

Если это не так, нажмите кнопку `Service Actions` в правом верхнем углу и запустите службу.

![](https://habrastorage.org/webt/up/_n/03/up_n03goncsrcgbkwqf8jqaa0vq.png)

Сначала давайте попробуем запустить операцию Grant с помощью пользователя Hbase.

Отключите политику общего доступа `HBase Global Allow` на Ranger Administration Portal - диспетчер политик.

![](https://habrastorage.org/webt/bj/kq/0q/bjkq0qi6ejh4bu5baj9fwkn0xrk.png)

Войдите в оболочку HBase как пользователь it1

```
su - it1
[it1@sandbox ~]$ hbase shell
```

![](https://habrastorage.org/webt/ts/3q/7t/ts3q7tv3qgpbj7aqrxam-dtzyom.png)

Запустите команду grant, чтобы предоставить пользователю `mktg1` доступ на чтение, запись и создание в таблице `iemployee`.

```
hbase(main):001:0> grant 'mktg1', 'RWC', 'iemployee'
```

вы должны получить отказ в доступе, как показано ниже:

![](https://habrastorage.org/webt/qk/ji/xt/qkjixtrzfstw8pzdb-9ictgl92m.png)

Перейдите в Ranger Administration Portal→ Policy Manager и создайте новую политику, чтобы назначить права `admin` пользователю `it1`.

![](https://habrastorage.org/webt/q_/ou/6-/q_ou6-tfbb7qgvfmdd8m-erhhye.png)

Сохраните политику и снова запустите команду HBase.

```
hbase(main):006:0> grant 'mktg1', 'RWC', 'iemployee'

0 row(s) in 0.8670 seconds
```

![](https://habrastorage.org/webt/mc/0g/yr/mc0gyrnq9vtr1dy8vrsvkwldyaq.png)

Проверьте политики HBase на портале Ranger Policy Administration. Разрешения на предоставление были добавлены в существующую политику для таблицы `iemployee`, которую мы создали на предыдущем шаге.

![](https://habrastorage.org/webt/2z/rr/i1/2zrri1auat8aoye4vzlyojxsby4.png)

Вы можете отозвать те же разрешения, и они будут удалены из администратора Ranger. Попробуйте это в той же оболочке HBase

```
hbase(main):007:0> revoke 'mktg1', 'iemployee'

0 row(s) in 0.4330 seconds
```

Вы можете проверить существующую политику и посмотреть, не были ли она изменена

![](https://habrastorage.org/webt/kf/ey/3j/kfey3j4wqnkel9oguja3twxlyfc.png)

Что здесь происходит?

Плагин Ranger перехватывает команды `GRANT/REVOKE` в Hbase и создает соответствующие политики на портале администратора. Затем плагин использует эти политики для принудительной авторизации.

Пользователи могут запускать дополнительные команды `GRANT` для обновления разрешений и команды REVOKE для отмены разрешений.

**REST APIs для администрирования политик**

Администрированием политик Ranger можно управлять через REST API. Пользователи могут использовать API для создания или обновления политик, вместо того, чтобы заходить на портал администрирования.

**Запуск REST API из командной строки**

В локальной оболочке командной строки запустите эту команду CURL. Этот API создаст политику с именем `hadoopdev-testing-policy2` в репозитории HDFS `sandbox_hdfs`

```
curl -i --header "Accept:application/json" -H "Content-Type: application/json" --user admin:admin -X POST http://127.0.0.1:6080/service/public/api/policy -d '{ "policyName":"hadoopdev-testing-policy2","resourceName":"/demo/data/test","description":"Testing policy for /demo/data/test","repositoryName":"sandbox_hdfs","repositoryType":"HDFS","permMapList":[{"userList":["mktg1"],"permList":["Read"]},{"groupList":["IT"],"permList":["Read"]}],"isEnabled":true,"isRecursive":true,"isAuditEnabled":true,"version":"0.1.0","replacePerm":false}'
```

![](https://habrastorage.org/webt/sb/8i/yy/sb8iyybegmp8juwrzji-gv7rkxo.png)

И диспетчер политик и просмотрите новую политику с именем `hadoopdev-testing-policy2`

![](https://habrastorage.org/webt/r0/ae/hm/r0aehmkmksmufx2y7etq3g9rzgc.png)

Щелкните политику и проверьте созданные разрешения.

![](https://habrastorage.org/webt/ja/u8/78/jau878k6obge_uanolwox1b_mzo.png)

Идентификатор политики является частью URL-адреса этой страницы сведений о политике http://127.0.0.1:6080/index.html#!/hdfs/1/policy/26

Мы можем использовать идентификатор политики для получения или изменения политики.

Выполните приведенную ниже команду CURL, чтобы получить сведения о политике с помощью API.

```
curl -i --user admin:admin -X GET http://127.0.0.1:6080/service/public/api/policy/26
```

Что здесь происходит?

Мы создали политику и получили ее детали с помощью REST API. Теперь пользователи могут управлять своими политиками с помощью инструментов API или приложений, интегрированных с REST API Ranger.

Надеюсь, во время этого головокружительного тура по Ranger вы познакомились с простотой и мощью Ranger для управления безопасностью.

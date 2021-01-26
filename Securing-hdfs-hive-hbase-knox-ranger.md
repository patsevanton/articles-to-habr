Введение

Apache Ranger обеспечивает комплексный подход к безопасности кластера Hadoop. Он обеспечивает централизованное администрирование политик безопасности по основным требованиям безопасности предприятия, включая авторизацию, учет и защиту данных.

Apache Ranger расширяет базовые функции для скоординированного применения в рамках рабочих нагрузок Hadoop, включая пакетный, интерактивный SQL и Hadoop в реальном времени.

В этом руководстве мы рассмотрим использование Apache Ranger для HDP 2.3 для защиты вашей среды Hadoop. Мы рассмотрим следующие темы:

1. Поддержка авторизации Knox и аудита

2. Политики командной строки в Hive

3. Политики командной строки в HBase

4. API REST для диспетчера политик

 

Предпосылки

Единственное предварительное условие для этого урока - наличие [Hortonworks Sandbox](http://hortonworks.com/sandbox).

Как только у вас появится Hortonworks Sandbox, войдите через SSH:

 

Запуск службы Knox и демонстрационной службы LDAP

В консоли Ambari по адресу http: // localhost: 8080 / (имя пользователя и пароль - admin и admin соответственно) выберите Knox из списка служб в левой части страницы.

 

Затем нажмите «Service Actions» в верхнем правом углу страницы и нажмите «Start».

! [] http://www.dropbox.com/s/jhb30dgey8m30n6/Screenshot%202015-09-08%2010.27.06.png?dl=1)

Далее вы можете отследить запуск службы Knox до его завершения:

 

Затем вернитесь к кнопке «Service Actions» в службе Knox и нажмите «Start Demo LDAP».

 

Вы можете отследить запуск демо-службы LDAP на следующем экране:

 

Сценарии доступа Knox

Проверьте, запущена ли консоль администратора Ranger по адресу http: // localhost: 6080 / с вашего хост-компьютера. Имя пользователя - admin, пароль - admin.

 

Если он не запущен, вы можете запустить его из командной строки с помощью команды

sudo service ranger-admin start

 

Щелкните ссылку sandbox_knox в разделе Knox на главном экране Ranger Administration Portal

 

Вы можете просмотреть подробные сведения о политике, щелкнув название политики.

 

Чтобы начать тестирование политик Knox, нам нужно отключить политику «global knox allow».

 

Найдите политику Sandbox for Guest в консоли Ranger Admin и отредактируйте политику.

 

и включите политику " Sandbox for Guest "

 

Из локального терминала SSHd (непосредственно в Sandbox) запустите эту команду CURL для доступа к WebHDFS.

curl -k -u admin:admin-password 'https://127.0.0.1:8443/gateway/knox_sample/webhdfs/v1?op=LISTSTATUS'

 

Перейдите в инструмент Ranger Policy Manager → Audit screen и проверьте, проверяется ли доступ (отказ) к Knox.

 

Теперь давайте попробуем ту же команду CURL, используя учетные данные «гостя» из терминала.

curl -k -u guest:guest-password 'https://127.0.0.1:8443/gateway/knox_sample/webhdfs/v1?op=LISTSTATUS'

 

Мы можем проверить аудит в Ranger Policy Manager → Audit screen.

 

Плагин Ranger для Knox перехватывает любые запросы, отправленные к Knox, и применяет политики, полученные с портала администрирования Ranger.

 

Вы можете настроить политики Knox в Ranger, чтобы ограничить их определенной службой (WebHDFS, WebHCAT и т. д.) и конкретным пользователем или группой, и вы даже можете привязать пользователя / группу к IP-адресу

 

Разрешения **grant/revoke в** Hive

Ranger может поддерживать импорт политик grant/revoke, установленных через командную строку или Hue for Hive. Ranger может централизованно хранить эти политики вместе с политиками, созданными на портале администрирования, и применять их в Hive с помощью своего плагина.

 

Первым делом отключите глобальную политику доступа для Hive на портале администрирования Ranger.

 

Давайте попробуем запустить операцию Grant, используя пользовательский Hive из командной строки. Войдите в инструмент beeline, используя следующую команду

beeline -u "jdbc:hive2://sandbox.hortonworks.com:10000/default" -n it1 -p it1-d org.apache.hive.jdbc.HiveDriver

 

Затем введите команду GRANT

grant select, update on table xademo.customer_details to user network1;

Вы должны увидеть следующую ошибку:

 

Давайте проверим журнал аудита в Ranger Administration Portal → Audit

 

Вы можете видеть, что доступ был запрещен для операции администратора для пользователя it1.

Мы можем создать политику в Ranger для пользователя it1 в качестве администратора. Создайте новую политику в консоли администратора Ranger и убедитесь, что конфигурация соответствует приведенной ниже иллюстрации.

 

Мы можем попробовать команду beeline еще раз, как только политика будет сохранена.

GRANT select, update on table xademo.customer_details to user network1;

 

Если команда прошла успешно, вы увидите, что политика создана / обновлена в Ranger Admin Portal → Policy Manager. Он проверяет, существует ли существующая соответствующая политика для обновления, в противном случае он создает новую.

 

Что здесь происходит?

Плагин Ranger перехватывает команды GRANT / REVOKE в Hive и создает соответствующие политики на портале администратора. Затем плагин использует эти политики для принудительной авторизации Hive (Hiveserver2).

Пользователи могут запускать дополнительные команды GRANT для обновления разрешений и команды REVOKE для отмены разрешений.

 

Сценарии **grant/revoke** в HBase

Ranger может поддерживать импорт политик grant/revoke, установленных через командную строку в Hbase. Подобно Hive, Ranger может хранить эти политики как часть диспетчера политик и применять их в Hbase с помощью своего плагина.

Прежде чем продолжить, убедитесь, что HBase запущен из Ambari - http://127.0.0.1:8080 (имя пользователя и пароль admin).

Если это не так, нажмите кнопку «Service Actions» в правом верхнем углу и запустите службу.

 

Сначала давайте попробуем запустить операцию Grant с помощью пользователя Hbase.

Отключите политику общего доступа «HBase Global Allow» на Ranger Administration Portal - диспетчер политик.

 

Войдите в оболочку HBase как пользователь it1

 

Запустите команду grant, чтобы предоставить пользователю mktg1 доступ на чтение, запись и создание в таблице iemployee.

hbase (main): 001: 0> grant 'mktg1', 'RWC', 'iemployee'

вы должны получить отказ в доступе, как показано ниже:

 

Перейдите в Ranger Administration Portal→ Policy Manager и создайте новую политику, чтобы назначить пользователю права «администратора» it1.

 

Сохраните политику и снова запустите команду HBase.

 

Проверьте политики HBase на портале Ranger Policy Administration. Разрешения на предоставление были добавлены в существующую политику для таблицы iemployee, которую мы создали на предыдущем шаге.

 

Вы можете отозвать те же разрешения, и они будут удалены из администратора Ranger. Попробуйте это в той же оболочке HBase

 

Вы можете проверить существующую политику и посмотреть, не были ли она изменена

 

Что здесь происходит?

Плагин Ranger перехватывает команды GRANT / REVOKE в Hbase и создает соответствующие политики на портале администратора. Затем плагин использует эти политики для принудительной авторизации.

Пользователи могут запускать дополнительные команды GRANT для обновления разрешений и команды REVOKE для отмены разрешений.

 

**REST APIs** для администрирования политик

Администрированием политик Ranger можно управлять через REST API. Пользователи могут использовать API для создания или обновления политик, вместо того, чтобы заходить на портал администрирования.

\#### Running REST APIs from command line (Запуск REST API из командной строки)

В локальной оболочке командной строки запустите эту команду CURL. Этот API создаст политику с именем «hadoopdev-testing-policy2» в репозитории HDFS «sandbox_hdfs»

 

И диспетчер политик и просмотрите новую политику с именем «hadoopdev-testing-policy2»

 

Щелкните политику и проверьте созданные разрешения.

 

Идентификатор политики является частью URL-адреса этой страницы сведений о политике http://127.0.0.1: 6080 / index.html #! / Hdfs / 1 / policy / 26

Мы можем использовать идентификатор политики для получения или изменения политики.

Выполните приведенную ниже команду CURL, чтобы получить сведения о политике с помощью API.

curl -i --user admin:admin -X GET http://127.0.0.1:6080/service/public/api/policy/26

 

Что здесь происходит?

Мы создали политику и получили ее детали с помощью REST API. Теперь пользователи могут управлять своими политиками с помощью инструментов API или приложений, интегрированных с REST API Ranger.

Надеюсь, во время этого головокружительного тура по Ranger вы познакомились с простотой и мощью Ranger для управления безопасностью.





![](https://habrastorage.org/webt/ja/u8/78/jau878k6obge_uanolwox1b_mzo.png)

![](https://habrastorage.org/webt/r0/ae/hm/r0aehmkmksmufx2y7etq3g9rzgc.png)

![](https://habrastorage.org/webt/sb/8i/yy/sb8iyybegmp8juwrzji-gv7rkxo.png)

![](https://habrastorage.org/webt/kf/ey/3j/kfey3j4wqnkel9oguja3twxlyfc.png)

![](https://habrastorage.org/webt/2z/rr/i1/2zrri1auat8aoye4vzlyojxsby4.png)

![](https://habrastorage.org/webt/mc/0g/yr/mc0gyrnq9vtr1dy8vrsvkwldyaq.png)

![](https://habrastorage.org/webt/q_/ou/6-/q_ou6-tfbb7qgvfmdd8m-erhhye.png)

![](https://habrastorage.org/webt/qk/ji/xt/qkjixtrzfstw8pzdb-9ictgl92m.png)

![](https://habrastorage.org/webt/ts/3q/7t/ts3q7tv3qgpbj7aqrxam-dtzyom.png)

![](https://habrastorage.org/webt/bj/kq/0q/bjkq0qi6ejh4bu5baj9fwkn0xrk.png)

![](https://habrastorage.org/webt/up/_n/03/up_n03goncsrcgbkwqf8jqaa0vq.png)

![](https://habrastorage.org/webt/ib/xm/ii/ibxmiiwqz4uwsrxmddmfk0h27fg.png)

![](https://habrastorage.org/webt/5h/ne/--/5hne--3cz4eu5e92_zd_jzcaxb8.png)

![](https://habrastorage.org/webt/hz/ew/k0/hzewk0m3izxa79_cto3-e1kzmou.png)

![](https://habrastorage.org/webt/dq/sn/mj/dqsnmjyva_hcbqqvgxpkrdjouxo.png)

![](https://habrastorage.org/webt/ld/w9/7s/ldw97stv8ndpjitozm08x4fuwru.png)

![](https://habrastorage.org/webt/vt/sg/z-/vtsgz-fpaapkgctkwqzm2ydj9dq.png)

![](https://habrastorage.org/webt/_b/lh/p-/_blhp-7826ifhrahzl4rmsyaslu.png)

![](https://habrastorage.org/webt/xh/nh/c3/xhnhc3i0-36tib6oot9_lwktwks.png)

![](https://habrastorage.org/webt/94/3f/at/943fatz35xaxgapfenqfyy6bsjg.png)

![](https://habrastorage.org/webt/q9/hs/tv/q9hstvxvb5iq_wd5nuqpe8qum4m.png)

![](https://habrastorage.org/webt/jd/es/ed/jdesed3qhwjnyc0wkmyzmskdojm.png)

![](https://habrastorage.org/webt/q1/2b/po/q12bpov9kpvgy9vaeib4afcxz68.png)

![](https://habrastorage.org/webt/xm/vv/ew/xmvvewxnkbmmwxtu0snx1qbkuoq.png)

![](https://habrastorage.org/webt/m9/uq/4b/m9uq4bwhm-yw83enxkdpd7oyjpw.png)

![](https://habrastorage.org/webt/7n/ag/w9/7nagw9pdafyrerb0-0krvci8eqk.png)

![](https://habrastorage.org/webt/hc/kl/wa/hcklwabfxohgowiibjsqk_ogfre.png)

![](https://habrastorage.org/webt/hl/qf/pf/hlqfpfhmsdf8ohwhdkwk2azglp8.png)

![](https://habrastorage.org/webt/y8/qq/ak/y8qqak31og2e6h-9hp40hj57u-s.png)

![](https://habrastorage.org/webt/hi/iq/2c/hiiq2csabtxms294cdf5emqo88u.png)

![](https://habrastorage.org/webt/fj/vu/pa/fjvupamf-kkyh9yhwr0bxy9x0ni.png)

![](https://habrastorage.org/webt/mk/9p/si/mk9psiz6ofyuq6svydltc86gwqy.png)

![](https://habrastorage.org/webt/kp/br/jn/kpbrjnsbe22yyznniok-pvl6mme.png)

![](https://habrastorage.org/webt/u4/do/gw/u4dogw6cf9dyx-2iswwzjjjbdqs.png)

![](https://habrastorage.org/webt/co/vm/5e/covm5emwmmgbzlvrhpw7htdbzys.png)

![](https://habrastorage.org/webt/yp/fz/9k/ypfz9kn0qtl7fvgctjyazey2e-i.png)

![](https://habrastorage.org/webt/x_/9u/xz/x_9uxzjjm-imo92qojs3bk4cuio.png)

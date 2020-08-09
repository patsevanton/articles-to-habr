Интеграция Youtrack со встроенным (embedded) Hub с Teamcity, Gitlab.

В этом посте будет рассказано о том как сделать интеграцию Youtrack со встроенным (embedded) Hub с Teamcity, Gitlab.

**[YouTrack](https://www.jetbrains.com/ru-ru/youtrack/)** — коммерческая система отслеживания ошибок, программное обеспечение для управления проектами. YouTrack поддерживает поисковые запросы, автодополнение, манипуляцию с наборами задач, настройку набора атрибутов задачи, создание пользовательских рабочих процессов и реализует подход, основанный на преимущественном использовании клавиатуры.

**[TeamCity](https://www.jetbrains.com/ru-ru/teamcity/)** — интеллектуальный сервер непрерывной интеграции.

[**GitLab**](https://about.gitlab.com/) — веб-инструмент жизненного цикла DevOps с открытым исходным кодом, представляющий систему управления репозиториями кода для Git с собственной вики, системой отслеживания ошибок, CI/CD пайплайном и другими функциями.

**[Upsource](https://www.jetbrains.com/ru-ru/upsource/)** – это решение для обзора исходного кода и просмотра его репозиториев, позволяющее анализировать код, делиться им с коллегами и обсуждать результаты совместной работы. 

![](https://habrastorage.org/webt/yh/w0/yx/yhw0yxkaa1mbl77wzi3t9eirjfy.png)

#### Требования к интеграции.

**Необходимо чтобы все инструменты резолвились по DNS. Если инструменты не будут резолвиться по DNS, то у вас часть функционала не будет работать.**

#### Установка, запуск.

**Youtrack**

```
yum install -y mc unzip
wget https://download-cf.jetbrains.com/charisma/youtrack-2020.3.1402.zip
unzip youtrack-2020.3.1402.zip 
cd youtrack-2020.3.1402/bin/
./youtrack.sh start
```

**Teamcity**

```
yum install -y unzip mc java-1.8.0-openjdk-devel
wget https://download-cf.jetbrains.com/teamcity/TeamCity-2020.1.2.tar.gz
tar zxvf TeamCity-2020.1.2.tar.gz 
cd TeamCity/bin
./runAll.sh start
```

После установки YouTrack со встроенным (embedded) Hub, Teamcity, Gitlab имеется возможность сделать следующие интеграции:

- YouTrack c TeamCity - Build Server Integration
- YouTrack с Gitlab - VCS Integrations

![](https://habrastorage.org/webt/vr/v5/x2/vrv5x28ehfkx5obctlxb_09chpy.png)

#### Интеграция YouTrack c TeamCity - Build Server Integration

Создадим одинаковый проект в  Youtrack, Teamcity и Gitlab.
![](https://habrastorage.org/webt/1b/se/fl/1bseflarn7nw7ehpy3slr2mmffk.png)

В Teamcity подключен VCS, который отображается на скриншоте с Gitlab.

![](https://habrastorage.org/webt/qq/8j/do/qq8jdogufutf6f7wxqzfken-itg.png)

![](https://habrastorage.org/webt/us/t0/kp/ust0kpy5s7ncsndgsbum8imyuuc.png)

В YouTrack создана issue c номером P1-1.

Задача: необходимо закрывать Issue в YouTrack после успешной сборки в Teamcity, используя номер Issue и команду Fixed в commit.

![](https://habrastorage.org/webt/ih/2r/3i/ih2r3imuqxlixv5bbk6unr9zlag.png)

Переходим в Build Server Integration. Подключаем Build Server в YouTrack.

![](https://habrastorage.org/webt/jv/b5/8-/jvb58-x4myajovghbrpifq5qjaw.png)

Интеграцию YouTrack c TeamCity нужно делать попроектно. Для этого в поле **Main YouTrack project** указываем проект в YouTrack. В поле **TeamCity project** указываем проект в TeamCity. В поле **TeamCity build** указываем таску в TeamCity, в которой будет собираться ваш проект. Можно выставить **Fixed in build** в поле **Add build numbers to the set of values in a custom field**. В YouTrack будет указано в каком билде исправлена эта issue.

![](https://habrastorage.org/webt/ux/ac/6i/uxac6iasxoltiqvueloetuf7chq.png)

Если вы видите сообщеие **The VCS user name does not match any user in YouTrack.**, то вам нужно проверить и сравнить **VCS user name** в git user.name и в настройках вашего пользователя.

Мне git log показал мои старые данные от прошлых экспериментов.

```
git log 
commit 34ea8e39320e668db4066aa98b425c9fa9f7f7ef (HEAD -> master, origin/master)
Author: Anton Patsev <user1@group1.com>
Date:   Wed Jul 29 11:45:23 2020 +0600

    text text1 #P1-1 Fixed this issue
```

Вам нужно либо создать нужно пользователя или добавить name и email в профиль пользователя.

Так же если вы нажмете на коммит **4643208d**, то вы перейдете в TeamCity где увидите ошибку *Unknown (none of TeamCity users defined **user1** username in their VCS username settings)*. TeamCity пишет что у него нет пользователя с именем user1. Поэтому нужно его добавить в TeamCity.

![](https://habrastorage.org/webt/ot/mr/ct/otmrct6qsyl1b6a3vploknpu-c4.png)

Если у вас **Unknown command: Fixed**, то это значит что ваш пользователь, который пытается закрыть issue, не имеет права это делать.

![](https://habrastorage.org/webt/3w/1t/gy/3w1tgygrqxbairaz4ywkvlly2du.png)

Добавляем юзеру user1 роль Developer в проекте project1.

![](https://habrastorage.org/webt/qy/7t/yy/qy7tyyg0o2vxpu_sfwnerfd0she.png)

После этого делаем еще один текстовый коммит, который закрывает issue. Мы видим что ошибок нет, state перешел в состояние Fixed, в поле Fixed in Build проставилась номер таски в TeamCity, в котором закрылась эта issue.

#### VCS Integrations

Чтобы продемонстрировать чистую VCS Integrations отключим интеграцию с TeamCity.

![](https://habrastorage.org/webt/ut/j7/u5/utj7u5ehvrgfyuetmrbqo43srqk.png)

В **Main YouTrack project** укажите нужный проект, выбрите Gitlab, в Repository URL нужно указать полный URL до нужного репозитория, в Personal access token нужно указать api токен. Нажимаем **Generate token** и откроется Access Tokens в User Settings вашего текущего пользователя.

![](https://habrastorage.org/webt/1h/ct/xv/1hctxvvsrsemwzdmby42k15jjri.png)

Генерируем токен для api. Почему не подходят другие scope? Потому что Youtrack создает на сервере Gitlab webhook.

Указываем токен в **Personal access token**. И создаем интеграцию.

После создания интеграции проверяем что Youtrack создал на сервере Gitlab webhook. Идем в *projects1 -> Webhook Settings*.

Youtrack создает на сервере Gitlab webhook.

![](https://habrastorage.org/webt/fl/ja/pf/fljapfohd4xnk55qdam9u-lqxrm.png)

Далее нужно разрешить webhook в локальной сети. Далее переходим: **Admin -> Settings -> Network -> Outbound Requests -> Allow requests to the local network from hooks and services**

Если этого не сделать, у вас будут ошибки: [**Unable to save project. Error: Import url is blocked: Requests to the local network are not allowed**](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/57948)

Сделаем коммит в репозиторий.

![](https://habrastorage.org/webt/jk/1z/rx/jk1zrx0yddmzdft6qwyijmrxaxu.png)

В YouTrack увидем что issue закрылся и увидем всплываеющее сообщение об изменении VCS.

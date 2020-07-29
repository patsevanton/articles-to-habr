Интеграция Youtrack, Teamcity, Gitlab без Hub.

В этом посте будет рассказано о том как сделать интеграцию Youtrack, Teamcity, Gitlab без Hub.

В следующей посте будет рассказано о том как сделать интеграцию Youtrack, Upsource, Teamcity, Gitlab c Hub.

**[YouTrack](https://www.jetbrains.com/ru-ru/youtrack/)** — коммерческая система отслеживания ошибок, программное обеспечение для управления проектами. YouTrack поддерживает поисковые запросы, автодополнение, манипуляцию с наборами задач, настройку набора атрибутов задачи, создание пользовательских рабочих процессов и реализует подход, основанный на преимущественном использовании клавиатуры.

**[TeamCity](https://www.jetbrains.com/ru-ru/teamcity/)** — интеллектуальный сервер непрерывной интеграции.

[**GitLab**](https://about.gitlab.com/) — веб-инструмент жизненного цикла DevOps с открытым исходным кодом, представляющий систему управления репозиториями кода для Git с собственной вики, системой отслеживания ошибок, CI/CD пайплайном и другими функциями.

![](https://habrastorage.org/webt/mj/k_/t8/mjk_t877n2mk_dhbaisx8mjg8yy.png)

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

После установки Gitlab, YouTrack, Teamcity без Hub имеется возможность сделать следующие интеграции:

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

![](https://habrastorage.org/webt/yi/hv/vp/yihvvpkwi7kgp3jjto0q75jul0a.png)

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

![](https://habrastorage.org/webt/o3/ma/fp/o3mafpaszhy0fzncpqesnaxz-yq.png)



https://gitlab.com/gitlab-org/gitlab-foss/-/issues/57948
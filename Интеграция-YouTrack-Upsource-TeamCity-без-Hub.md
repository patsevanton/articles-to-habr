Интеграция Youtrack, Upsource, Teamcity, Gitlab без Hub.



**[YouTrack](https://www.jetbrains.com/ru-ru/youtrack/)** — коммерческая система отслеживания ошибок, программное обеспечение для управления проектами. YouTrack поддерживает поисковые запросы, автодополнение, манипуляцию с наборами задач, настройку набора атрибутов задачи, создание пользовательских рабочих процессов и реализует подход, основанный на преимущественном использовании клавиатуры.

**[TeamCity](https://www.jetbrains.com/ru-ru/teamcity/)** — интеллектуальный сервер непрерывной интеграции.

**[Upsource](https://www.jetbrains.com/ru-ru/upsource/)** инструмент для код-ревью, анализа истории проектов, организации совместной работы или совершенствования своих навыков разработки.

[**GitLab**](https://about.gitlab.com/) — веб-инструмент жизненного цикла DevOps с открытым исходным кодом, представляющий систему управления репозиториями кода для Git с собственной вики, системой отслеживания ошибок, CI/CD пайплайном и другими функциями.

![](https://habrastorage.org/webt/uu/co/2g/uuco2gdwp7bn51qiaeduz-fpel8.png)

#### Требования к интеграции.

Необходимо чтобы все инструменты резолвились по DNS. Если инструменты не будут резолвиться по DNS, то у вас часть функционала не будет работать.

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

**Upsource**

```
yum install -y mc unzip
wget https://download-cf.jetbrains.com/upsource/upsource-2020.1.1782.zip
unzip upsource-2020.1.1782.zip 
cd upsource-2020.1.1782/bin/
./upsource.sh start
```


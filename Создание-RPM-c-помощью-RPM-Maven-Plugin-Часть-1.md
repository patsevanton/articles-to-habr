Этот пост призван обеспечить быстрый и простой способ упаковки файлов в RPM с помощью плагина rpm-maven-plugin. Шаги представлены таким образом, что человек, не имеющий предварительных знаний о maven, все же сможет начать использовать этот инструмент для упаковки файлов в RPM.

Исходный код проекта находитя по адресу: https://github.com/unixutils/Code-snippets/tree/master/Build_rpm

### Краткое введение в Maven

Maven, на самом базовом уровне, является инструментом сборки для упаковки файлов в различные форматы, такие как zip, tar, jar, war и т. Д., Из проекта и обычно используется при упаковке Java-проектов в виде jar-файлов при разработке программного обеспечения. Хотя Maven может сделать больше, чем просто создавать пакеты, в этом посте мы расскажем вам только о шагах по созданию RPM с использованием Maven, чтобы мы не сбились с пути.

### Установка Maven

После того, как вы включили репозиторий EPEL в своей системе, установить maven так же просто, как выполнить команду `yum install maven`. Вы также можете установить maven, используя шаги, рекомендованные на [веб-сайте Apache Maven](https://maven.apache.org/install.html).

### Что такое pom.xml?

Чтобы достичь нашей простой цели (goal) - просто упаковать файлы в RPM, все, что нам нужно знать об этом файле, это то, что это XML-файл, который содержит информацию о проекте и сведения о конфигурации, используемые Maven для создания проекта. И проект - это не что иное, как каталог, содержащий файлы, которые вы пытаетесь упаковать в RPM.

Чтобы лучше понять, что происходит в pom.xml, мы разработаем пример, где мы создаем RPM со следующими целями.

- После установки этого RPM в системе образец файла .ini (давайте назовем его sample.ini) должен быть установлен как /opt/myapp.ini.
- Пользователь и группа для этого файла должны быть myappid и myappgroup соответственно
- Файл должен иметь 755 разрешений.
- Сценарий с именем `preinstall.sh` должен быть запущен до установки /opt/myapp.ini. Мы должны использовать этот скрипт, чтобы проверить, существуют ли пользователь и группа, и если нет, мы должны создать их.
- Сценарий с именем `postinstall.sh` должен запускаться после установки
  /opt/myapp.ini. Мы должны использовать этот скрипт для перезапуска службы httpd.

**ПРИМЕЧАНИЕ** : когда вы запускаете команду `mvn package` для начала упаковки, mvn ищет инструкцию из pom.xml в текущем рабочем каталоге. Поэтому в текущем рабочем каталоге ожидается файл pom.xml. В том же каталоге в идеале должны содержаться файлы проекта, которые вы пытаетесь упаковать в RPM.

Наш каталог проектов выглядит так до сборки RPM.

```
[admin@unixutils project]# tree
.
├── ini
│   └── sample.ini
├── pom.xml
└── scripts
    ├── postinstall.sh
    └── preinstall.sh

2 directories, 4 files
```

Для достижения наших целей (goal) мы напишем наши цели (target) в виде инструкций в виде тегов XML в файле нашего проекта pom.xml (Код ниже). Файл pom.xml обычно начинается с тега `<project>` и заканчивается тегом `</ project>` . Этот тег определяет пространство имен XML для документа, и он не изменяется. То же относится и к `<modelVersion>4.0.0</ modelVersion >`, который является версией дескриптора, используемой в файле для maven, чтобы иметь возможность понять содержимое файла pom.xml. Это тоже не изменится. Не нужно много думать о том, как эти два тега влияют на сборку, однако вы всегда можете посмотреть дальше на [Apache Maven - Введение в POM](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html).

`<groupId>`, `<artifactId>`, `<version>` - это три тега, которые однозначно идентифицируют пакет после его сборки . Каждый пакет, созданный с Maven, будет иметь это. Пакет обычно называется Артефактом, и компании / сообщества создают Артефакты и хранят их в частном репозитории, таком как [Artifactory](https://jfrog.com/artifactory/), или в [общедоступном репозитории Maven](https://mvnrepository.com/). Предположим, что компания с именем company.com создает пакеты, каждый из которых будет иметь уникальный artifactId, версию и общий groupId, чтобы указать, что он был создан группой компании. GroupId - это, как правило, обратное доменное имя компании (com.company). Таким образом, на каждый пакет, помещаемый в репозиторий, ссылаются однозначно, используя эти три тега. Это оборачивает основы pom.xml. Для создания проекта Java и упаковки файлов .jar это все, что нужно. На этом этапе, если команда mvn build запущена, mvn автоматически ищет исходные файлы в (`src/main/java`) в рабочем каталоге, компилирует их и упаковывает как .jar. Однако у нас нет ни структуры каталогов (`src/main/java`), ни файлов java для компиляции и сборки. Все, что у нас есть, это некоторые другие файлы (которые не нуждаются в компиляции), что мы намереваемся упаковать как rpm. Для этого мы воспользуемся плагином rpm-maven-plugin. **Подобные плагины - это не что иное, как пакеты или артефакты, которые были созданы и предоставлены в официальном репозитории Maven** сообществами или компаниями в качестве открытого источника. Как мы только что узнали, артефакт уникально идентифицируется тремя тегами, а именно: `<groupId>`, `<artifactId>`,`<version>`. Чтобы подключить плагин и использовать внешний артефакт / пакет, нам нужно вызвать плагин по его `groupId` , `artifactId` и `version` и включите его втеги  `<build></build>`.

В приведенном ниже файле проекта pom.xml мы будем использовать [RPM Maven plugin](https://mvnrepository.com/artifact/org.codehaus.mojo/rpm-maven-plugin/2.2.0), для которого значениями `<groupId>`, `<artifactId>`,`<version>` являются `org.codehaus.mojo`, `rpm-maven-plugin`, `2.0.1` соответственно (строки с 12 по 14 ниже в pom.xml). Теги, которые идут после этого, являются тем, что определил разработчик плагина. Использование этого плагина можно найти на [официальной странице разработчика плагина](https://www.mojohaus.org/rpm-maven-plugin/usage.html). Основываясь на документе об использовании, мы определяем конфигурацию (строки с 15 по 38 в нижеприведенном pom.xml) , которая поясняется ниже.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.unixutils</groupId>
    <artifactId>unixutils-test-rpm</artifactId>
    <version>1.0.0</version>
    <build>
      <plugins>
        <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>rpm-maven-plugin</artifactId>
        <version>2.0.1</version>
        <executions>
         <execution>
         <goals>
         <goal>rpm</goal>
         </goals>
        </execution>
       </executions>
          <configuration>
                    <copyright>2019, UnixUtils</copyright>
                    <group>Development/Tools</group>
                    <release>1.0.0</release>
                    <mappings>
                        <mapping>
                            <directory>/opt/myapp.ini</directory>
                            <username>myuser</username>
                            <groupname>mygroup</groupname>
                            <filemode>755</filemode>
                            <sources>
                                <source>
                                    <location>${project.basedir}/ini/sample.ini</location>
                                </source>
                            </sources>
                        </mapping>
                    </mappings>
                    <preinstallScriptlet>
                        <scriptFile>${project.basedir}/scripts/preinstall.sh</scriptFile>
                    </preinstallScriptlet>
                   <postinstallScriptlet>
                        <scriptFile>${project.basedir}/scripts/postinstall.sh</scriptFile>
                        <fileEncoding>utf-8</fileEncoding>
                   </postinstallScriptlet>
          </configuration>
         </plugin>
       </plugins>
     </build>
</project>
```

Внутри блока `<execution>` есть блок `<goal>`, который определяет, что создание RPM, который будет артефактом / пакетом (конечным результатом этой сборки). Блок `<mapping>`  определяет, что файлы из местоположения / рабочего каталога проекта, указанного в разделе `<source>`, упаковываются в rpm, а когда rpm установливается, он развертывается по пути, указанному враздел `<directory>`. имя пользователя, имя группы и разрешения указываются в разделе `<username>`, `<groupname>`, `<filemode>` разделы соответственно.

Обратите внимание, что `${project.basedir}` - это предопределенная переменная maven, которая обозначает текущий рабочий каталог, в котором находятся файлы проекта. При желании `<preinstallScriplet>` можно использовать для определения пути к сценарию из местоположения проекта, который будет запущен до установки rpm. `<postinstallScriptlet>` можно использовать для определения пути к сценарию из местоположения проекта, который будет запускаться после установки rpm.

```
mvn package
```

Приведенная выше команда запускает сборку пакета RPM и выдает их в расположении ниже

```
project/target/rpm/unixutils-test-rpm/RPMS/noarch/unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm
```

Теперь давайте попробуем установить созданный пакет.

```
[admin@unixutils project]# cd target/rpm/unixutils-test-rpm/RPMS/noarch
[admin@unixutils noarch]# yum localinstall unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm
Loaded plugins: fastestmirror
Examining unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm: unixutils-test-rpm-1.0.0-1.0.0.noarch
Marking unixutils-test-rpm-1.0.0-1.0.0.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package unixutils-test-rpm.noarch 0:1.0.0-1.0.0 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

====================================================================================================================================================================
 Package                                Arch                       Version                         Repository                                                  Size
====================================================================================================================================================================
Installing:
 unixutils-test-rpm                     noarch                     1.0.0-1.0.0                     /unixutils-test-rpm-1.0.0-1.0.0.noarch                      60

Transaction Summary
====================================================================================================================================================================
Install  1 Package

Total size: 60
Installed size: 60
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
running command [id -u myuser]
id: myuser: no such user
myuser does not exist..
creating myuser..
running command [getent group mygroup]
mygroup does not exist..
creating mygroup..
  Installing : unixutils-test-rpm-1.0.0-1.0.0.noarch                                                                                                            1/1
restarting HTTPD services
  Verifying  : unixutils-test-rpm-1.0.0-1.0.0.noarch                                                                                                            1/1

Installed:
  unixutils-test-rpm.noarch 0:1.0.0-1.0.0

Complete!
[admin@unixutils noarch]#
```

Обороты установлены успешно. Сценарии postinstall и preinstall также были запущены. Здесь мы также видим, что INI-файл развернут, как задумано, с правильными правами доступа и владельцем.

```
[admin @ unixutils project] # ll /opt/myapp.ini/
всего 4
-rwxr-хт-х. 1 myuser mygroup 60 5 января 17:39 sample.ini
[admin @ unixutils project] #
```

В этом посте мы рассмотрели основы использования maven в качестве инструмента для создания RPM с `RPM Maven Plugin`. В следующих статьях мы рассмотрим различные функции, доступные в этом плагине, и рассмотрим различные сценарии, такие как развертывание обновления RPM.
В этом посте будет описана настройка автоматизации HotFix в Maven проектах с использованием Teamcity.

Перед тем как перейдем к теме, хочется затронуть важную и сложную тему — **версионирования** программного обеспечения. Кратко о Semver можно понять на этом скриншоте. ![](http://www.8bytes.net/wp-content/uploads/2017/12/semver.png)

Подробнее можно прочитать по ссылке: [1](http://www.8bytes.net/2017/12/11/semver-semanticheskoe-versionirovanie/).

Все настройки описанные в этом посте опираются на [Semver](https://semver.org/lang/ru/) и [Trunk-Based Development](https://trunkbaseddevelopment.com/).

В Trunk-Based Development для каждого релиза нужно создать свой бранч. Все изменения (hotfix) в рамках этого релиза коммитятся в этот бранч.



В рамках этого поста автоматизируем следующие вещи:

- CI сборка

- Создание нового релиза
- Создание бранча для релиза
- Изменение bugfix версии

![](https://trunkbaseddevelopment.com/branch-for-release/branch_for_release.png)



Требования:

- Git репозиторий для хранения вашего кода. В посте будет использоваться репозиторий https://gitlab.com/anton_patsev/automation-maven-hotfix.
- Teamcity сервер и агент. Вы можете поднять свой локальный Teamcity сервер и агент с помощью [docker-compose](https://github.com/JetBrains/teamcity-docker-samples/blob/master/compose-ubuntu/docker-compose.yml)
- Там где у вас Teamcity агент, должны быть установлены java, maven, git



Создадим в Teamcity проект "Automation Maven Hotfix" и создадим там 4 таски.

- CI Build (CI сборка)

- Create branch for release (Создание бранча для релиза)

- Maven increment bugfix (Изменение bugfix версии))

- Maven release (Создание нового релиза)

  

Скриншот проекта:

![](https://habrastorage.org/webt/tr/w7/wm/trw7wm4ezmga_h5opy2l6jptsjy.png)

### **Общие настройки**

Во всех тасках необходимо выставить галку "**Clean build: Delete all files in the checkout directory before the build**", так как при отстутствии этой галки у меня появлялись ошибки.

Создаем единственный VCS. Особенности VCS обведенны красным.

![](https://habrastorage.org/webt/a2/pk/in/a2pkinpahr-3q5viu9b1ptpoboi.png)

Обычно VCS испольуют схему HTTS. В **Branch specification:** указано смотреть все бранчи и все теги:

```bash
+:refs/heads/*
+:refs/tags/*
```

Необходимо создать 4 Configuration Parameters.

- BRANCH_FOR_INCREMENT
- TAG_FROM_VERSION
- TEAM_USER
- TEAM_USER_EMAIL

Поле value в BRANCH_FOR_INCREMENT и TAG_FROM_VERSION нужно оставить пустым.

![](https://habrastorage.org/webt/i6/hb/av/i6hbavy70skqahcff1lapsjqdh4.png)

Необходимо закачать/добавить приватный ключ. Во всех тасках кроме CI Build необходим приватный ключ.

![](https://habrastorage.org/webt/yv/qf/nc/yvqfncwkbsoshgk3vay6l3v2rvu.png)

В каждой таске кроме CI Build в разделе Build Features нужно подключить приватный ключ.

Пример для **Maven release**

![](https://habrastorage.org/webt/i6/hb/av/i6hbavy70skqahcff1lapsjqdh4.png)

### CI Build**.

В таске **CI Build** всего лишь один шаг **mvn clean test**

![](https://habrastorage.org/webt/tu/16/h-/tu16h-8v82uscogf9wx3nqnccua.png)

### **Maven release**

В таске **Maven release** 2 шага. Первый шаг проверяет что бранч это **master**. Если бранч не **master**, то таска падает.

```bash
BRANCH=$(git branch | grep \* | cut -d ' ' -f2)
echo "$BRANCH"
if [[ "$BRANCH" != "master" ]]; then
  echo 'Branch is not master';
  echo 'Aborting';
  exit 1;
fi
```

![](https://habrastorage.org/webt/hw/zh/vt/hwzhvtdrnjtw1rdkug7j0g6_hvm.png)

Второй шаг это стандартный **mvn release:prepare** c опцией **--batch-mode**

![](https://habrastorage.org/webt/d-/7w/sx/d-7wsxiobalnzjpsfqtwerogi5k.png)

### **Create branch for release**

Чтобы создать hotfix для релиза нужно создать бранч. Этим занимается таска **Create branch for release**. У нее 2 шага.

Первый шаг проверяет что бранч не **master**, и второе проверяет что версия в файле **pom.xml** не содержало слово **SNAPSHOT**

```bash
BRANCH=$(git branch | grep \* | cut -d ' ' -f2)
echo "$BRANCH"
if [[ "$BRANCH" == "master" ]]; then
  echo 'Branch is master';
  echo 'Aborting';
  exit 1;
fi

echo "Get version package from pom.xml"
version=`python -c "import xml.etree.ElementTree as ET; print(ET.parse(open('pom.xml')).getroot().find('{http://maven.apache.org/POM/4.0.0}version').text)"`

echo "Check SNAPSHOT"
if [[ $version == "*SNAPSHOT*" ]]; then
    echo "******************* W A R N I N G *************************"
    echo "************ You are create branch for SNAPSHOTS ******************"
    echo "***********************************************************"
    exit 1
fi
```

![](https://habrastorage.org/webt/ap/ky/xp/apkyxp675ywbv6m2lrjchon7gds.png)

Второй шаг изменяет в developerConnection схему подключения с HTTPS на GIT.

```bash
# Здесь получаем developerConnection из файла pom.xml
developerConnection=$(xmllint -xpath "/*[local-name() = 'project' ]//*[local-name() = 'developerConnection']/text()" pom.xml | sed  's|scm:git:ssh://||')
echo developerConnection
echo $developerConnection
# Здесь меняем / на : в URL для git_remote_url
git_remote_url=$(echo $developerConnection| sed 's/gitlab.com\//gitlab.com:/g')
echo git_remote_url
echo $git_remote_url

git remote set-url origin $git_remote_url

# Если вы не используете ввстроенную возможность Teamcity получения user и email из ~/.gitconfig, то можно указать их здесь
echo 'git config user.name %TEAM_USER%'
git config user.name %TEAM_USER%
echo 'git config user.email %TEAM_USER_EMAIL%'
git config user.email %TEAM_USER_EMAIL%

# Здесь получаем версию из файла pom.xml
echo "Get version package from pom.xml"
version=`python -c "import xml.etree.ElementTree as ET; print(ET.parse(open('pom.xml')).getroot().find('{http://maven.apache.org/POM/4.0.0}version').text)"`
echo $version

# Почему-то без fetch выдавало ошибку.
git fetch

if [ `git branch -a | egrep "${version}$"` ]
then
    echo "Branch exists"
    exit 1
fi

# Создаем бранч той версии, который был в файле pom.xml
echo "Create branch"
git checkout -b $version

# Чистый git всегда предлагает настроить политику отправки.
git config --global push.default simple

# Пушим в ветку совпадающую с версией в pom.xml
echo "Push release branch"
git push --set-upstream origin $version
```

![](https://habrastorage.org/webt/3e/3j/yd/3e3jydfdnlg6avoxzwwy5kubmke.png)

### **Maven increment bugfix**

Таска состоит из 6 частей. Можно было отрефакторить, но и так работает.

Первый шаг - проверка что бранч не **master**. Если бранч **master** таска падает. 

```bash
BRANCH=$(git branch | grep \* | cut -d ' ' -f2)
echo "$BRANCH"
if [[ "$BRANCH" == "master" ]]; then
  echo 'Branch is master';
  echo 'Aborting';
  exit 1;
fi

# Здесь получаем версию из файла pom.xml
echo "Get version package from pom.xml"
BRANCH=`python -c "import xml.etree.ElementTree as ET; print(ET.parse(open('pom.xml')).getroot().find('{http://maven.apache.org/POM/4.0.0}version').text)"`
# Приходится делать checkout на нужный бранч.
# Иначе git status показывает detached к нужному бранчу.
# Нужно чтобы git status показывал просто бранч
git checkout $BRANCH
# Экспортируем переменную bash в переменную Teamcity для дальнейшего использования.
echo "##teamcity[setParameter name='BRANCH_FOR_INCREMENT' value='$BRANCH']"
```

![](https://habrastorage.org/webt/aw/kd/o9/awkdo9irrlxyqfutph53lgtusqc.png)

Второй Maven шаг изменение bugfix версии в файле pom.xml.

**Goals:** у maven все одной строкой

```bash
build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit
```

![](https://habrastorage.org/webt/hg/j-/xy/hgj-xyeo4zgh7zucznk7kfp_qeq.png)

Третий шаг - вывод информации Git status и другие:

```bash
echo 'cat pom.xml'
cat pom.xml
echo 'git status'
git status
echo 'git remote -v'
git remote -v
echo 'git branch'
git branch
```

![](https://habrastorage.org/webt/y2/nl/l3/y2nll3lgpjtixjf5dwohvdoqevm.png)

Четвертый шаг изменяет в developerConnection схему подключения с HTTPS на GIT.

И пушит изменения в ветку, указанную в Teamcity переменной %BRANCH_FOR_INCREMENT%

```bash
# Здесь получаем developerConnection из файла pom.xml
developerConnection=$(xmllint -xpath "/*[local-name() = 'project' ]//*[local-name() = 'developerConnection']/text()" pom.xml | sed  's|scm:git:ssh://||')
echo developerConnection
# Здесь меняем / на : в URL для git_remote_url
git_remote_url=$(echo $developerConnection| sed 's/gitlab.com\//gitlab.com:/g')
echo git_remote_url
echo $git_remote_url

git remote set-url origin $git_remote_url

# Если вы не используете ввстроенную возможность Teamcity получения user и email из ~/.gitconfig, то можно указать их здесь
echo 'git config user.name %TEAM_USER%'
git config user.name %TEAM_USER%
echo 'git config user.email %TEAM_USER_EMAIL%'
git config user.email %TEAM_USER_EMAIL%
echo 'git add .'
git add .
echo 'git commit -m "Increment bugfix"'
git commit -m "Increment bugfix"

git push --set-upstream origin %BRANCH_FOR_INCREMENT%
```

![](https://habrastorage.org/webt/9e/kx/aa/9ekxaa0fnm3sfve2iem246iihuy.png)

Пятый шаг получает из файла **pom.xml** версию и устанавливает ее в **Teamcity** переменную **TAG_FROM_VERSION**. Заметьте что версия из файла **pom.xml** без буквы v впереди. А тег, на основе этой версии уже с буквой v вначале.

```bash
echo "Get version package from pom.xml"
VERSION_AFTER_CHANGE=`python -c "import xml.etree.ElementTree as ET; print(ET.parse(open('pom.xml')).getroot().find('{http://maven.apache.org/POM/4.0.0}version').text)"`
echo $VERSION_AFTER_CHANGE
echo "##teamcity[setParameter name='TAG_FROM_VERSION' value='v$VERSION_AFTER_CHANGE']"
```

![](https://habrastorage.org/webt/4a/7t/2y/4a7t2yc_wgfyhwtmqgw_pnecquo.png)

Шестой шаг - тегирование **bugfix** версии. Делается это с помощью **Maven** с нужной опцией в **Goal**.

Опция **Goals**:

```bash
-Dtag=%TAG_FROM_VERSION% scm:tag
```

![](https://habrastorage.org/webt/eq/lo/32/eqlo32gfwn7hx4nnkoyhl9ir5y0.png)
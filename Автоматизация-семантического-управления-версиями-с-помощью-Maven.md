Вы используете семантический подход к управлению версиями? Вы используете gitflow? Скорее всего, вы знакомы с процессом корректировки версий, создания веток, слияния с master/dev, повторной корректировки версий, борьбы с конфликтами слияния,…

В этой статье я кратко объясню процесс выпуска, который мы, по сути, используем для наших библиотек, и то, как мы его автоматизировали. Обычно мы используем подход CI/CD, используя номера сборок, но для наших библиотек мы решили использовать семантическое управление версиями. Я столкнулся с утомительным процессом выпуска, который сопровождает это в нескольких компаниях, и теперь я наконец нашел решение.

<cut />

Эта статья посвящена Maven, но есть и много альтернатив [Gradle](https://plugins.gradle.org/search?term=gitflow).

Пример проекта можно найти на нашей странице [GitHub](https://github.com/viesure/blog-gitflow-maven).

### **Семантическое управление версиями и Git**

[Семантическое управление версиями](https://semver.org/) - это система классификации ваших выпусков. Я уверен, что вы видели такие номера версий, как 1.6.4, 1.7.10, 1.12.2 и другие. Эти цифры обозначают MAJOR.MINOR.PATCH (МАЖОРНАЯ.МИНОРНАЯ.ПАТЧ)

Кроме того, существуют версии SNAPSHOT, которые выглядят одинаково, но с добавлением «-SNAPSHOT» в конце, например 1.14.4-SNAPSHOT.

Типичный процесс выпуска состоит из следующих шагов:

1. Создайте release ветку из ветки разработки (здесь и далее ветка разработки - development branch).

2. Измените версию во всех файлах pom.xml с SNAPSHOT (1.2.3-SNAPSHOT) на non-SNAPSHOT (1.2.3) в release ветке.

3. Увеличьте версию SNAPSHOT на ветке develop(1.2.4-SNAPSHOT).

4.   Когда все будет сделано для release, объедините release ветку с master веткой. Это актуальный релиз.

5. Слейте master ветку или ветку release обратно в ветку разработки.

Есть несколько вариантов этого процесса с разными таймингами / стратегиями слияния, но он сводится к следующему: версия должна быть изменена на два разных значения в двух разных ветвях, но обе ветки должны иметь одну и ту же историю, чтобы предотвратить конфликты слияния.

Хотя эти шаги не являются сложными сами по себе, они утомительны и подвержены ошибкам, когда вам приходится выполнять их часто. В то же время они требуют много размышлений, чтобы предотвратить merge конфликты.

### **Каковы были мои цели?**

·    Я хочу автоматизировать весь процесс, чтобы не беспокоиться об ошибках.

·    Текущее состояние master ветви никогда не должно содержать версию SNAPSHOT.

·    Я хочу подключить его к конвейерам CI, но с ручным запуском.

·    Я никогда не хочу иметь дело с merge конфликтами.

·    Я также хочу использовать его для hotfixes (которые происходят из master ветки).
 
### **Плагин gitflow-maven**

Возможно, вы знаете плагин версий maven, который легко позволяет устанавливать версии внутри ваших файлов pom.xml. Однако для этого мне необходимо вручную ввести новую версию. Я хочу, чтобы мое решение автоматически выводило следующую версию на основе текущей версии.

К счастью, уже есть инструменты, помогающие достичь моих целей. Поскольку проблема довольно распространена, несколько человек пытались ее решить. Однако из всех найденных мною подключаемых модулей и инструментов поддерживается только один, при этом предоставляющий достаточно параметров конфигурации: [gitflow-maven-plugin](https://aleksandr-m.github.io/gitflow-maven-plugin/).

В нем есть именно то, что мне нужно:

·    Увеличение номеров версий.

·    Создание release ветки.

·    Создание ветки hotfix.

·    Объединение (Merging) веток.

Кроме того, выпуски возможны непосредственно из ветки разработки без дополнительной ветки выпуска. Это очень удобно, поскольку наша компания в любом случае использует CI/CD, что означает, что ветвь разработки всегда находится в готовом к выпуску состоянии.

Все это можно сделать, выполнив цели (goals) maven. Нам нужно только указать плагину, как себя вести и какое число увеличивать.

### **Некоторые примеры:**

Следующие команды показывают несколько примеров, которые я использовал при оценке плагина.

```
$ mvn gitflow:release-start -B
```

Создает ветку release из ветки разработки без ввода данных пользователем (`-B` Batch Mode)

```
$ mvn gitflow:release
```

Создает релиз прямо из ветки разработки. Плагин объединяет текущее состояние разработки с master веткой и увеличивает номер версии в ветке разработки.

В этом примере нет флага пакетной обработки, что означает, что пользователю предлагается ввести всю недостающую информацию.

```
$ mvn gitflow:hotfix-start -B
```

```
$ mvn gitflow:hotfix-finish -B -DhotfixVersion=1.8.9b 
```

На первом этапе создает ветку hotfix из master ветки, затем выпускает исправление с версией исправления 1.8.9b на втором этапе. Все цели плагина могут быть выполнены в любой ветке. Из-за этого я должен указать плагину, какое исправление я хочу выпустить.


### **Конфигурация**

Добавить зависимости плагина достаточно просто, просто добавьте следующее в плагины сборки poms:

 ```
<build>
    <plugins>
        <plugin>
            <groupId>com.amashchenko.maven.plugin</groupId>
            <artifactId>gitflow-maven-plugin</artifactId>
            <version>1.13.0</version>
            <configuration>
                <!-- optional configuration -->
            </configuration>
        </plugin>
    </plugins>
</build>
 ```

Последнюю версию можно найти на [GitHub](https://github.com/aleksandr-m/gitflow-maven-plugin/releases) или в [maven central](https://mvnrepository.com/artifact/com.amashchenko.maven.plugin/gitflow-maven-plugin).

Дополнительная настройка хорошо документирована на странице плагинов GitHub. Вот пример конфигурации с выделением некоторых параметров конфигурации:

 ```
<configuration>
    <!-- We use maven wrapper in all our projects instead of a local maven installation -->
    <mvnExecutable>./mvnw</mvnExecutable>
 
    <!-- Don’t push to the git remote. Very useful for testing locally -->
    <pushRemote>true</pushRemote>
 
    <!-- Set to true to immediately bump the development version when creating a release branch -->
    <commitDevelopmentVersionAtStart>false</commitDevelopmentVersionAtStart>
 
    <!-- Which digit to increas in major.minor.patch versioning, the values being 0.1.2 respectively.
         By default the rightmost number is increased.
         Pass in the number via parameter or profile to allow configuration,
         since everything set in the file can't be overwritten via command line -->
    <versionDigitToIncrement>${gitflowDigitToIncrement}</versionDigitToIncrement>
 
    <!-- Execute mvn verify before release -->
    <preReleaseGoals>verify</preReleaseGoals>
    <preHotfixGoals>verify</preHotfixGoals>
 
    <!-- Configure branches -->
    <gitFlowConfig>
        <productionBranch>master</productionBranch>
        <!-- default is develop, but we use development -->
        <developmentBranch>development</developmentBranch>
    </gitFlowConfig>
</configuration>
 ```

Это отличный плагин, все работает безупречно, когда я запускаю его локально. Есть несколько избранных настроек, чтобы работать с Gitlab CI.

### **Автоматизация с помощью Gitlab CI**

Мы запускаем конвейер CI/CD с использованием Gitlab CI для наших проектов, поэтому каждый новый commit в нашей ветке разработки приводит к созданию snapshot, а merge для master - к release.

Как описано выше, моя цель - автоматизировать разработку, master merge и связанные с этим обновления версий, а также hotfixes.

Я использую конвейеры Gitlab CI, чтобы упростить процесс и в то же время сделать его доступным для моих коллег. Вот пример:

![](https://habrastorage.org/webt/zk/vo/3h/zkvo3hl6bdc8hy5rmsufiz09qwa.png)

Вот как выглядит конвейер ветвления разработки в моем примере с дополнительными этапами release. Поэтому, если я хочу создать release, я просто запускаю этап release, и он автоматически устанавливает версию без snapshot, объединяет (merge) ее с master и переводит версию в ветке разработки на следующую версию snapshot. Без каких-либо дополнительных действий с моей стороны.

### **Заставляем git работать в Gitlab CI**

Если вы пробовали использовать команды git внутри Gitlab CI, то знаете: это не совсем просто, поскольку у исполнителей CI нет учетных данных git.

Для простоты я использую токены доступа с правами write_repository. Я назначил для этого специального пользователя, чтобы видеть, какие коммиты были созданы автоматически.

Я ввожу токен через переменную среды GITLAB_TOKEN, которую я установил как protected, и помечаю ветки `development`, `release/*` `и hotfix/*` как защищенные (protected). Таким образом, только сборки на тех ветках имеют доступ к токену.

Я вручную устанавливаю git remote на runner, чтобы среда CI могла отправлять данные в репозиторий. Я использую переменные, предоставленные Gitlab, поэтому мне не нужно ничего жестко кодировать:

```
$ git remote set-url --push origin "https://oauth2:${GITLAB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
```

С помощью этого git команды могут выполняться с использованием учетных данных введенного токена доступа. Git также требует некоторой информации о «пользователе», выполняющем действия git. Желательно, чтобы пользователь git соответствовал пользователю, чей токен доступа мы используем. В моем примере:

```
$ git config user.name "Gitlab CI"
$ git config user.email gitlab-ci@viesure.io
```

Благодаря этому я могу легко найти все коммиты git, сделанные CI. Весь фрагмент кода доступен в [репозитории примеров](https://github.com/viesure/blog-gitflow-maven/blob/master/.gitlab-ci.yml#L8-L11).

### **Собираем все вместе**

Теперь, когда я могу получить доступ к git из CI, мне нужно настроить плагин gitflow. Я столкнулся с некоторыми особенностями.

Для простоты я использую предопределенные переменные, чтобы решить, какую цифру версии я хочу увеличить:

·    MINOR для релизов

·    PATCH для hotfixes

Можно также передать переменную при выполнении конвейера или получить ее другим способом.

Все цели (goals) передаются с параметром `-B`, который означает пакетный режим и означает, что пользовательский ввод не будет запрашиваться.

### **Автоматический Release**

```
$ ./mvnw gitflow: release -B -DgitflowDigitToIncrement = $RELEASE_DIGIT
```

Полный процесс выпуска относительно прост. Плагин объединяет ветку разработки в master без SNAPSHOT внутри версии, а затем увеличивает версию для разработки. Я просто вызываю цель (goal ) maven и сообщаю ей, какую цифру увеличивать.

### **Ручная разблокировка**

```
$ ./mvnw gitflow: release-start -B -DgitflowDigitToIncrement = $RELEASE_DIGIT
```

```
$ git push origin HEAD
```

Выпуск вручную начинается с создания ветки выпуска. Технически цифра не нужна, так как увеличение выполняется при слиянии, но если опция увеличения версии для разработки при разветвлении активна, она будет работать с ней из коробки (например, если она позже изменится, или другие проекты используйте разные настройки). Обратите внимание, что мне нужно нажать ветку вручную, поскольку плагин предназначен для локального использования.

```
$ git symbolic-ref refs/heads/$CI_COMMIT_REF_NAME refs/remotes/origin/$CI_COMMIT_REF_NAME
$ ./mvnw gitflow:release-finish -B -DgitflowDigitToIncrement=$RELEASE_DIGIT
```

Завершение release приносит с собой первый обходной путь. Git хранит ссылку (сокращенно ref) на HEAD всех ваших веток. Однако Gitlab CI не устанавливает все эти ссылки при проверке репозитория. Плагин использует эти ссылки на HEAD для проверки веток. Тем не менее, у него есть правильные локальные ссылки, поэтому я просто создаю недостающую ссылку HEAD.

После этого цель окончательного выпуска объединяет ветвь выпуска в master ветку и ветку разработки, увеличивая версию для разработки.

### **Исправление (Hotfix)**

Исправление следует тому же процессу, что и выпуск вручную, за исключением того, что оно запускается на мастере, а не на разработке.

 ```
$ ./mvnw gitflow:hotfix-start -B -DgitflowDigitToIncrement=$HOTFIX_DIGIT
$ git push origin HEAD
 ```

Hotfix-start создает ветку hotfix, которая уже содержит увеличенную версию.

```
$ export CURRENT_VERSION=${CI_COMMIT_REF_NAME/hotfix\/}
$ git symbolic-ref refs/heads/$CI_COMMIT_REF_NAME refs/remotes/origin/$CI_COMMIT_REF_NAME
$ ./mvnw gitflow:hotfix-finish -B -DgitflowDigitToIncrement=$HOTFIX_DIGIT -DhotfixVersion=$CURRENT_VERSION
```

Hotfix-finish объединяет его в master ветку и ветку разработки. Однако для исправления требуется еще один параметр: версия исправления. Причина этого опять же в том, что плагин разработан для локального использования, где у вас может быть несколько веток. Он не ожидает выполнения цели внутри ветки исправления.

Как сказано выше, hotfix-start уже увеличивает номер версии, чтобы обозначить версию исправления. Таким образом, имя ветки уже указывает версию исправления.

Я просто извлекаю версию из имени текущей ветки. Поскольку задание конвейера исправления определено для ветвей исправления, оно будет выполняться только в указанной ветке.

### **Резюме**

Это оно. Благодаря этому вы всего в одном клике от всех изменений версий и ветвлений, которые вам когда-либо понадобятся!

Потребовалось немного усилий, чтобы заставить взаимодействия git работать внутри бегунов CI и обойти отсутствующие ссылки. Но теперь, когда это работает, я очень доволен результатом.

Мне больше никогда не придется думать о порядке, в котором мне нужно разветвлять и увеличивать версии. CI позаботится об этом за меня. 

Вы можете найти полный сценарий [Gitlab CI](https://github.com/viesure/blog-gitflow-maven/blob/master/.gitlab-ci.yml) и пример проекта со всеми приведенными выше фрагментами кода в нашем репозитории [GitHub](https://github.com/viesure/blog-gitflow-maven).

## Ссылки

- [Viesure GitHub Repository](https://github.com/viesure/blog-gitflow-maven)
- [Semantic Versioning](https://semver.org/)
- [gitflow-maven-plugin](https://aleksandr-m.github.io/gitflow-maven-plugin)
- [gitflow-maven-plugin Documentation](https://github.com/aleksandr-m/gitflow-maven-plugin)
- [Gitlab Access Tokens Documentation](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)

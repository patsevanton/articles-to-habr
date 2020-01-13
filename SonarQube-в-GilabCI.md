SonarQube - это открытая платформа для обеспечения непрерывного контроля качества исходного кода, поддерживающая большое количество языков программирования и позволяющая получать отчеты по таким метрикам, как дублирование кода, соответствие стандартам кодирования, покрытие тестами, сложность кода, потенциальные ошибки и т.д. SonarQube удобно визуализирует результаты анализа и позволяет отслеживать динамику развития проекта во времени.


### SonarQube

Для установки sonarqube из rpm пакетов воспользуемся репозиторием https://harbottle.gitlab.io/harbottle-main.

Установим пакет с репозиторием для CentOS 7.

```bash
yum install -y https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
```

Устанавливаем сам sonarqube.

```bash
yum install -y sonarqube
```

При установке установятся большинство плагинов, но нужно доустановить findbugs и pmd

```
yum install -y sonarqube-findbugs sonarqube-pmd
```

Запустим и включим в автомзагрузку

```bash
systemctl start sonarqube
systemctl enable sonarqube
```

Если долго загружается, то добавьте в конец опций sonar.web.javaOpts генератор случайных чисел /dev/./urandom

```bash
sonar.web.javaOpts=другие параметры -Djava.security.egd=file:/dev/./urandom
```

Для тестирования будем использовать репозиторий https://github.com/uweplonus/spotbugs-examples.

Импортируем в Gitlab. Добавляем файл .gitlab-ci.yml:

```yaml
variables:
  MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=~/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
  MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  SONAR_HOST_URL: "http://172.26.9.226:9000"
  LOGIN: "admin" # логин sonarqube
  PASSWORD: "admin" # пароль sonarqube

cache:
  paths:
    - .m2/repository

build:
  image: maven:3.3.9-jdk-8
  stage: build
  script:
    - wget http://ftp.ru.debian.org/debian/pool/main/j/jq/jq_1.4-2.1+deb8u1_amd64.deb
    - dpkg -i jq_1.4-2.1+deb8u1_amd64.deb && rm -f jq_1.4-2.1+deb8u1_amd64.deb || true
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.failure.ignore=true org.jacoco:jacoco-maven-plugin:0.8.5:prepare-agent clean verify org.jacoco:jacoco-maven-plugin:0.8.5:report
    - mvn $MAVEN_CLI_OPTS -Dmaven.test.skip=true verify sonar:sonar -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$LOGIN -Dsonar.password=$PASSWORD -Dsonar.gitlab.project_id=$CI_PROJECT_PATH -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
    - export URL=$(cat target/sonar/report-task.txt | grep ceTaskUrl | cut -c11- ) #URL where report gets stored
    - echo $URL
    - |
      while : ;do
          curl -k -u "$LOGIN":"$PASSWORD" "$URL" -o analysis.txt
          export status=$(cat analysis.txt | jq -r '.task.status') #Status as SUCCESS, CANCELED, IN_PROGRESS or FAILED
          echo $status
          if [ ${status} == "SUCCESS" ];then
            echo "SONAR ANALYSIS SUCCESS";
            break
          fi
          sleep 5
      done
    - curl -k -u "$LOGIN":"$PASSWORD" "$URL" -o analysis.txt
    - export status=$(cat analysis.txt | jq -r '.task.status') #Status as SUCCESS, CANCELED or FAILED
    - export analysisId=$(cat analysis.txt | jq -r '.task.analysisId') #Get the analysis Id
    - |
      if [ "$status" == "SUCCESS" ]; then
        echo -e "SONAR ANALYSIS SUCCESSFUL...ANALYSING RESULTS";
        curl -k -u "$LOGIN":"$PASSWORD" "$SONAR_HOST_URL/api/qualitygates/project_status?analysisId=$analysisId" -o result.txt; #Analysis result like critical, major and minor issues
        export result=$(cat result.txt | jq -r '.projectStatus.status');

        if [ "$result" == "ERROR" ];then
          echo -e "91mSONAR RESULTS FAILED";
          echo "$(cat result.txt | jq -r '.projectStatus.conditions')"; #prints the critical, major and minor violations
          exit 1 #breaks the build for violations
        else
          echo -e "SONAR RESULTS SUCCESSFUL";
          echo "$(cat result.txt | jq -r '.projectStatus.conditions')";
          exit 0
        fi
      else
          echo -e "\e[91mSONAR ANALYSIS FAILED\e[0m";
          exit 1 #breaks the build for failure in Step2
      fi
  tags:
    - docker
```

Файл .gitlab-ci.yml неидеальный. Тестировался если задачи проверки в sonarqube завершались статусом: "SUCCESS". Пока что другие статусов не было. Как будут другие статусы, поправлю .gitlab-ci.yml  в этом посте.


кто может поставьте + этой теме

https://community.sonarsource.com/t/badges-on-private-projects/4894/13

SonarQube не может отобразить Badges для  приватных проектов.

например поместить их в  readme.md

бага https://jira.sonarsource.com/browse/MMF-1178

Автоматизация релизов с помощью github-action и semantic-release. А так же использование Pre-commit в Github action.

#### Семантически управлением версиями с помощью semantic-release и github-action

В этом посте будет описано практическое применение semantic-release для terraform модуля [terraform-yandex-compute](https://github.com/patsevanton/terraform-yandex-compute) (Модуль Terraform, который создает вычислительные ресурсы в облаке Яндекса) c Github action.

Semantic-release автоматизирует весь рабочий процесс выпуска пакета, включая: определение номера следующей версии, создание release notes (CHANGELOG.md) и публикацию пакета.

Это устраняет непосредственную связь между человеческими эмоциями и номерами версий, строго следуя спецификации семантического управления версиями и сообщая потребителям о влиянии изменений.

Библиотека следует нескольким концепциям.

1. Есть одна главная ветка в репозитории (master). Все остальные — feature ветки
2. Новый Pull Request — новый релиз
3. Начальная версия продукта — 1.0.0

Для semantic-release необходимо создать .releaserc.json. Вы можете взять [.releaserc.json](https://github.com/patsevanton/terraform-yandex-compute/blob/main/.releaserc.json) и ничего там не менять. Все заработает.

По умолчанию Semantic Release следует правилам коммитирования по гайдлайну [Angular Commit Messages](https://github.com/angular/angular/blob/master/CONTRIBUTING.md#-commit-message-format). В репозитории  [terraform-yandex-compute](https://github.com/patsevanton/terraform-yandex-compute) использую правила [ESLint](https://github.com/patsevanton/terraform-yandex-compute/blob/main/.releaserc.json#L11)

Когда очередной пул-реквест попадает в master ветку, semantic release производит скан коммитов от текущего состояния до последнего релизного тега. В [releaseRules](https://github.com/patsevanton/terraform-yandex-compute/blob/main/.releaserc.json#L12) описаны правила смены версии. 

```
        "releaseRules": [
          {"tag": "breaking", "release": "major"},
          {"tag": "chore", "release": false},
          {"tag": "ci", "release": false},
          {"tag": "docs", "release": false},
          {"tag": "feat", "release": "minor"},
          {"tag": "fix", "release": "patch"},
          {"tag": "refactor", "release": "patch"},
          {"tag": "security", "release": "patch"},
          {"tag": "style", "release": "patch"},
          {"tag": "test", "release": false}
        ]
```

После успешного релиза (команда определяется конфигом) в Git создается новый тег, в описании которого добавляется отформатированная информация из полученных коммитов.

Вот пример конечного [Release Notes (CHANGELOG.md)](https://github.com/patsevanton/terraform-yandex-compute/blob/main/CHANGELOG.md)

Для проверки что вы правильно делаете заголовок PR можно использовать проверочный [pr-title.yml](https://github.com/patsevanton/terraform-yandex-compute/blob/main/.github/workflows/pr-title.yml)



У себя вы можете использовать плагины для [Maven](https://github.com/conveyal/maven-semantic-release) и [Gradle](https://github.com/tschulte/gradle-semantic-release-plugin).

Более подробно про semantic-release можно прочитать в статьях:

- [Как мы автоматизировали процесс генерации Release Notes](https://habr.com/ru/company/ru_mts/blog/572774/)

- [Как релизить библиотеку с открытым кодом в 2020 году](https://habr.com/ru/company/jugru/blog/527436/)

  

#### Автоматизируем проверку кода с помощью Pre-commit в Github action.

При code review ты указываешь разработчикам на вещи, которые можно было отловить автоматически? С ужасом видишь код с синтаксической ошибкой? Хватит это терпеть! У нас же есть Pre-commit!

Итак, для начала нам надо разобраться с тем, что такое pre-commit hooks. В системе контроля версий Git есть специальный механизм для запуска скриптов и/или команд по определенному событию (см. рисунок ниже), благодаря которому мы можем автоматизировать некоторые рутинные операции.

Так как из feature ветки в main/master это обычный коммит, то можно применить git-hook [pre-commit](https://pre-commit.com/)

Допустим, что перед каждым commit мы должны выполнять следующие шаги:

- Переформатирование кода, согласно правилам форматирования кода.
- Удаление неиспользуемых импортов библиотек и неиспользуемых переменных.
- Апгрейд кода под новый стиль написания, который добавлен в новых версиях языка.
- Переформатирование импортов согласно правил форматирования.
- Проверить все переформатирования на соответствие стандартам форматирования.
- Запуск тестов.

Для запуска pre-commit в PR необходимо создать .pre-commit-config.yaml в корне проекта и запускать pre-commit run --all-files в этом PR. Для этого можно использовать [GitHub action to run pre-commit](https://github.com/pre-commit/action) или взять в качестве примера [старндартный pre-commit в репозитории openai/gym](https://github.com/openai/gym/blob/master/.github/workflows/pre-commit.yml)

Более подробно про pre-commit:

- [Что такое pre-commit hooks для Git и зачем они нужны](https://dou.ua/forums/topic/32915/)

- [Автоматизируем проверку кода или еще немного о pre-commit hook'ах](https://habr.com/ru/post/302932/)

- [Интеграция .pre-commit hook в Django проект](https://habr.com/ru/post/503270/)

Примеры на основе репозитория https://github.com/patsevanton/terraform-yandex-compute

В начале коммита должны быть следующие типы c двоеточием:

            breaking
            chore
            ci
            docs
            feat
            fix
            refactor
            security
            style
            test

Эти типы регулируются здесь в  [releaseRules](https://github.com/patsevanton/terraform-yandex-compute/blob/main/.releaserc.json#L12) в .releaserc.json. В заголовке коммиты должны присутствовать эти типы. Проверяются с помощью github action:

```
name: 'Validate PR title'

on:
  pull_request_target:
    types:
      - opened
      - edited
      - synchronize

jobs:
  main:
    name: Validate PR title
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v3.4.6
        env:
          GITHUB_TOKEN: ${{ secrets.TERRAFORM_YANDEX_COMPUTE_TOKEN }}
        with:
          # Configure which types are allowed.
          # Default: https://github.com/commitizen/conventional-commit-types
          types: |
            breaking
            chore
            ci
            docs
            feat
            fix
            refactor
            security
            style
            test
          requireScope: false
          subjectPattern: ^(?![A-Z]).+$
          subjectPatternError: |
            The subject "{subject}" found in the pull request title "{title}"
            didn't match the configured pattern. Please ensure that the subject
            doesn't start with an uppercase character.
          wip: true
          validateSingleCommit: false
```



Нужно иметь ввиду что в этом примере релиз создается при изменении следующих файлов: 

```
      - '**/*.tpl'
      - '**/*.py'
      - '**/*.tf'
      - '.github/workflows/release.yml'
```



Это описывается с помощью следующего github action:

```
name: Release

on:
  workflow_dispatch:
  push:
    branches:
      - main
      - master
    paths:
      - '**/*.tpl'
      - '**/*.py'
      - '**/*.tf'
      - '.github/workflows/release.yml'

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    # Skip running release workflow on forks
    if: github.repository_owner == 'patsevanton'
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          persist-credentials: false
          fetch-depth: 0

      - name: Release
        uses: cycjimmy/semantic-release-action@v2
        with:
          semantic_version: 18.0.0
          extra_plugins: |
            @semantic-release/changelog@6.0.0
            @semantic-release/git@10.0.0
            conventional-changelog-conventionalcommits@4.6.3
            conventional-changelog-eslint@3.0.9
        env:
          GITHUB_TOKEN: ${{ secrets.SEMANTIC_RELEASE_TOKEN }}
```

Разберем таблицу releaseRules, описанную выше.

    breaking создается релиз major версии
    chore не создается релиз 
    ci не создается релиз 
    docs не создается релиз 
    feat создается релиз minor версии
    fix создается релиз patch версии
    refactor создается релиз patch версии
    security создается релиз patch версии
    style создается релиз patch версии
    test не создается релиз 

Так как в репозитории слишком специфичная проверка для terraform, то можно сказать что проверку кода на оформление, синтаксис, стиль в общем случае можно проверять с помощью такого github action:

```
# https://pre-commit.com
# This GitHub Action assumes that the repo contains a valid .pre-commit-config.yaml file.
name: pre-commit
on:
  pull_request:
  push:
    branches: [master]
jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
      - run: pip install pre-commit
      - run: pre-commit --version
      - run: pre-commit install
      - run: pre-commit run --all-files
```

Если в коммите код оформлен не по принятому стилю или есть unit-test падают, то зафейлится github action

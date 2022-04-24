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


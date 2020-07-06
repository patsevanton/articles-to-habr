Git hooks – инструмент, помогающий держать в порядке ваш репозиторий. Можно настроить автоматические правила оформления ваших коммитов. 

Все вы наверное знаете про [pre-commit](https://pre-commit.com/) - проверку вашего кода перед коммитом. Но ведь не все можно проверить перед коммитом. Некоторые ограничения хочется использоваться глобально на всем Gitlab. 

Кто запутался в pre-commit и pre-receive хуках, в этом посте описываются различия между ними https://blog.gitguardian.com/git-hooks-automated-secrets-detection/ в абзаце "What are git hooks?".

Если у вас Gitlab Enterprise Edition, вы можете настроить хуки, которые описаны в посте через WEB интерфейс.

Но что делать, если у вас Gitlab Community Edition?

В этой статье будут описаны 5 pre-receive хуков, которые выполняются на сервере Gitlab Community Edition:

- block_confidentials.sh - Блокирование отправки приватных ключей и AWS токенов
- block_file_extensions.sh - Блокирование отправки архивов (Regex настраивается)
- check-large-files.sh - Блокирование отправки больших файлов (Размер настраивается)
- reject-not-allowlist-email.sh - Блокирование коммитов с email не из allow списка (Список email доменов настраивается)
- require-issue.sh - Блокирование коммитов без issue в названии (Список issue настраивается)

<cut />

В основном хуки взяты из репозитория [platform-samples](https://github.com/github/platform-samples) в директории **pre-receive-hooks** (относится к GitHub Enterprise).

Весь исходный код серверных хуков вы можете посмотреть на Github - https://github.com/patsevanton/git-server-pre-receive-hooks

### Установка на Gitlab

- Необходимо создать директорию `/opt/gitlab/embedded/service/gitlab-shell/hooks/pre-receive.d/`
- Скопировать в эту директорию хуки
- Не забыть выставить права запуска для хуков (chmod +x файл-хука)

### Блокирование отправки приватных ключей и AWS токенов

В файле block_confidentials.sh настраиваем список regex_list, который описывает конфиденциальную информацию.

```
# Define list of REGEX to be searched and blocked
regex_list=(
  # block any private key file
  '(\-){5}BEGIN\s?(RSA|OPENSSH|DSA|EC|PGP)?\s?PRIVATE KEY\s?(BLOCK)?(\-){5}.*'
  # block AWS API Keys
  'AKIA[0-9A-Z]{16}'
  # block AWS Secret Access Key (TODO: adjust to not find validd Git SHA1s; false positives)
  # '([^A-Za-z0-9/+=])?([A-Za-z0-9/+=]{40})([^A-Za-z0-9/+=])?'
  # block confidential content
  'CONFIDENTIAL'
)
```

Добавляем в репозиторий приватный ключ, делаем коммит и при `git push` получаем ошибку.

![](https://habrastorage.org/webt/jq/fw/kx/jqfwkxqsg4ugu94oads9zjvm3tk.png)

###  Блокирование отправки архивов

В файле block_file_extensions.sh настраиваем case `*.zip|*.gz|*.tgz`, в котором указываются расширения файлов, которые будут блокироваться.

Добавляем в репозиторий zip архив, делаем коммит и при `git push` получаем ошибку.

![](https://habrastorage.org/webt/2s/iy/z3/2siyz35_wtazzauh8asoj5zicji.png)

### Блокирование отправки больших файлов

В файле check-large-files.sh настраиваем параметр `maxsize`, который указывает размер файла в мегабайтах, выше которого отправка будет блокироваться.

Добавляем в репозиторий файл больше 1 мегабайта, делаем коммит  и при `git push` получаем ошибку.

![](https://habrastorage.org/webt/1n/c3/sj/1nc3sjqy3yfgzjfidsuboksqju4.png)

### Блокирование коммитов с email не из allow списка

В файле reject-not-allowlist-email.sh настраиваем список email-доменов, для которых разрешены коммиты.

```
declare -a DOMAIN_ARRAY=("group1.com" "group2.com")
```

Меняем почту в git на ту, которой нет в разрешенном списке.

```
git config user.email user1@group3.com
```

Добавляем в репозиторий любой файл, делаем коммит и при `git push` получаем ошибку.

![](https://habrastorage.org/webt/lw/8w/jj/lw8wjjazuc9qonuw_g4yo4oshq8.png)

### Блокирование коммитов без issue в названии

Этот серверный хук был взят из блога [Majilesh](https://www.majilesh.com/author/majilesh/).

В файле require-issue.sh настраиваем список commit_format, для которых разрешены коммиты.

```
commit_format="(JIRA|PROJECTKEY|MULE|ECOM|SAP|XLR-[1-9]+Merge)"
```

Добавляем в репозиторий любой файл, делаем коммит, в названии которого нет слов из commit_format и при `git push` получаем ошибку.

![](https://habrastorage.org/webt/z2/fk/c1/z2fkc1hgh6359ujbld7r6ph_exy.png)

Надеюсь что мой пост сподвигнет сообщество развивать направление серверных хуков. 

Telegram чат по Gitlab https://t.me/ru_gitlab


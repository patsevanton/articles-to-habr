Git hooks – инструмент, помогающий держать в порядке ваш репозиторий. Можно настроить автоматические правила оформления ваших коммитов. 

Все вы наверное знаете про [pre-commit](https://pre-commit.com/) - проверку вашего кода перед коммитом. Но ведь не все можно проверить перед коммитом. Некоторые ограничения хочется использоваться глобально на всем Gitlab. В этой статье будут описаны 5 pre-receive хуков, которые выполняются на сервере Gitlab:

- block_confidentials.sh - Блокирование отправки приватных ключей и AWS токенов
- block_file_extensions.sh - Блокирование отправки архивов (Regex настраивается)
- check-large-files.sh - Блокирование отправки больших файлов (Размер настраивается)
- reject-not-allowlist-email.sh - Блокирование коммитов с email не из allow списка (Список email доменов настраивается)
- require-issue.sh - Блокирование коммитов без issue в название (Список issue настраивается)

хабракат

Весь исходный код серверных хуков вы можете посмотреть на Github - https://github.com/patsevanton/git-server-pre-receive-hooks

### Установка на Gitlab

- Необходимо создать директорию `/opt/gitlab/embedded/service/gitlab-shell/hooks/pre-receive.d/`
- Скопировать в эту директорию хуки
- Не забыть выставить права запуска для хуков (chmod +x файл-хука)

### Блокирование отправки приватных ключей и AWS токенов

Добавляем в репозиторий приватный ключ и при `git push` получаем ошибку `[POLICY BLOCKED] You're trying to commit a private key or confidential information`



###  Блокирование отправки архивов (Regex настраивается)

Добавляем в репозиторий zip архив и при `git push` получаем ошибку `We have restricted committing that filetype. Do not store archive in git.`


Этот пост преследует следующие цели:

- Популяризация Katacoda
- Показать что можно переводить на русский язык сценарии Katacoda

Katacoda - это интерактивная платформа для обучения разработчиков программного обеспечения. Платформа предоставляет среды, которые уникально доступны через браузер, без необходимости настройки или загрузки. С большим количеством бесплатных интерактивных сценариев (более 300).

Примеры популярных обучающих сценариев: Docker, Kubernetes, Opensift, Machine Learning, Linux, CI/CD, Networking, Cloud Native Storage, Cloud Native Security, Serverless, Cloud Platforms, Infrastructure Automation and Configuration, TensorFlow, Traefik и другие.

В первую очередь вам необходимо залогиниться в https://katacoda.com/. При этом автоматически создастся ваш профайл, в котором вы можете создавать нужные вам сценарии.

Следующим шагом необходимо синхронизировать Github репозиторий и Katacoda через WebHooks

#### Шаг 1 - Создайте Github репозиторий

Создайте новый общедоступный репозиторий Github для своих сценариев Katacoda. Пример активного репозитория https://github.com/patsevanton/katacoda-scenarios

![](https://habrastorage.org/webt/ft/dl/le/ftdllewtp8wcsy6-9g8vvoukvbm.png)

В файл README.md можно добавить 

```
# Interactive Katacoda Scenarios

Посетите https://katacoda.com/youlogin, чтобы просмотреть и использовать интерактивные сценарии.
```

#### Шаг 2 - Обновите профайл Katacoda

Сохраните URL-адрес вашего репозитория Github в своем профиле Katacoda.

![](https://habrastorage.org/webt/ws/t-/5k/wst-5kakjbac5t5i1x1f90zk8l8.jpeg)

#### Шаг 3 - Найдите секретный токен Webhook

Скопируйте секрет токен Webhook (Git Webhook Secret) и сохраните профиль.

Секрет Katacoda используется, чтобы гарантировать, что только проверенные репозитории Github могут обновлять ваши сценарии. Это можно найти на странице настроек [вашего профиля](https://katacoda.com/profile/settings).

#### Шаг 4 - Создайте Github Webhook

В настройках репозитория Github добавьте новый Github Webhook

![](https://habrastorage.org/webt/xf/mc/fl/xfmcflki1t43_w-dkct9fkhsw8g.png)

Используйте URL-адрес https://editor.katacoda.com/scenarios/updated и свой скопированный секрет (Git Webhook Secret). В Content type необходимо выставить application/json.

![](https://habrastorage.org/webt/ps/cq/mh/pscqmhckumwdvu_tusy5zljl9q8.png)

#### Шаг 5 - Начните писать интерактивные сценарии

Осталось только начать писать ваши интерактивные сценарии! Пример сценария можно найти на https://github.com/katacoda/scenario-example

Файл index.json содержит свойство ImageID. Это относится к базовой среде. Для Docker используйте докер ImageID. Для Kubernetes используйте kubernetes.

Каждый сценарий должен появиться в собственном каталоге. Имя каталога станет URL-адресом сценария.

#### Шаг 6 - Push

После того, как вы отправите свои изменения в репозиторий Github, сценарии и обновления появятся в вашем [профиле](https://katacoda.com/profile) Katacoda.


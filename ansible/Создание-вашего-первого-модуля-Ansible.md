https://techforce1.nl/creating-your-first-ansible-module

### Создание вашего первого модуля Ansible

В этом блоге я расскажу, как создать свой первый модуль Ansible.

![](https://habrastorage.org/webt/j1/zk/u4/j1zku43odwzsxthgb6_eptarlvw.gif)

Конечно, есть документация, доступная и на Ansible.com, но разобраться в ней достаточно трудно. Запуск своего первого модуля на основе этого введения дался мне с большим трудом. Вот почему я создал это пошаговое руководство. Новые пользователи заслуживают лучшей отправной точки.

В этом блоге рассматриваются следующие темы:

 - Что такое модуль Ansible

 - Настройка нашей среды сборки

 - Серверное API

 - Разработка собственно модуля

### Что такое модуль Ansible

Если вы знакомы с Ansible, то, вероятно, знаете, что каждая задача, которую вы выполняете в Ansible, является модулем Ansible. Если нет, то теперь вы это знаете.

Например, посмотрите на следующую задачу Ansible:

```
- name: установить последнюю версию python-requests
  yum:
    name: python-requests
    state: latest
```

В данном случае используется модуль Ansible `yum` для установки определенного пакета.

По умолчанию у вас уже есть много доступных модулей. В большинстве случаев вы можете использовать стандартные для общих задач.

Однако иногда вам нужно сделать что-то действительно конкретное, и встроенного модуля недостаточно. В этом случае вы можете искать в Интернете настраиваемые модули, например, на веб-сайте Ansible Galaxy, или создать свой собственный модуль Ansible.

В этом блоге мы создадим модуль Ansible, который может взаимодействовать с сервером API для добавления или удаления пользователей. Это простой пример, и, вероятно, вам не нужен специальный модуль, поскольку для него есть модуль по умолчанию, но он помогает объяснить, как работает концепция модуля.

Прежде чем мы создадим наш собственный модуль Ansible, нам сначала потребуется настройка нашей среды разработки.

### Настройка нашей среды сборки

Я использую VSCode для разработки модуля Ansible. Если вы хотите использовать что-то другое, то вам, вероятно, нужно будет сделать некоторые шаги немного по-другому.

Сначала мы создаем репозиторий, который содержит структуру папок для модуля Ansible и сервер API, с которым можно общаться.

Этот серверное API очень простое. На нем мы можем создавать и удалять пользователей - вот и все. Для целей этого блога этого достаточно.

Итак, давайте загрузим этот репозиторий и откроем его с помощью VSCode.

```
git clone https://gitlab.com/techforce1/ansible-module.git -b blog-setup
```

Убедитесь, что вы используете ветку **blog-setup**!

По завершении вы должны увидеть 3 папки (`.devcontainer`, `ansible`, `api-server`). Сначала нам нужно запустить серверное API. Для этого откройте `cmd` или другое терминальное приложение, перейдите в папку `api-server` и выполните `docker build -t api-server` (не забудьте точку в конце).

Теперь мы можем его запустить. Запустите `docker run -it -d -p 5000: 5000 api-server`. Это запускает API-сервер. Если вы теперь перейдете по адресу http://localhost:5000, вы должны увидеть простую веб-страницу.

Единственное, что осталось сделать, прежде чем мы разработаем наш модуль Ansible, - это настроить VSCode. В скачанном репозитории есть папка .devcontainer. Это специальная папка для VSCode с конфигурацией в ней того, как открыть `devcontainer`.

С помощью этого `devcontainer` вы запускаете свой VSCode внутри `devcontainer`.

Преимущество этой настройки заключается в том, что вы можете настроить все виды инструментов, которые будут доступны внутри этого `devcontainer`. В этом случае я добавил Ansible в `devcontainer`. Таким образом, вам не нужно вручную устанавливать Ansible на локальный компьютер, просто используйте его напрямую из VSCode.

Чтобы открыть VSCode внутри контейнера разработчика, вам нужно щелкнуть значок в левом нижнем углу, а затем щелкнуть `Reopen in container`.

![](https://habrastorage.org/webt/hd/ev/zn/hdevzn_ycnscn1sb-xiyia8obuq.gif)

*Теперь вы готовы приступить к самой работе.*

### Сервер API

У нашего сервера API есть разные конечные точки API, с которыми можно общаться. Давайте откроем браузер* и перейдем по адресу http://localhost:5000/. Вы получите веб-страницу с таблицей, в которой перечислены пользователи с правами администратора. Теперь давайте попробуем открыть конечную точку API, http://localhost:5000/API/users.

Вероятно, вы получите сообщение «Несанкционированный доступ», потому что нам нужно сначала пройти аутентификацию, прежде чем мы сможем взаимодействовать с API и добавить пользователей. Мы делаем это, перейдя по адресу http://localhost:5000/API/get-token. Он запросит пароль и имя пользователя. Вы можете использовать `admin` и пароль `initial_password`.

Предоставляя учетные данные, вы получаете обратно токен, который затем можно использовать для будущих вызовов API вместо имени пользователя и пароля

*Обычно вы не переходите к конечной точке API через браузер, но используете какой-то инструмент для подключения к нему, например, другое приложение или какой-то сценарий автоматизации.*

Для доступа к API из командной строки мы можем использовать `curl`. Чтобы запросить токен с помощью curl, мы используем следующую команду:

```
$ curl -u admin:initial_password http:/172.17.0.1:5000/API/get-token
{
  "duration": 600, 
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6MSwiZXhwIjoxNjA1NzE0NjI5Ljk1NzU4ODd9.8yDkOzN0umO2hN_D84KLV4Q4OuWzQoNf8puXWku9F14"
}
```

*Я использую другой URL для подключения. Это потому, что я запускаю эту команду из терминала в VSCode. Поскольку это выполняется в контейнере докера, мы используем адрес шлюза контейнера, IP-адрес хоста докера, для сервера API. В остальной части этого блога мы используем этот адрес шлюза.*

Чтобы просмотреть всех пользователей, настроенных на сервере API, мы используем следующую команду `curl`:

```
$ curl -H 'Accept: application/json' -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6MSwiZXhwIjoxNjA1NzE0NjI5Ljk1NzU4ODd9.8yDkOzN0umO2hN_D84KLV4Q4OuWzQoNf8puXWku9F14" http:/172.17.0.1:5000/API/users
{
  "Users": [
    {
      "admin": true, 
      "created": "Wed, 18 Nov 2020 14:41:31 GMT", 
      "email": "admin@api.local", 
      "id": 1, 
      "username": "admin"
    }
  ]
}
```

Чтобы добавить пользователя, вы используете ту же конечную точку API, но вместо выполнения запроса `GET` напишите запрос `POST` со следующим телом:

```
{
  "Users": [
    {
      "username": "test",
      "email": "test@api.local", 
      "password": "password",
      "admin": true
    }
  ]
}
```

### Разработка модуля Ansible

Давайте сначала создадим простую книгу. Откройте файл `ansible/tasks/main.yml`. И вставьте следующий код:

```
name: Add test user to API
our_api_module:
  name: test1
  state: present
  email: test1@test.local
  admin: False
```

Здесь имя модуля - `our_api_module`, у него 4 параметра. Теперь нам нужно его создать. Я уже подготовил файл модуля с основным содержимым, необходимым для работы с Ansible.

Пользовательские модули хранятся в папке библиотеки. Вы видите, что название точно такое же, как в нашем playbook. Таким образом, Ansible знает, что ему нужно использовать этот настраиваемый модуль.

Позвольте мне быстро объяснить некоторые важные части кода. Мы начинаем со строки 44, это та часть, которая вызывается, когда Ansible загружает модуль. С помощью параметра `arguments_spec` мы определяем, какие параметры вы можете настроить в своей playbook. Также мы установили режим проверки поддержки на false. Это означает, что этот модуль не поддерживает режим проверки Ansible.

```
def main():
    module = AnsibleModule(
        argument_spec=dict(
            state=dict(type='str', default='present',
                       choices=['absent', 'present']),
            name=dict(type='str', required=True),
            email=dict(type='str', required=True),
            admin=dict(type='bool', default=False),
            base_url=dict(requred=False, default=None),
            username=dict(requred=False, default=None),
            password=dict(requred=False, default=None, no_log=True),
        ),
        supports_check_mode=False,
    )
```

Затем в строке 59 мы инициализируем класс `ApiModule`, это загружает класс, определенный в строке 23. При инициализации этого класса он выполняет код функции __init__. Здесь мы определяем аргументы модуля и запрашиваем токен у сервера API. Посмотрите, как это работает, в функции `getToken` в строке 37.

Благодаря встроенному модулю `urls` Ansible для запроса токена требуется всего 3 строки кода.

```
def getToken(self):
    url = "{baseUrl}/API/get-token".format(baseUrl=self.baseUrl)
    response = open_url(url, method="GET", url_username=self.username, url_password=self.password, validate_certs=self.verifySsl)
    return json.loads(response.read())['token']
```

Хорошо. Модуль загружен, теперь пора написать код для создания пользователя. Мы собираемся добавить это в строку 69. Начнем с проверки того, что нам нужно сделать. Есть 2 варианта: либо добавить пользователя, либо удалить пользователя в зависимости от состояния параметра (*state*). Итак, давайте сделаем оператор `if`, добавив следующий код:

```
if api.state == 'absent':
    if api.user_exist(api.name):
       # do something to delete user
elif api.state == 'present':
    if not api.user_exist(api.name):
       # do something to add user
```

Этот код довольно прост. Но он вводит новую функцию под названием user_exist. Нам нужно добавить это в класс `ApiModule`:

```
def user_exist(self, name):
    url = "{baseUrl}/API/users".format(baseUrl=self.baseUrl)
    headers = {
        'Accept': "application/json",
        "Authorization": "Bearer {}" . format(self.token),
    }
    response = open_url(url, method="GET", headers=headers, validate_certs=self.verifySsl)
    results = json.loads(response.read())
    for user in results['Users']:
        if name == user['username']:
            return True
    return False
```

Как видите, мы используем заголовки, чтобы указать токен, который мы получили при инициализации модуля. После запроса всех пользователей из api endpoint */API/users* мы проверяем, есть ли пользователь в ответе. В этом случае мы возвращаем True, в противном случае мы возвращаем False.

Теперь мы знаем, существует пользователь или нет, давайте добавим функции для добавления или удаления пользователя.

Добавьте следующее:

```
def user_add(self):
        url = "{baseUrl}/API/users".format(baseUrl=self.baseUrl)
        headers = {
            'Accept': "application/json",
            'Content-Type': "application/json",
            "Authorization": "Bearer {}" . format(self.token),
        }
        data = {
            'username': self.name,
            'email': self.email,
            'admin': self.admin,
            'password': self.password
        }
        json_data = json.dumps(data, ensure_ascii=False)
        try:
            open_url(url, method="POST", headers=headers, data=json_data, validate_certs=self.verifySsl)
            return True
        except:
            return False
```

К этому времени вы сможете самостоятельно создать функцию удаления. При необходимости просто используйте функцию user_add в качестве примера. Обратите внимание: вам нужно изменить метод `HTTP` с `POST` на `DELETE`, и вам не нужно указывать тело, вместо этого вы передаете имя пользователя в качестве параметра URL-адреса.
Таким образом, URL-адрес будет примерно таким:

```
url = "{baseUrl}/API/users/{username}".format(baseUrl=self.baseUrl, username=self.name)
```

Не забудьте включить функцию удаления в оператор if.

Наконец, расширьте playbook (`tasks/main.yml`) следующим:

```
- name: Add test2 user to API
  our_api_module:
    name: test2
    state: present
    email: test2@test.local
    admin: False
    password: "test2test2"
 
- name: Delete test1 user to API
  our_api_module:
    name: test1
    state: absent
    email: test1@test.local
    admin: False
    password: "test3test3"
```

Запустите playbook, чтобы увидеть, что он создает 2 новых пользователя, а также удаляет 1 пользователя, которого вы только что создали.

### Итоги

Поздравляю! Теперь у вас есть функциональный модуль Ansible. Пробовали добавить функцию изменения пользователя? Также рассмотрите возможность добавления обработки ошибок в этот модуль, как если бы вы делали это в реальном модуле, чтобы заботиться об ошибках и правильно возвращать их в Ansible.

Не так уж и сложно, правда? Это не так. Простой код Python имеет большое значение.
Если вы хотите увидеть полный конечный результат, загляните в блог-результат ветки.

![](https://habrastorage.org/webt/j1/zk/u4/j1zku43odwzsxthgb6_eptarlvw.gif)

Проверка ролей Ansible через делегированный драйвер Molecule

https://medium.com/@fabio.marinetti81/validate-ansible-roles-through-molecule-delegated-driver-a2ea2ab395b5

![](https://habrastorage.org/webt/e9/sx/a0/e9sxa0wfcnay3jxflccoapcdv5o.jpeg)

Molecule - отличный инструмент для тестирования ролей Ansible, он выполняет надежный и гибкий процесс проверки для обеспечения хорошего уровня качества ролей. Почти вся документация Molecule сосредоточена на драйвере докера, где тесты выполняются с контейнерным сервером, но, несмотря на то, что это хороший выбор в большинстве случаев использования, могут быть случаи, в которых полезно переключиться на внешний облачный бэкэнд с использованием [делегированного драйвера](https://molecule.readthedocs.io/en/latest/configuration.html#delegated).

К сожалению, документация по делегированному драйверу в основном состоит всего из нескольких строк в официальном документе, тогда как более четкое объяснение и несколько примеров могут оказать огромную помощь тем разработчикам, которые хотят использовать Molecule таким образом.

Этот пост основан на моем опыте разработки простой роли Ansible от 0 до galaxy и фокусирует внимание на использовании делегированного драйвера, интегрированного с Google Cloud Platform. В качестве отправной точки я взял следующие полезные ссылки для своего проекта:

- Официальный документ Molecule (https://molecule.readthedocs.io/en/latest/#)
- Хорошее введение в Molecule от Джеффа Герлинга (https://www.jeffgeerling.com/blog/2018/testing-your-ansible-roles-molecule)
- Редкий пример делегированного драйвера из проблемы, открытой для сообщества Molecule (https://github.com/ansible-community/molecule/issues/1292)
- Полное руководство по использованию ansible вместе с Google Cloud Platform от Гэри А. Стаффорда (https://itnext.io/getting-started-with-red-hat-ansible-for-google-cloud-platform-fa666c42a00c)

**Делегированный драйвер: что говорит документация Molecule?**

Одна из причин, которая заставила меня написать это руководство, - это заявление в официальной документации Molecule:

*Разработчик должен придерживаться instance-config API. Пособие по созданию разработчика должно содержать следующие данные instance-config, а руководство по уничтожению разработчика должно сбрасывать instance-config*.

Возникает вопрос: что такое instance-config и какие данные должен предоставить разработчик?

Instance-config - это факт Ansible, хранящийся в файле YAML в кэше Molecule ( `$HOME/.cache/molecule/<role-name>/<scenario-name>/instance_config.yml`), который имеет следующую структуру:

```bash
- address: 10.10.15.17
 identity_file: /home/fabio/.ssh/id_rsa # mutually exclusive with
                                        # password
 instance: millennium_falcon
 port: 22
 user: hansolo
# password: ssh_password # mutually exclusive with identity_file
 become_method: sudo # optional
# become_pass: password_if_required # optional
```

Для тех, кому нужно иметь дело с узлами Windows, документация также предоставляет эквивалентную структуру для WinRM.

**Файл create.yml**

После того, как мы прояснили, что такое instance-config, мы можем переходить к следующему шагу. К счастью, Molecule также помогает нам сделать дополнительный шаг вперед, предоставляя файлы шаблонов сценариев с помощью команды `molecule init`, например:

 ```
molecule init scenario -driver-name=delegated
 ```

который создает следующую структуру каталогов:

 ```
.
├── INSTALL.rst
├── converge.yml
├── create.yml
├── destroy.yml
├── molecule.yml
└── verify.yml
 ```

- `molecule.yml` - это файл конфигурации Molecule, который определяет переменные, устанавливает фазовую последовательность и конфигурацию для каждой из них.
- `create.yml` - код Ansible для создания экземпляров на облачной платформе и хранения данных в instance-config.
- `destroy.yml` код Ansible для уничтожения экземпляров на облачной платформе и удаления их из instance-config
- `converge.yml` исполнение роли
- `verify.yml` набор проверочных тестов
- `INSTALL.rst` инструкции по установке необходимых зависимостей для запуска тестов Molecule

Теперь давайте сосредоточимся на файле `create.yml`, созданном Molecule:

 ```
---
- name: Create
 hosts: localhost
 connection: local
 gather_facts: false
 no_log: "{{ molecule_no_log }}"
 tasks:
 
 
 # Developer must implement.
 # Developer must map instance config.
 # Mandatory configuration for Molecule to function.
 
 
 — name: Populate instance config dict
 set_fact:
 instance_conf_dict: {
 'instance': "{{ }}",
 'address': "{{ }}",
 'user': "{{ }}",
 'port': "{{ }}",
 'identity_file': "{{ }}", }
 with_items: "{{ server.results }}"
 register: instance_config_dict
 when: server.changed | bool
 
 — name: Convert instance config dict to a list
 set_fact:
 instance_conf: {{ instance_config_dict.results | map(attribute='ansible_facts.instance_conf_dict') | list }}"
 when: server.changed | bool
 
 — name: Dump instance config
 copy:
 content: "{{ instance_conf | to_json | from_json | molecule_to_yaml | molecule_header }}"
 dest: "{{ molecule_instance_config }}"
 when: server.changed | bool
 ```

Три задачи: заполнение, преобразование и дамп, создание в конце файла `instance-config.yml`. Прокомментированный раздел является заполнителем для кода Ansible, который должен создавать облачные ресурсы и возвращать массив серверов (содержащий детали экземпляра) в качестве зарегистрированной переменной или факта. Следующий фрагмент кода, взятый из этой [проблемы с github](https://github.com/ansible-community/molecule/issues/1292), предоставляет пример того, что указано выше для контекста VMWare:

 ```
…
 7     - name: Create molecule instance(s)
 8      vmware_guest:
 9        hostname: "{{ molecule_yml.driver.hostname }}"
10        esxi_hostname: "{{ molecule_yml.driver.esxi_hostname }}"
11        username: "{{ molecule_yml.driver.username }}"
12        password: "{{ molecule_yml.driver.password }}"
13        datacenter: "{{ molecule_yml.driver.datacenter }}"
14        validate_certs: "{{ molecule_yml.driver.validate_certs }}"
15        resource_pool: "{{ molecule_yml.driver.resource_pool }}"
16         folder: "{{ molecule_yml.driver.folder }}"
17         name: "{{ item.name }}"
18         template: "{{ item.template }}"
19         hardware:
20           memory_mb: "{{ item.memory | default(omit) }}"
21           num_cpus: "{{ item.cpu | default(omit) }}"
22         wait_for_ip_address: "yes"
23         state: poweredon
24       register: server
25       with_items: "{{ molecule_yml.platforms }}"
26     
27     - name: Populate instance config dict
28       set_fact:
29         instance_conf_dict: {
30           'instance': "{{ item.instance.hw_name }}",
31           'address': "{{ item.instance.ipv4 }}",
32           'user': "vagrant",
33           'port': "22",
34           'identity_file': 'identity_file': "{{
                     molecule_yml.driver.ssh_identity_file }}"
35         }
36       with_items: "{{ server.results }}"
37       register: instance_config_dict
38       when: server is changed
…
 ```

Код вызывает модуль `vmware_guest` (строки 7–23) для создания виртуальной машины на сервере VMWare. Это делается для каждого элемента массива платформ, определенного в файле `molecule.yml` (строка 25). Как видите, переменные, определенные в файле `molecule.yml`, доступны через факт `molecule_yml`.

Значения, возвращаемые каждым вызовом `vmware_guest`, регистрируются как элементы массива сервера (строка 24), который, в свою очередь, используется для заполнения конфигурации экземпляра (`instance-config`) (строки 27 и далее). Обратите внимание, что обновление факта конфигурации экземпляра пропускается, если переменная сервера не изменяется.

**Работа с Google Cloud Platform (GCP)**

Теперь, когда я разъяснил, что и как должен делать разработчик при работе с делегированным драйвером, я собираюсь поделиться работой, проделанной для моей [роли docker-secured Ansible](https://galaxy.ansible.com/fabiomarinetti/fmarinetti_docker_secured). Для этой роли я решил использовать GCP в качестве облачного сервера для делегированного драйвера. Ansible предоставляет семейство модулей GCP для работы с таким облачным провайдером, и я надеюсь, что вы легко сможете адаптировать мой код, если вам нужно сменить семейство модулей и облачного провайдера.

Для этого проекта я использовал следующие версии инструментов:

- python 2.7
- ansible 2.9.6
- molecule 3.0.2
- ansible-lint 4.2.0
- yamllint 1.20.0
- flake8 3.7.9 (mccabe: 0.6.1, pycodestyle: 2.5.0, pyflakes: 2.1.1) CPython 2.7.17 на Linux

где yamllint, ansible-lint и flake8 - инструменты для проверки кода, которые включены в фазы молекулы.

**Роль docker-secured**

Роль для тестирования устанавливает докер на узел, предоставляет API-интерфейсы докеров и защищает их с помощью ssl. Процедура, которой я следовал, описана в этих двух ссылках из документации Docker:

- https://success.docker.com/article/how-do-i-enable-the-remote-api-for-dockerd

- https://docs.docker.com/engine/security/https/

Репозиторий содержит готовые к использованию файлы сертификатов ssl для целей тестирования, но есть также возможность предоставить свои собственные, если они у вас есть.

Вы можете посмотреть мой проект, клонировав репозиторий GitHub:

```bash
git clone https://github.com/fabiomarinetti/fmarinetti.docker-secured.git
```

**Предварительные шаги для GCP**

Прежде всего, мне нужно было создать проект GCP, учетную запись службы и загрузить связанный ключ. Эти шаги выходят за рамки этого руководства, и вы можете обратиться к [официальному документу GCP](https://cloud.google.com/docs) для получения общей процедуры. В качестве хорошей ссылки я также нашел эту [ссылку](https://itnext.io/getting-started-with-red-hat-ansible-for-google-cloud-platform-fa666c42a00c) полезной для всего, что связано с совместной работой Ansible и GCP.

Для этой роли я создал проект `ansible-272015` и службу учетной записи `service`, ее ключ хранится в файле `secret.json`.

**Файл molecule.yml**

В этом разделе я покажу и прокомментирую соответствующий раздел моего файла `molecule.yml`.

Проект, тип аутентификации и секретный ключ вставляются в файл `molecule.yml` в разделе `driver`. Кроме того, я также добавил в тот же раздел все другие параметры, которые остаются постоянными на этапах создания и уничтожения, например, регион и зона GCP, пользователь ssh и файл идентификатора, а также параметры сети, поскольку предполагается, что виртуальные машины находятся в той же сети, в которой создан ad-hoc на время теста. Ко всем этим значениям можно получить доступ из сценария через `molecule_yml` (например, `molecule_yml.driver.region` для доступа к региону).

```
20 driver:
21   name: delegated
22   gcp_service_account_key: ${GOOGLE_APPLICATION_CREDENTIALS}
23   gcp_project_id: ansible-272015
24   region: us-east1
25   zone: us-east1-c
26   ssh_user: ${SSH_USER}
27   ssh_pub_key_file: "${SSH_ID_FILE}.pub"
28   ssh_key_file: "${SSH_ID_FILE}"
29   network_name: ansible-network
30   subnet_name: ansible-subnet
31   firewall_name: ansible-firewall
32   ip_cidr_range: 172.16.0.0/28
```

Раздел платформы в файле `molecule.yml` содержит массив, содержащий параметры (имя, изображение, тип, размер…) для экземпляров, которые я хочу проверить. Мое тестовое покрытие включает CentOS 7, Ubuntu Xenial 16.04 и Ubuntu Bionic 18.04. Эти машины сгруппированы по типу ОС (например, CentOS или Ubuntu), чтобы использовать группы инвентаря при выполнении Ansible.

```
41 platforms:
42   - name: "ds-centos7-${TRAVIS_BUILD_ID}"
43     image_family: projects/centos-cloud/global/images/family
                     /centos-7
44     machine_type: n1-standard-1
45     size_gb: 200
46     groups:
47       - centos
48   - name: "ds-ubuntu-bionic-${TRAVIS_BUILD_ID}"
49     image_family: projects/ubuntu-os-cloud/global/images/family
                     /ubuntu-1804-lts
50     machine_type: n1-standard-1
51     size_gb: 200
52     groups:
53       - ubuntu
54   - name:  "ds-ubuntu-xenial-${TRAVIS_BUILD_ID}"
55     image_family: projects/ubuntu-os-cloud/global/images/family
                     /ubuntu-1604-lts
56     machine_type: n1-standard-1
57     size_gb: 200
58     groups:
59       - ubuntu
```

В других разделах файла `molecule.yml` определяется тестовая последовательность и конфигурации для каждой фазы, хотя они не заданы по умолчанию.


**Фаза создания и файл create.yml**

Как уже говорилось ранее, `create.yml` - это сценарий, который управляет этапом создания. Здесь я широко использовал модули семейства gcp для управления ресурсами облачного провайдера (GCP). Модули GCP нуждаются в каком-то фиксированном параметре, таком как идентификатор проекта, тип аутентификации и путь к секретному ключу, и, чтобы избежать повторения этих значений в коде при каждом вызове модуля, я установил их как `module_defaults` для всего семейства gcp.

```
 7   module_defaults:
 8     group/gcp:
 9       project: "{{ molecule_yml.driver.gcp_project_id }}"
10       auth_kind: serviceaccount
11       service_account_file: "{{ 
              molecule_yml.driver.gcp_service_account_key }}"
```

В отличие от того, что мы видели ранее в случае VMWare, создание экземпляра в GCP - это не просто вопрос использования одного модуля, а процесс, состоящий из нескольких шагов: создание загрузочного диска, назначение IP-адреса и создание самого экземпляра. Это означает, что для цикла по платформам мне нужно было поместить задачи создания в отдельный файл и включить его в цикл:

```
16 — name: create instances
17   include_tasks: tasks/create_instance.yml
18   loop: "{{ molecule_yml.platforms }}"
```

Файл `create_instance.yml` содержит задачи по резервированию IP-адреса, созданию загрузочного диска и созданию экземпляра. То, как я вызвал связанные модули, довольно стандартно и их можно менять, если вы хотите переключиться на другого облачного провайдера, поэтому я не буду их обсуждать дальше, а лишь хочу сказать несколько слов о том, как возвращать данные экземпляра для подачи задач заполнения instance-config.

```
7 - name: initialize instance facts
 8   set_fact:
 9     instance_created:
10       instances: []
11   when: instance_created is not defined
... create the instance and return instance variable ...
56 - name: update instance facts
57   set_fact:
58     instance_created:
59       changed: instance.changed | bool
60       instances: "{{ instance_created.instances + [ instance ]}}"
```

Затем после цикла платформы для заполнения isntance-config используется факт `instance_create`:

```
20     - name: Populate instance config dict
21       set_fact:
22         instance_conf_dict: {
23           'instance': "{{ item.name }}",
24           'address': "{{
               item.networkInterfaces[0].accessConfigs[0].natIP }}",
25           'user': "{{ molecule_yml.driver.ssh_user }}",
26           'port': "22",
27           'identity_file': "{{ molecule_yml.driver.ssh_key_file
               }}", }
28       with_items: "{{ instance_created.instances }}"
29       register: instance_config_dict
30       when: instance_created.changed
```

Здесь эта задача выполняется, только если один из экземпляров был изменен, как это произошло в случае VMWare, когда было указано предложение `servers is changed`

Наконец, я протестировал этап создания, введя команду:

```
molecule create --scenario-name=gcp
```

Убедившись, что результаты созданы правильно, я перешел к конвейеру и выполнил / протестировал фазы:

- **lint**, которая выполняет проверку кода
- **prepare**, которая подготавливает экземпляр для применения роли. В данном случае это просто обновление исходных кодов пакетов для группы ubuntu.
- **converge**, которая просто применяет роль
- **idempotence**, которая применяет роль во второй раз для обеспечения ее идемпотентности
- **verify**, которая подтверждает, что результаты применения роли соответствуют ожиданиям

```
molecule <phase> --scenario-name=gcp
```

В этом случае, учитывая простоту роли и ограниченные требования, мне не пришлось так сильно менять Молекулу, генерируемую при инициализации сценария.

На последнем шаге я написал `destroy.yml` для удаления созданных ресурсов из проекта (а также моего счета 😄). Код для уничтожения ресурсов следует той же философии, что и тот, который их создает. Очевидно, проверка проводилась путем выдачи:

```
molecule destroy --scenario-name=gcp
```

Как только все этапы были правильными и не выдали ошибок, я мог протестировать весь процесс с помощью команды:

```
molecule test --scenario-test=gcp
```

**Выводы**

В этом посте я объяснил, как использовать делегированный драйвер Molecule, и показал, как я реализовал его с помощью GCP. Этот же код легко адаптировать к другому облачному провайдеру: AWS, Azure, Digital Ocean… и я надеюсь, что вы наверняка получите выгоду от использования Molecule. Пожалуйста, оставьте отзыв.

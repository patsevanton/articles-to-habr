Тестирование Ansible с использованием Molecule с Ansible в качестве верификатора

![](https://habrastorage.org/webt/ut/eu/ai/uteuaip-uwrawuyyrc98peuhoqo.jpeg)

В этом руководстве мы будем изучать, как тестировать код инфраструктуры, написанный на Ansible, с использованием инфраструктуры тестирования, известной как Molecule. Внутри Molecule мы будем использовать Ansible в качестве верификатора, чего я пока нигде не мог найти. Давай сделаем это!

 

Содержание

1. Введение

2. Установка Molecule

3. Установка Molecule в Ansible

4. Написание тестов Ansible с помощью Ansible Verifier

5. Выводы

 

1. Введение

Ansible - это инструмент автоматизации ИТ с открытым исходным кодом, используемый в управлении конфигурацией, развертывании приложений, оркестровке инфраструктурных служб, облачном обеспечении и многом другом. Это простой в использовании инструмент, но при этом упрощает выполнение очень сложных повторяющихся задач автоматизации ИТ. Его можно использовать для многоуровневого развертывания ИТ-приложений.

Как и в любой другой сфере  ИТ, тестирование неизбежно. Непроверенную инфраструктуру можно легко списать на уже сломанную инфраструктуру. Тестируя код инфраструктуры, мы обеспечиваем разработку кода инфраструктуры производственного уровня, лишенного ошибок и ошибок, которые могут оказаться очень дорогостоящими, если не будут обнаружены до производства.

Molecule - это фреймворк, предназначенный для помощи в разработке и тестировании ролей в Ansible. По состоянию на 26 сентября Ansible объявила о принятии Molecule и Ansible-lint в качестве официальных проектов Red Hat Ansible. Это показывает уверенность сообщества Red Hat в этом инструменте и объем работы, который они прилагают, чтобы сделать его еще лучше и лучше.

Molecule позволяет тестировать роли в разных экземплярах, операционных системах и дистрибутивах, поставщиках виртуализации, тестовых средах и сценариях тестирования.

Molecule поддерживает TDD-подобную форму кода инфраструктуры тестирования. В этом руководстве мы рассмотрим жизненный цикл, которому, по моему мнению, должно следовать тестирование Molecule.

 

2. Установка Molecule

Предполагается, что читатель уже имеет некоторый опыт управления пакетами в системах UNIX.

Для работы Molecule необходимы следующие пакеты:

- Python 2.7 или Python 3.5 или выше (в этом руководстве мы будем использовать Python 3.7)

- Ansible 2.5 или выше (для этого урока мы будем использовать Ansible 2.9.6)

- Docker (последняя версия)

Pip - единственный официально поддерживаемый менеджер пакетов для установки Molecule. Если вы используете Python 2.7.9 (или выше) или Python 3.4 (или выше), то PIP по умолчанию устанавливается вместе с Python.

Чтобы установить Molecule с помощью pip:

 ```
$ pip3 install molecule
 ```



Дополнительные советы по установке см. В разделе «[Установка Molecule](https://molecule.readthedocs.io/en/latest/installation.html)». Чтобы проверить, правильно ли установлена Molecule, запустите ```$ molecule --version```.

При тестировании Ansible Playbooks важно понимать, что мы **не** проверяем, что Ansible работает, т.е. что Ansible выполнил свою работу, например, по созданию файла, пользователя или группы, а мы проверяем, что наше намерение, как выражено простым английским языком, соответствует декларативному языку Ansible, то есть то, что было создано, - это именно то, что мы хотели создать, и не было человеческих ошибок (например, типографских ошибок или упущений).

 

3. Установка Molecule в Ansible роль.

Есть два способа установки Molecule для тестирования ролей Ansible:

а. **Установка Molecule с новой ролью Ansible**

Molecules использует [Ansible Galaxy](https://galaxy.ansible.com/) для создания стандартного макета ролей Ansible. Чтобы создать новую роль с помощью Molecule:

 ```
$ molecule init role <the_role_name>
 ```

б. **Иницизилизация Initiating для уже существующей роли Ansible**

Molecule также можно использовать для тестирования уже существующих ролей, просто введите следующую команду в корневом каталоге, где находится роль, или внутри каталога ролей, убедившись, что имена ролей совпадают: 

 ```
$ molecule init scenario -r <the_already_existing_role_name>
 ```

Независимо от того, как вы инициализируете Molecule, новая папка Molecule добавляется в корневую папку проекта. В результате макет папки выглядит следующим образом:

 ```
.
├── README.md
├── files/                                                
├── handlers/                                              
├── meta/                                                            
├── tasks/                                                 
├── templates/                                              
├── tests/                                               
├── vars/                                              
└── molecule/
        └── default                        
                ├── molecule.yml                        
                ├── converge.yml                        
                ├── verify.yml                                               
                └── INSTALL.rst
 ```

Ниже мы обсудим содержимое папки Molecule и их использование:

**molecule.yml**

В файле molecule.yml мы указываем всю конфигурацию Molecule, необходимую для проверки ролей.

 ```
---
dependency:
  name: galaxy
  enabled: true # to disable, set to false
driver:
  name: docker
platforms:
  - name: instance
    image: docker.io/pycontribs/centos:7
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
 ```



**dependency**:

Это диспетчер зависимостей, который отвечает за разрешение всех зависимостей ролей. Ansible Galaxy - это зависимость по умолчанию, используемая Molecule. Другие менеджеры зависимостей - Shell и Gilt. По умолчанию для `dependency` установлено значение `true`, но ее можно отключить, установив для параметра `enabled` значение false.

 

**driver**:

Драйвер сообщает Molecule, откуда мы хотим получить наши тестовые экземпляры. Драйвер Molecule по умолчанию - Docker, но есть и другие варианты, такие как: AWS, Azure, Google Cloud, Vagrant, Hetzner Cloud и многие другие. См. Драйверы Molecule для получения дополнительной информации.

 

**platforms**:

Ключ платформы указывает, какой тип инстансов мы хотим запустить для тестирования наших ролей. Это должно соответствовать драйверу, например, в приведенном выше фрагменте, он говорит, какой тип образа докера мы хотим запустить.

 

**provisioner:** 

Provider - это инструмент, который запускает файл converge.yml для всех запущенных экземпляров (указанных в платформах). Единственный поддерживаемый провайдер - Ansible.

 

**verifier:** 

*verifier* - это инструмент, который проверяет наши роли. Этот верификатор запускает файл verify.yml, чтобы убедиться, что фактическое состояние нашего экземпляра (состояние схождения) соответствует желаемому состоянию (состояние проверки). Верификатор по умолчанию - Ansible, но есть и другие верификаторы, такие как: testinfra, goss и inspec. Раньше testinfra был верификатором по умолчанию, но из-за необходимости в унифицированном тестировании UX и во избежание необходимости изучать другой язык, Python, в случае testinfra сообщество решило, что Ansible станет верификатором по умолчанию, и я поддерживаю это решение. Смотрите git issue [здесь](https://github.com/ansible-community/molecule/issues/2013).

 

Дополнительные ключи, которые не генерируются по умолчанию, - это lint и сценарий. Эти ключи могут быть добавлены в файл Molevel.yml по желанию.

 

**lint:** 

Lint представляет, какой инструмент Molecule должен использовать, чтобы гарантировать обнаружение и пометку декларативных ошибок, ошибок, стилистических ошибок и подозрительных конструкций. Популярные линты - yamllint, ansible-lint, flake8 и др.

 

**scenario:** 

Сценарий описывает жизненный цикл теста Molecule. Сценарий тестирования можно настроить, поскольку шаги в последовательности можно менять местами или закомментировать, чтобы соответствовать любому необходимому сценарию. У каждой роли должен быть сценарий по умолчанию, который называется по умолчанию.

Если не указано иное, имя сценария - это обычно имя каталога, в котором расположены файлы Molecule. Ниже приведен сценарий по умолчанию, запускаемый при запуске соответствующей последовательности команд:

 ```
scenario:
  create_sequence:
    - dependency
    - create
    - prepare
  check_sequence:
    - dependency
    - cleanup
    - destroy
    - create
    - prepare
    - converge
    - check
    - destroy
  converge_sequence:
    - dependency
    - create
    - prepare
    - converge
  destroy_sequence:
    - dependency
    - cleanup
    - destroy
  test_sequence:
    - dependency
    - lint
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - cleanup
    - destroy
 ```



Из приведенного выше фрагмента мы можем сказать, что происходит, когда мы запускаем команду Molecule, например, `$ molecule create` будет запускать, затем create_sequence, а `$ molecule check` запустит check_sequence и так далее.

 

В общем, мы добавляем ключ сценария только тогда, когда мы хотим настроить наш сценарий, иначе это не нужно, так как это сценарий по умолчанию и, следовательно, неявный.

 

Converge.yml

Файл converge.yml, как следует из названия, используется для преобразования состояния экземпляров в реальное состояние, объявленное в реальных тестируемых ролях. Он запускает одиночную конвергентную игру на запущенных экземплярах. Этот файл запускается, когда мы запускаем команду `$ molecule converge`.

 

verify.yml

Файл verify.yml запускает игру, которая вызывает тестовые роли. Эти роли используются для проверки того, что состояние уже конвергированного экземпляра соответствует желаемому состоянию. Этот файл запускается, когда мы запускаем команду `$ molecule verify`.

 

INSTALL.rst

Этот файл содержит инструкции по дополнительным зависимостям, необходимым для успешного взаимодействия между Molecule и драйвером.

 

4. Написание тестов Ansible с помощью Ansible Verifier

В этом разделе мы будем практиковать то, что, на мой взгляд, должно быть рабочим процессом для тестирования ролей Ansible с использованием верификатора Ansible в Molecule.

Выполнение `$ molecule test` запускает всю test_sequence, но всегда уничтожает созданные экземпляры в конце, и это может занять много времени, учитывая, что мы должны воссоздавать экземпляры каждый раз, когда мы вносим изменения в наши фактические или тестовые роли.

Таким образом, рабочий процесс, который соответствует подходу BDD, следующий:

 ```
# given phase
$ molecule create

# when phase
$ molecule converge

# then phase
$ molecule verify
 ```



В приведенном выше фрагменте данная фаза не меняется часто, поэтому мы просто создаем экземпляр (ы) один раз. После этого мы выполняем итерацию между этапами когда и затем, пока все наши тесты не будут проверены и не будут содержать ошибок.

В этом руководстве наша цель - реализовать TDD при тестировании нашей инфраструктуры. Мы бы писали модульные тесты. Допустим, мы хотели реализовать роль под названием alpha-services, которая решала следующие задачи:

 

- Задача 1. Устанавливает Java-1.8 на хост-машину.

- Задача 2: Создает каталог по пути `/var/log/tomcat`, принадлежащий владельцу `tomcat`, группе `tomcat` и режиму `0755`

- Задача 3. Устанавливает, запускает и включает httpd

- Задача 4. Скопируйте файл шаблона из `template/tomcat/context.xml` в `/etc/tomcat/context.xml`


Сначала мы создаем роль, используя команду Molecule инициализации:

 ```
$ molecule init role alpha-services
 ```

Это создает макет папки, подобный показанному ранее. Далее мы создаем путь `alpha-services/molecule/default/roles/test_alpha-services`:

 ```
$ cd alpha-services
$ mkdir -p molecule/default/roles/test_alpha-services
 ```

Здесь будут содержаться наши тестовые роли. Внутри каталога `test_alpha-services` мы

создайте наши тестовые роли, используя стандартный макет ролей Ansible (мы создаем только те папки, которые нам нужны для тестирования, в данном случае значения по умолчанию, задачи и вары). Каждая созданная папка должна иметь свой файл main.yml. Для отдельной задачи мы создадим отдельные файлы yml, чтобы различать их друг от друга, добавив к имени задачи префикс `test_`. Например, задача по установке java будет называться `test_java.yml`.

 ```
$ cd molecule/default/roles/test_alpha-services
$ mkdir defaults && touch defaults/main.yml
$ mkdir tasks && touch tasks/main.yml tasks/test_java.yml tasks/test_tomcat.yml tasks/test_httpd.yml tasks/test_aws.yml
$ mkdir vars && touch vars/main.yml
 ```

Тогда у нас останется следующий макет папок: 

 ```
alpha-services/
        ├── README.md
        ├── files/                                                
        ├── handlers/                                              
        ├── meta/                                                            
        ├── tasks/                                                 
        ├── templates/                                              
        ├── tests/                                               
        ├── vars/                                              
        └── molecule/
                └── default                        
                        ├── molecule.yml                        
                        ├── converge.yml                        
                        ├── verify.yml                                               
                        ├── INSTALL.rst                       
                        └── roles/    
                              └── test_alpha-services/                        
                                        ├── defaults/                        
                                              └── main.yml                                               
                                        ├── tasks/                        
                                              ├── main.yml 
                                              ├── test_java.yml                        
                                              ├── test_tomcat.yml                        
                                              ├── test_httpd.yml                                               
                                              └── test_aws.yml      
                                        └── vars/                        
                                              └── main.yml
 ```

Настраиваем файл molecule.yml:

 ```
---
dependency:
  name: galaxy
  enabled: false
driver:
  name: docker
platforms:
  - name: instance
    image: docker.io/pycontribs/centos:7
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
 ```

Мы оставляем файл converge.yml как есть:

 ```
---
- name: Converge
  hosts: all
  tasks:
    - name: "Include alpha-services"
      include_role:
        name: "alpha-services"
 ```

Мы редактируем файл verify.yaml, чтобы включить в него вашу роль поставщика тестов:

 ```
---
# This is an example playbook to execute Ansible tests.
- name: Verify
  hosts: all
  tasks:
    - name: "Include test_alpha-services"
      include_role:
        name: "test_alpha-services"
 ```



GIVEN ФАЗА: Мы запускаем `$ molecule create` для создания экземпляров.

 ```
$ molecule create

--> Test matrix
    
└── default
    ├── dependency
    ├── create
    └── prepare
    
--> Scenario: 'default'
--> Action: 'dependency'
Skipping, dependency is disabled.
--> Scenario: 'default'
--> Action: 'create'
--> Sanity checks: 'docker'
    
    PLAY [Create] ******************************************************************
    
    TASK [Log into a Docker registry] **********************************************
    skipping: [localhost] => (item=None) 
    
    TASK [Check presence of custom Dockerfiles] ************************************
    ok: [localhost] => (item=None)
    ok: [localhost]
    
    TASK [Create Dockerfiles from image names] *************************************
    skipping: [localhost] => (item=None) 
    
    TASK [Discover local Docker images] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]
    
    TASK [Build an Ansible compatible image (new)] *********************************
    skipping: [localhost] => (item=molecule_local/docker.io/pycontribs/centos:7) 
    
    TASK [Create docker network(s)] ************************************************
    
    TASK [Determine the CMD directives] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]
    
    TASK [Create molecule instance(s)] *********************************************
    changed: [localhost] => (item=instance)
    
    TASK [Wait for instance(s) creation to complete] *******************************
    FAILED - RETRYING: Wait for instance(s) creation to complete (300 retries left).
    changed: [localhost] => (item=None)
    changed: [localhost]
    
    PLAY RECAP *********************************************************************
    localhost                  : ok=5    changed=2    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
    
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
 ```

WHEN ФАЗА: Мы запускаем `$ molecule converge` для выполнения фактических ролей, которые еще предстоит реализовать. Это не повлияет на созданный экземпляр.

 ```
$ molecule converge

--> Test matrix
    
└── default
    ├── dependency
    ├── create
    ├── prepare
    └── converge
    
--> Scenario: 'default'
--> Action: 'dependency'
Skipping, dependency is disabled.
--> Scenario: 'default'
--> Action: 'create'
Skipping, instances already created.
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
--> Scenario: 'default'
--> Action: 'converge'
--> Sanity checks: 'docker'
    
    PLAY [Converge] ****************************************************************
    
    TASK [Gathering Facts] *********************************************************
    ok: [instance]
    
    TASK [Include alpha-services] **************************************************
    
    TASK [alpha-services : include java installation tasks] ************************
    included: /Users/chukwudiuzoma/Documents/DevOps/ANSIBLE/MyTutorials/AnsibleTestingWithMolecule/alpha-services/tasks/java.yml for instance
    
    TASK [alpha-services : Install java] *******************************************
    changed: [instance]
    
    PLAY RECAP *********************************************************************
    instance                   : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
 ```

Теперь перейдем к развитию ролей.

Следуя подходу TDD, мы сначала создаем тесты и проверяем, что они не работают, прежде чем реализовывать роли, которые они тестируют.

 

**ЗАДАЧА 1. Установите Java-1.8.0 на хост-машину**

```
---
- name: "java - check Java package status"
  package:
    name: "java-1.8.0"
    state: "installed"
  check_mode: yes
  register: pkg_status

- name: "java - test java package is installed"
  assert:
    that:
      - not pkg_status.changed
```

Задача проверки статуса пакета Java пытается установить `java-1.8.0` в режиме проверки и регистрирует результат этой операции в `pkg_status`. Фактически, если `java-1.8.0` уже установлен, утверждение `not pkg_status.changed` вернет `true`, потому что состояние не изменилось бы. Спасибо [Хуану Антонио](https://www.adictosaltrabajo.com/author/juan-antonio/) за этот совет.

Мы включаем задачи `test_java.yml` в файл `alpha-services/molecule/default/roles/test_alpha-services/tasks/main.yml` следующим образом:

 ```
---
- name: "include tasks for testing Java"
  include_tasks: "test_java.yml"
 ```



THEN ЭТАП: Запускаем `$ molecule verify`. Как и ожидалось, произойдет сбой со следующей ошибкой:

```
$ molecule verify

--> Test matrix
    
└── default
    └── verify
    
--> Scenario: 'default'
--> Action: 'verify'
--> Running Ansible Verifier
--> Sanity checks: 'docker'
    
    PLAY [Verify] ******************************************************************
    
    TASK [Gathering Facts] *********************************************************
    ok: [instance]
    
    TASK [Include test_alpha-services] *********************************************
    
    TASK [test_alpha-services : include tasks for testing Java] ********************
    included: /Users/chukwudiuzoma/Documents/DevOps/ANSIBLE/MyTutorials/AnsibleTestingWithMolecule/alpha-services/molecule/default/roles/test_alpha-services/tasks/test_java.yml for instance
    
    TASK [test_alpha-services : Check Java package status] *************************
    changed: [instance]
    
    TASK [test_alpha-services : Test java package is installed] ********************
fatal: [instance]: FAILED! => {
    "assertion": "not pkg_status.changed",
    "changed": false,
    "evaluated_to": false,
    "msg": "Assertion failed"
}
    
    PLAY RECAP *********************************************************************
    instance                   : ok=3    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
    
ERROR:
```

Теперь мы реализуем Задачу 1. Сначала мы создаем файл `alpha-services/tasks/java.yml` и заполняем его следующим:

 ```
---
- name: "Install '{{ java_required_software }}'"
  package:
    name: "{{ java_required_software }}"
    lock_timeout: 60
    state: "present"
 ```

Затем мы включаем задачи `java.yml` в файл `alpha-services/tasks/main.yml` следующим образом:

 ```
---
- name: "Include java installation tasks"
  include_tasks: "java.yml"
 ```

WHEN ФАЗА: Теперь мы запускаем `$ molecule converge`, чтобы произвести изменения в экземпляре.

THEN ФАЗА: Здесь мы запускаем `$ molecule verify`, которая должна пройти тест, если фаза схождения прошла успешно.

 ```
$ molecule verify

--> Test matrix
    
└── default
    └── verify
    
--> Scenario: 'default'
--> Action: 'verify'
--> Running Ansible Verifier
--> Sanity checks: 'docker'
    
    PLAY [Verify] ******************************************************************
    
    TASK [Gathering Facts] *********************************************************
    ok: [instance]
    
    TASK [Include test_alpha-services] *********************************************
    
    TASK [test_alpha-services : include tasks for testing Java] ********************
    included: /Users/chukwudiuzoma/Documents/DevOps/ANSIBLE/MyTutorials/AnsibleTestingWithMolecule/alpha-services/molecule/default/roles/test_alpha-services/tasks/test_java.yml for instance
    
    TASK [test_alpha-services : Check Java package status] *************************
    ok: [instance]
    
    TASK [test_alpha-services : Test java package is installed] ********************
    ok: [instance] => {
        "changed": false,
        "msg": "All assertions passed"
    }
    
    PLAY RECAP *********************************************************************
    instance                   : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    
Verifier completed successfully.
 ```

ЗАДАЧА 2. Создайте каталог по пути `/var/log/tomcat`, принадлежащий владельцу `tomcat`, группе `tomcat` и режиму `0755`.

```
---
- name: "tomcat - '{{ test_tomcat_home_dir }}' - retrieve information from path"
  stat:
    path: "{{ test_tomcat_home_dir }}"
  register: directory

- name: "tomcat - assert that directory '{{ test_tomcat_home_dir }}' is created correctly"
  assert:
    that:
      - "directory.stat.exists"
      - "directory.stat.isdir"
      - "directory.stat.mode == {{ test_tomcat_mode }}"
      - "directory.stat.pw_name == {{ test_tomcat_user }}"
      - "directory.stat.gr_name == {{ test_tomcat_group}}"
```

Мы определяем переменные в yml-файле тестовых значений по умолчанию Molecule:

```
---
#TOMCAT
test_tomcat_mode: "0755"
test_tomcat_user: "tomcat"
test_tomcat_group: "tomcat"
test_tomcat_home_dir: "/var/log/tomcat"
```

Первая задача использует модуль `stat` Ansible для получения статуса файловой системы, а вторая задача проверяет соответствие статусов.

Затем мы включаем задачу `test_java.yml` в файл `alpha-services/molecule/default/roles/test_alpha-services/tasks/main.yml` следующим образом:

 ```
---
- name: "include tasks for testing Tomcat"
  include_tasks: "test_tomcat.yml"
 ```

После этого мы продолжаем запускать `$ molecule verify`, которая по праву терпит неудачу, как и в предыдущем тесте. Поэтому мы реализуем актуальную задачу:

```
---
- name: "tomcat - create required tomcat logging directory"
  file:
    path: "{{ tomcat_home_dir }}"
    state: "directory"
    mode: "0755"
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    recurse: yes
```

Мы определяем переменные в файле yml фактических значений по умолчанию для роли:

```
---
#TOMCAT
tomcat_mode: "0755"
tomcat_user: "tomcat"
tomcat_group: "tomcat"
tomcat_home_dir: "/var/log/tomcat"
```

Мы включаем задачи tomcat.yml в файл `alpha-services/tasks/main.yml` следующим образом:

 ```
---
- name: "Include java installation tasks"
  include_tasks: "java.yml"
 ```

КОГДА ФАЗА: Теперь мы запускаем `$ molecule converge`, чтобы произвести изменения в экземпляре.

ТОГДА ФАЗА: Затем мы запускаем `$ molecule verify`, которая должна пройти тест, если фаза схождения прошла успешно.

 

**ЗАДАЧА 3: Установить, запустить и включить httpd**

Мы не будем тестировать эту задачу, потому что Ansible делает это за нас. Как указано в [документации Ansible](https://docs.ansible.com/ansible/latest/reference_appendices/test_strategies.html), «ресурсы Ansible - это модели желаемого состояния. Таким образом, нет необходимости проверять, запущены ли службы, установлены ли пакеты или что-то подобное. Ansible - это система, которая гарантирует декларативную истинность этих вещей ».

Таким образом, если служба не завершает работу, а мы пытаемся ее запустить, задача завершится ошибкой, показанной ниже:

```
TASK [alpha-services : httpd - start and enable httpd service] *****************
fatal: [instance]: FAILED! => {"changed": false, "msg": "Could not find the requested service httpd: host"}
```

Поэтому будем реализовывать только задачу:

 ```
---
- name: "Httpd - install httpd service"
  package:
    name: "httpd"
    state: "latest"

- name: "Httpd - start and enable httpd service"
  service:
    name: "httpd"
    state: "started"
    enabled: "yes"
 ```

Стоит отметить, что для запуска `httpd` в системах Linux требуется systemd, которого по умолчанию нет в контейнерах докеров. Чтобы иметь возможность запускать службу в контейнере докеров, мы добавляем следующий отредактированный ключ `platforms` в файл Molelele.yml:

 ```
---
platforms:
  - name: instance
    image: docker.io/pycontribs/centos:7
    pre_build_image: false # we don't need ansible installed on the instance
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
 ```

Для получения дополнительной информации о запуске systemd смотрите https://molecule.readthedocs.io/en/latest/examples.html

Теперь мы запускаем `$ molecule create` и `$ molecule converge`. Если они оба работают успешно, тогда служба httpd запущена и работает. Чтобы вручную проверить службу httpd, мы запускаем:

 ```
$ molecule login         # this logs you into the docker container shell

$ systemctl | grep httpd
httpd.service loaded active running The Apache HTTP Server

$ exit                  # this logs you out of the docker container to your local terminal
 ```



**ЗАДАЧА 4. Скопируйте файл шаблона из template/tomcat/context.xml в /etc/tomcat/context.xml.**

Как и раньше, мы будем проверять наличие именно того файла, который мы хотим скопировать с контроллера в хост-систему. Мы хотим проверить, соответствует ли имя файла ожидаемому и, возможно, присутствует какое-то конкретное содержимое файла, как и ожидалось. Мы могли допустить ошибку при названии файла или создании содержимого файла, и это то, что нам нужно проверить. По умолчанию, если файл не копируется, Ansible сообщит нам об этом.

Мы добавляем в соответствующие файлы следующее:

 ```
- name: "tomcat - test tomcat file"
  block:
    - name: "tomcat - retrieve information from path '{{ test_tomcat_context_xml_file }}'"
      stat:
        path: "{{ test_tomcat_context_xml_file }}"
      register: remote_file
    - name: "tomcat - assert that '{{ test_tomcat_context_xml_file }}' file is created correctly"
      assert:
        that:
          - "remote_file.stat.exists"
          - "remote_file.stat.isreg" # is a regular file
          - "remote_file.stat.path == '{{ test_tomcat_context_xml_file }}'"
          - "remote_file.stat.mode == '0755'"
 ```

```
test_tomcat_conf_dir: "/etc/tomcat"
test_tomcat_context_xml_file: "{{ test_tomcat_conf_dir }}/context.xml"
```

После этого мы запускаем `$ molecule verify`, чтобы убедиться, что она не работает. После этого реализуем актуальные задачи:

 ```
- name: "tomcat - copy dynamic tomcat server config files"
  template:
    src: "{{ tomcat_context_xml_file }}"
    dest: "{{ tomcat_conf_dir }}"
 ```

```
tomcat_conf_dir: "/etc/tomcat"
tomcat_context_xml_file: "tomcat/context.xml"
```

Затем мы запускаем `$ molecule converge`, а затем `$ molecule verify`. Тесты должны пройти, если все сделано правильно.

Наконец, чтобы быть уверенным, мы запускаем `$ molecule test`, чтобы выполнить всю Molecule `test_sequence`. Все должно работать без сбоев.

 

5. Выводы

В заключение, на мой взгляд, это правильный подход к разработке тестов Molecule для ролей Ansible. Код инфраструктуры следует протестировать перед развертыванием в производственной среде, чтобы избежать неприятных сюрпризов. Это руководство было простой демонстрацией того, как можно проводить тестирование Ansible с помощью Molecule, используя Ansible Verifier. Таким образом, нет необходимости изучать другой язык программирования, такой как Python, Ruby или Go.

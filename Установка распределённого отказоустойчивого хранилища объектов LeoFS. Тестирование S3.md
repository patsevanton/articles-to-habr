Согласно [Opennet](https://www.opennet.ru/opennews/art.shtml?num=48357): [LeoFS](http://leo-project.net/leofs/index.html) - распределённое отказоустойчивое хранилище объектов [LeoFS](http://leo-project.net/leofs/index.html), совместимое с клиентами, использующими API Amazon S3 и REST-API, а также поддерживающего режим работы в роли NFS-сервера. Имеются оптимизации для хранение как мелких, так и очень больших объектов, присутствует встроенный механизм кэширования, возможна репликация хранилищ между дата-центрами. Среди целей проекта отмечается достижение надёжности 99.9999999% за счёт избыточного реплицирования дубликатов и исключения единой точки отказа. Код проекта написан на языке Erlang.

LeoFS состоит из трёх компонентов:

- [LeoFS Storage](https://leo-project.net/leofs/docs/architecture/leo_storage/) - обслуживает операции добавления, извлечения и удаления объектов и метаданных, отвечает за выполнение репликации, восстановления и формирования очереди запросов клиентов.
- [LeoFS Gateway](https://leo-project.net/leofs/docs/architecture/leo_gateway/) - обслуживает HTTP-запросы и перенаправляет ответы клиентам с использованием REST-API или S3-API, обеспечивает кэширование наиболее востребованных данных в памяти и на диске.
- [LeoFS Manager](https://leo-project.net/leofs/docs/architecture/leo_manager/) - отслеживает работу узлов LeoFS Gateway и LeoFS Storage, ведёт мониторинг состояния узлов и проверяет контрольные суммы. Гарантирует целостность данных и высокую доступность хранилища.

В этом посте установим Leofs c помощью ansible-playbook, протестируем S3.

Если вы попытаетесь установить LeoFS используя официальные playbook-и, то вас ждут разные ошибки: [1](https://github.com/leo-project/leofs_ansible/issues/5),[2](https://github.com/leo-project/leofs_ansible/issues/4). В этом посте напишу что нужно сделать чтобы эти ошибки избежать.

Там где вы будете запускать ansible-playbook, нужно установить netcat.

Пример inventory (в репозитории hosts.sample):

```ini
# Please check roles/common/vars/leofs_releases for available versions
[all:vars]
leofs_version=1.4.3
build_temp_path="/tmp/leofs_builder"
build_install_path="/tmp/"
build_branch="master"
source="package"

[builder]
172.26.9.177

# nodename of leo_manager_0 and leo_manager_1 are set at group_vars/all
[leo_manager_0]
172.26.9.176

# nodename of leo_manager_0 and leo_manager_1 are set at group_vars/all
[leo_manager_1]
172.26.9.178

[leo_storage]
172.26.9.179 leofs_module_nodename=S0@172.26.9.179
172.26.9.181 leofs_module_nodename=S0@172.26.9.181
172.26.9.182 leofs_module_nodename=S0@172.26.9.182
172.26.9.183 leofs_module_nodename=S0@172.26.9.183
172.26.9.184 leofs_module_nodename=S0@172.26.9.184

[leo_gateway]
172.26.9.180 leofs_module_nodename=G0@172.26.9.180

[leofs_nodes:children]
leo_manager_0
leo_manager_1
leo_gateway
leo_storage
```



Отключение Selinux. Надеюсь что сообщество напишит политики Selinux для LeoFS.

```yaml
    - name: Install libselinux as prerequisite for SELinux Ansible module
      yum:
        name: "{{item}}"
        state: latest
      with_items:
        - libselinux-python
        - libsemanage-python

    - name: Disable SELinux at next reboot
      selinux:
        state: disabled

    - name: Set SELinux in permissive mode until the machine is rebooted
      command: setenforce 0
      ignore_errors: true
      changed_when: false

```

Установка `netcat` и `redhat-lsb-core`. `netcat` нужен для `leofs-adm`,  `redhat-lsb-core` нужен для  определения версии ОС [здесь](https://github.com/leo-project/leofs_ansible/blob/master/roles/common/tasks/install_package.yml#L5).

```yaml
    - name: Install Packages
      yum: name={{ item }} state=present
      with_items:
        - nmap-ncat
        - redhat-lsb-core
```

Создание юзера leofs и добавление его в группу wheel

```yaml
    - name: Create user leofs
      group:
        name: leofs
        state: present

    - name: Allow 'wheel' group to have passwordless sudo
      lineinfile:
        dest: /etc/sudoers
        state: present
        regexp: '^%wheel'
        line: '%wheel ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: Add the user 'leofs' to group 'wheel'
      user:
        name: leofs
        groups: wheel
        append: yes
```

Установка Erlang

```yaml
    - name: Remote erlang-20.3.8.23-1.el7.x86_64.rpm install with yum
      yum: name=https://github.com/rabbitmq/erlang-rpm/releases/download/v20.3.8.23/erlang-20.3.8.23-1.el7.x86_64.rpm

```

Далее выполняем как написано в <https://github.com/leo-project/leofs_ansible> без build_leofs.yml

```bash
## Install LeoFS
$ ansible-playbook -i hosts install_leofs.yml

## Config LeoFS
$ ansible-playbook -i hosts config_leofs.yml

## Start LeoFS
$ ansible-playbook -i hosts start_leofs.yml
```

Проверяем статус кластера

```bash
leofs-adm status
```

Вывод будет примерно такой

```bash
 [System Confiuration]
-----------------------------------+----------
 Item                              | Value    
-----------------------------------+----------
 Basic/Consistency level
-----------------------------------+----------
                    system version | 1.4.3
                        cluster Id | leofs_1
                             DC Id | dc_1
                    Total replicas | 2
          number of successes of R | 1
          number of successes of W | 1
          number of successes of D | 1
 number of rack-awareness replicas | 0
                         ring size | 2^128
-----------------------------------+----------
 Multi DC replication settings
-----------------------------------+----------
 [mdcr] max number of joinable DCs | 2
 [mdcr] total replicas per a DC    | 1
 [mdcr] number of successes of R   | 1
 [mdcr] number of successes of W   | 1
 [mdcr] number of successes of D   | 1
-----------------------------------+----------
 Manager RING hash
-----------------------------------+----------
                 current ring-hash | ef9549a6
                previous ring-hash | ef9549a6
-----------------------------------+----------

 [State of Node(s)]
-------+----------------------+--------------+---------+----------------+----------------+----------------------------
 type  |         node         |    state     | rack id |  current ring  |   prev ring    |          updated at         
-------+----------------------+--------------+---------+----------------+----------------+----------------------------
  S    | S0@172.26.9.179      | running      |         | ef9549a6       | ef9549a6       | 2019-12-05 08:36:52 +0000
  S    | S0@172.26.9.181      | running      |         | ef9549a6       | ef9549a6       | 2019-12-05 08:36:53 +0000
  S    | S0@172.26.9.182      | running      |         | ef9549a6       | ef9549a6       | 2019-12-05 08:36:52 +0000
  S    | S0@172.26.9.183      | attached     |         |                |                | 2019-12-05 08:37:07 +0000
  S    | S0@172.26.9.184      | running      |         | ef9549a6       | ef9549a6       | 2019-12-05 08:36:52 +0000
  G    | G0@172.26.9.180      | running      |         | ef9549a6       | ef9549a6       | 2019-12-05 08:36:55 +0000
-------+----------------------+--------------+---------+----------------+----------------+----------------------------

```

Создаем юзера leofs:

```bash
leofs-adm create-user leofs leofs
```

Создал юзера

```bash
leofs-adm create-user leofs leofs
  access-key-id: 9c2615f32e81e6a1caf5
  secret-access-key: 8aaaa35c1ad78a2cbfa1a6cd49ba8aaeb3ba39eb
```

Список юзеров:

```bash
leofs-adm get-users
user_id     | role_id | access_key_id          | created_at                
------------+---------+------------------------+---------------------------
_test_leofs | 9       | 05236                  | 2019-12-02 06:56:49 +0000
leofs       | 1       | 9c2615f32e81e6a1caf5   | 2019-12-02 10:43:29 +0000
```

Сделал bucket

```bash
leofs-adm add-bucket leofs 9c2615f32e81e6a1caf5
OK
```

Список bucket:

```bash
 leofs-adm get-buckets
cluster id   | bucket   | owner  | permissions      | created at                
-------------+----------+--------+------------------+---------------------------
leofs_1      | leofs    | leofs  | Me(full_control) | 2019-12-02 10:44:02 +0000
```

Конфигурирование s3cmd. В поле `HTTP Proxy server name` указываем IP сервера Gateway

```bash
s3cmd --configure 

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
Access Key [9c2615f32e81e6a1caf5]: 
Secret Key [8aaaa35c1ad78a2cbfa1a6cd49ba8aaeb3ba39eb]: 
Default Region [US]: 

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
S3 Endpoint [s3.amazonaws.com]: 

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: leofs

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password: 
Path to GPG program [/usr/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
Use HTTPS protocol [No]: 

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
HTTP Proxy server name [172.26.9.180]: 
HTTP Proxy server port [8080]: 

New settings:
  Access Key: 9c2615f32e81e6a1caf5
  Secret Key: 8aaaa35c1ad78a2cbfa1a6cd49ba8aaeb3ba39eb
  Default Region: US
  S3 Endpoint: s3.amazonaws.com
  DNS-style bucket+hostname:port template for accessing a bucket: leofs
  Encryption password: 
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: False
  HTTP Proxy server name: 172.26.9.180
  HTTP Proxy server port: 8080

Test access with supplied credentials? [Y/n] Y
Please wait, attempting to list all buckets...
Success. Your access key and secret key worked fine :-)

Now verifying that encryption works...
Not configured. Never mind.

Save settings? [y/N] y
Configuration saved to '/home/user/.s3cfg'
```

Если у вас появляется ошибка ERROR: S3 error: 403 (AccessDenied): Access Denied:

```bash
s3cmd put test.py s3://leofs/
upload: 'test.py' -> 's3://leofs/test.py'  [1 of 1]
 382 of 382   100% in    0s     3.40 kB/s  done
ERROR: S3 error: 403 (AccessDenied): Access Denied
```

То нужно в конфиге s3cmd поправить signature_v2 на True. Подробности в этом [issue](https://github.com/leo-project/leofs/issues/487).

Тестирование загрузки.

Создаем файл 1ГБ

```bash
fallocate -l 1GB 1gb
```

Загружаем его в Leofs 

```bash
time s3cmd put 1gb s3://leofs/
real	12m29,169s
user	0m12,961s
sys	0m1,672s
```


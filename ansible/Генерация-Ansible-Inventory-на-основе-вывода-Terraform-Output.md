Генерация Ansible Inventory на основе вывода Terraform Output или Infractructure as Code: Terraform + Ansible.

Исходные данные:

- есть пустое облако (Например, Yandex Cloud).
- есть конфигурация в Git, в котором описана инфраструктура.

Необходимо:

- Поднять виртуальные машины, сети.
- Установить необходимый софт на эти виртуальные машины.

Пост навеян https://www.linkbynet.com/produce-an-ansible-inventory-with-terraform, только адаптирован под Yandex Cloud. И так же добавлен пример по установке Postgres_cluster на свежесозданные виртуальные машины.


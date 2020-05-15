Minio это простое, быстрое и совместимое с AWS S3 хранилище объектов. Minio создан для размещения неструктурированных данных, таких как фотографии, видеозаписи, файлы журналов, резервные копии. В minio также поддерживается распределенный режим (distributed mode), который предоставляет возможность подключения к одному серверу хранения объектов множества дисков, в том числе расположенных на разных машинах.

Цель этого поста настроить minio так чтобы каждый юзер мог работать только со своим bucket.
<cut />
В целом, Minio подходит для следующих случаев:

- хранилище без репликации поверх надежной файловой системы с доступом по S3 (малые и средние хранилища, размещенные на NAS и SAN);
- хранилище без репликации поверх ненадежной файловой системы с доступом по S3 (для разработки и тестирования);
- хранилище с репликацией на небольшой группе серверов в одной стойке с доступом по протоколу S3 (отказоустойчивое хранилище с доменом отказа равным стойке).

На RedHat системах подключаем неофициальный репозиторий Minio.
```
yum -y install yum-plugin-copr
yum copr enable -y lkiesow/minio
yum install -y minio minio-mc
```
Генерируем и добавляем в MINIO_ACCESS_KEY и MINIO_SECRET_KEY в /etc/minio/minio.conf.
```
# Custom username or access key of minimum 3 characters in length.
MINIO_ACCESS_KEY=

# Custom password or secret key of minimum 8 characters in length.
MINIO_SECRET_KEY=
```

Если вы не будете использовать nginx перед Minio, то нужно изменить.
```
--address 127.0.0.1:9000
```
на
```
--address 0.0.0.0:9000
```

Запускаем Minio.
```
systemctl start minio
```

Создаем подключение к Minio под названием myminio.
```
minio-mc config host add myminio http://localhost:9000 MINIO_ACCESS_KEY 
MINIO_SECRET_KEY
```

Создаем bucket user1bucket.
```
minio-mc mb myminio/user1bucket
```

Создаем bucket user1bucket.
```
minio-mc mb myminio/user2bucket
```

Создаем файл политики user1-policy.json.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:PutBucketPolicy",
        "s3:GetBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:ListAllMyBuckets",
        "s3:ListBucket"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::user1bucket"
      ],
      "Sid": ""
    },
    {
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListMultipartUploadParts",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::user1bucket/*"
      ],
      "Sid": ""
    }
  ]
}
```

Создаем файл политики user2-policy.json.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:PutBucketPolicy",
        "s3:GetBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:ListAllMyBuckets",
        "s3:ListBucket"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::user2bucket"
      ],
      "Sid": ""
    },
    {
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListMultipartUploadParts",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::user2bucket/*"
      ],
      "Sid": ""
    }
  ]
}
```

Создаем пользователя user1 с паролем test12345.
```
minio-mc admin user add myminio user1 test12345
```

Создаем пользователя user2 с паролем test54321.
```
minio-mc admin user add myminio user2 test54321
```

Создаем политику в Minio под названием user1-policy из файла user1-policy.json.
```
minio-mc admin policy add myminio user1-policy user1-policy.json
```

Создаем политику в Minio под названием user2-policy из файла user2-policy.json.
```
minio-mc admin policy add myminio user2-policy user2-policy.json
```

Применяем политику user1-policy к юзеру user1.
```
minio-mc admin policy set myminio user1-policy user=user1
```

Применяем политику user2-policy к юзеру user2.
```
minio-mc admin policy set myminio user2-policy user=user2
```

Проверяем подключение политик к пользователям
```
minio-mc admin user list myminio
```
Проверка подключения политик к пользователям будет примерно такая
```
enabled    user1                 user1-policy
enabled    user2                 user2-policy
```

Для наглядности заходим через браузер по адресу http://ip-сервера-где-запущен-minio:9000/minio/

Видим что мы подключились к Minio под MINIO_ACCESS_KEY=user1. Для нас доступен bucket user1bucket.

![](https://habrastorage.org/webt/hp/xs/kz/hpxskzoxakkwtr5czrforp3ay60.png)

Создать bucket не получится, так соответствующего Action в политике нет.

![](https://habrastorage.org/webt/wi/ox/j2/wioxj2oadnwmplczoqly3hcdaly.png)

Создадим файл в bucket user1bucket.

![](https://habrastorage.org/webt/23/_6/o7/23_6o7onuqt5iycguk1bjvlqoyy.png)

Подключимся к Minio под MINIO_ACCESS_KEY=user2. Для нас доступен bucket user2bucket.

И не видим ни user1bucket ни файлов из user1bucket.

![](https://habrastorage.org/webt/z7/il/no/z7ilnonujyycvxx_rukael-kkra.png)

Создал Telegram чат по Minio https://t.me/minio_ru

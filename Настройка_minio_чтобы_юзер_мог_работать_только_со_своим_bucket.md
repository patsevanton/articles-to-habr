Minio это простое, быстрое и совместимое с AWS S3 хранилище объектов. Minio создан для размещения неструктурированных данных, таких как фотографии, видеозаписи, файлы журналов, резервные копии. В minio также поддерживается распределенный режим (distributed mode), который предоставляет возможность подключения к одному серверу хранения объектов множества дисков, в том числе расположенных на разных машинах.

В целом, Minio подходит для следующих случаев:

- хранилище без репликации поверх надежной файловой системы с доступом по S3 (малые и средние хранилища, размещенные на NAS и SAN);
- хранилище без репликации поверх ненадежной файловой системы с доступом по S3 (для разработки и тестирования);
- хранилище с репликацией на небольшой группе серверов в одной стойке с доступом по протоколу S3 (отказоустойчивое хранилище с доменом отказа равным стойке).

Цель этого поста настрить minio так чтобы каждый юзер мог работать только со своим bucket.

На RedHat системах подключаем неофициальный репозиторий Minio.
```
yum -y install yum-plugin-copr
yum copr enable -y lkiesow/minio
yum install -y minio minio-mc
```
Генерируем и добавляем в MINIO_ACCESS_KEY и MINIO_SECRET_KEY в /etc/minio/minio.conf
```
# Custom username or access key of minimum 3 characters in length.
#MINIO_ACCESS_KEY=

# Custom password or secret key of minimum 8 characters in length.
#MINIO_SECRET_KEY=
```

Если вы не будете использовать nginx перед Minio, то нужно изменить
```
--address 127.0.0.1:9000
```
на
```
--address 0.0.0.0:9000
```

Запускаем Minio
```
systemctl start minio
```

Создаем подключение к Minio под названием myminio
```
mc config host add myminio http://localhost:9000 MINIO_ACCESS_KEY 
MINIO_SECRET_KEY
```

Создаем bucket user1bucket
```
mc mb myminio/user1bucket
```

Создаем bucket user1bucket
```
mc mb myminio/user2bucket
```

Создаем файл политики
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

```
mc admin user add myminio user1 test12345
```
```
mc admin user add myminio user2 test54321
```
```
mc admin policy add myminio user1-policy user1-policy.json
```
```
mc admin policy add myminio user2-policy user2-policy.json
```
```
mc admin policy set myminio user1-policy user=user1
```
```
mc admin policy set myminio user2-policy user=user2
```
```
mc admin user list myminio
```
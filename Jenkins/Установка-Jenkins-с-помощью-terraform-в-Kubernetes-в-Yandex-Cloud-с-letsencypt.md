Установка Jenkins с помощью terraform в Kubernetes в Yandex Cloud с letsencypt

В этой статье будет следующее:

- Заведение DNS домена на reg.ru
- Управление DNS зоной в Yandex DNS c помощью terraform
- Создание Kubernetes в Yandex Cloud
- Резервируем внешний статический IP адрес
- Установка Jenkins c помощью terraform модуля helm_release 
- Создание ClusterIssue(Issue) для создания letsencypt сертификата



**Быстрая установка Jenkins с помощью terraform в Kubernetes в Yandex Cloud с letsencypt**

Скачать репозиторий

```
git clone https://github.com/patsevanton/infrastructure-as-a-code-example.git
```

Перейти в каталог terraform-helm-release-jenkins

```
cd terraform-helm-release-jenkins
```

Заполнить private.auto.tfvars на базе шаблона private.auto.tfvars.example

Запустить установку

```
k8s_install.sh
```

Разберем подробнее.

Заводим DNS домен на reg.ru - `mycompany.ru` или `mycompany.org.ru`. В настройках домена на вкладке "DNS-серверы и управление зоной" указываем DNS Yandex Cloud:

- ns1.yandexcloud.net
- ns2.yandexcloud.net

DNS зона управляется с помощью terraform кода. Документация - https://cloud.yandex.ru/docs/dns/.

```
resource "yandex_dns_zone" "dns_domain" {
  name   = replace(var.dns_domain, ".", "-")
  zone   = join("", [var.dns_domain, "."])
  public = true
}
```

Так как у zone на конце должна быть точка, то используем метод join для соединения нашего домена и точки.



**Создание DNS записи.** 

В имени DNS записи на конце необходима точка, поэтому используем метод join для соединения DNS записи и точки. Код:

```
resource "yandex_dns_recordset" "jenkins_dns_domain" {
  zone_id = yandex_dns_zone.dns_domain.id
  name    = join("", [var.jenkins_dns_name, "."])
  type    = "A"
  ttl     = 200
  data    = [yandex_vpc_address.addr.external_ipv4_address[0].address]
}
```

Поле data - список IP адресов, на которые должны резолвиться эта DNS запись.



**Создание сервисного аккаунта**

Перед созданием Kubernetes кластера необходимо рассказать о создание сервисного аккаунта. Название сервисного аккаунта - `sa-k8s-admin`.

```
resource "yandex_iam_service_account" "sa-k8s-admin" {
  folder_id = var.yc_folder_id
  name      = "sa-k8s-admin"
}

resource "yandex_resourcemanager_folder_iam_member" "sa-k8s-admin-permissions" {
  folder_id = var.yc_folder_id
  role      = "admin"
  member    = "serviceAccount:${yandex_iam_service_account.sa-k8s-admin.id}"
}
```

Лучше использовать `yandex_resourcemanager_folder_iam_member` вместо `yandex_resourcemanager_folder_iam_binding`, так как не всегда работает корректно. У меня на другом коде вызывает ошибку.  Issue - https://github.com/yandex-cloud/terraform-provider-yandex/issues/267 и https://github.com/yandex-cloud/terraform-provider-yandex/issues/283.

```
"Binding for role "mdb.dataproc.agent" not found in policy"
```

В `role `  указываем какие права имеет сервисный аккаунт.  Более подробно в документации https://cloud.yandex.ru/docs/iam/concepts/access-control/roles.



**Создание kubernetes кластера в Yandex Cloud**

Для создание kubernetes кластера в Yandex Cloud используется уже обычный terraform код:

```
resource "yandex_kubernetes_cluster" "zonal_k8s_cluster" {
  name        = "my-cluster"
  description = "my-cluster description"
  network_id  = yandex_vpc_network.k8s-network.id

  master {
    version = "1.21"
    zonal {
      zone      = yandex_vpc_subnet.k8s-subnet.zone
      subnet_id = yandex_vpc_subnet.k8s-subnet.id
    }
    public_ip = true
  }

  service_account_id      = yandex_iam_service_account.sa-k8s-admin.id
  node_service_account_id = yandex_iam_service_account.sa-k8s-admin.id
  release_channel         = "STABLE"
  // to keep permissions of service account on destroy
  // until cluster will be destroyed
  depends_on = [yandex_resourcemanager_folder_iam_member.sa-k8s-admin-permissions]
}

# yandex_kubernetes_node_group

resource "yandex_kubernetes_node_group" "k8s_node_group" {
  cluster_id  = yandex_kubernetes_cluster.zonal_k8s_cluster.id
  name        = "name"
  description = "description"
  version     = "1.21"

  labels = {
    "key" = "value"
  }

  instance_template {
    platform_id = "standard-v3"

    network_interface {
      nat        = true
      subnet_ids = [yandex_vpc_subnet.k8s-subnet.id]
    }

    resources {
      cores         = 2
      memory        = 4
      core_fraction = 50
    }

    boot_disk {
      type = "network-hdd"
      size = 32
    }

    scheduling_policy {
      preemptible = true
    }

    metadata = {
      ssh-keys = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
    }

  }

  scale_policy {
    fixed_scale {
      size = 1
    }
  }

  allocation_policy {
    location {
      zone = "ru-central1-b"
    }
  }

  maintenance_policy {
    auto_upgrade = true
    auto_repair  = true

    maintenance_window {
      day        = "monday"
      start_time = "15:00"
      duration   = "3h"
    }

    maintenance_window {
      day        = "friday"
      start_time = "10:00"
      duration   = "4h30m"
    }
  }
}

locals {
  kubeconfig = <<KUBECONFIG
apiVersion: v1
clusters:
- cluster:
    server: ${yandex_kubernetes_cluster.zonal_k8s_cluster.master[0].external_v4_endpoint}
    certificate-authority-data: ${base64encode(yandex_kubernetes_cluster.zonal_k8s_cluster.master[0].cluster_ca_certificate)}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: yc
  name: ycmk8s
current-context: ycmk8s
users:
- name: yc
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: yc
      args:
      - k8s
      - create-token
KUBECONFIG
}

output "kubeconfig" {
  value = local.kubeconfig
}
```

Если кластер тестовый, то можно снизить стоимость используя 50% ядра и [использовать прерываемые виртуальные машины.](https://cloud.yandex.ru/docs/compute/pricing#prices-instance-resources)

```
    resources {
      ...
      core_fraction = 50
    }
    
    scheduling_policy {
      preemptible = true
    }
```



**Сеть и внешний статический IP адрес.**

Указываем сеть

```
resource "yandex_vpc_network" "k8s-network" {
  name = "k8s-network"
}
```

 Указываем подсеть

```
resource "yandex_vpc_subnet" "k8s-subnet" {
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.k8s-network.id
  v4_cidr_blocks = ["10.5.0.0/24"]
  depends_on = [
    yandex_vpc_network.k8s-network,
  ]
}
```

Указываем внешний статический IP адрес

```
resource "yandex_vpc_address" "addr" {
  name = "static-ip"
  external_ipv4_address {
    zone_id = "ru-central1-b"
  }
}
```



**Jenkins установливаем c помощью terraform модуля helm_release.** 

Рассмотрим `helm_release.tf`:

`provider "helm"` описывает настройки подключения к kubernetes для helm.

```
provider "helm" {
  kubernetes {
    host                   = yandex_kubernetes_cluster.zonal_k8s_cluster.master[0].external_v4_endpoint
    cluster_ca_certificate = yandex_kubernetes_cluster.zonal_k8s_cluster.master[0].cluster_ca_certificate
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      args        = ["k8s", "create-token"]
      command     = "yc"
    }
  }
}
```

**Создаем ingress-nginx.**

```
resource "helm_release" "ingress_nginx" {
  name       = "ingress-nginx"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.2.1"
  wait       = true
  depends_on = [
    yandex_kubernetes_node_group.k8s_node_group
  ]
  set {
    name  = "controller.service.loadBalancerIP"
    value = yandex_vpc_address.addr.external_ipv4_address[0].address
  }
}
```

В terraform опции `--set key=value` передаются так:

```
  set {
    name  = "controller.service.loadBalancerIP"
    value = yandex_vpc_address.addr.external_ipv4_address[0].address
  }
```

В данном случае для `controller.service.loadBalancerIP` указываем внешний статический IP адрес.



**Устанавливаем cert-manager по аналогии с ingress-nginx**

```
resource "helm_release" "cert-manager" {
  namespace        = "cert-manager"
  create_namespace = true
  name             = "jetstack"
  repository       = "https://charts.jetstack.io"
  chart            = "cert-manager"
  version          = "v1.9.1"
  wait             = true
  depends_on = [
    yandex_kubernetes_node_group.k8s_node_group
  ]
  set {
    name  = "installCRDs"
    value = true
  }
}
```



**Устанавливаем jenkins**

```
resource "helm_release" "jenkins" {
  namespace        = "jenkins"
  create_namespace = true
  name             = "jenkins"
  repository       = "https://charts.jenkins.io"
  chart            = "jenkins"
  wait             = true
  version          = "4.1.17"
  depends_on = [
    yandex_kubernetes_node_group.k8s_node_group
  ]
  values = [
    #file("${path.module}/jenkins-values-google-login.yaml")
    yamlencode(local.jenkins_values_google_login)
  ]
}
```

Jenkins helm чарту на вход необходимо указать value.yaml с нашими настройками. 

Есть 2 способа указать value.yaml.

Первый способ использовать готовый и настроеный value.yaml в текущей директории

```
  values = [
    file("${path.module}/jenkins-values-google-login.yaml")
  ]
```

Второй способ использовать yamlencode для перевода terraform кода в yaml код.

```
  values = [
    yamlencode(local.jenkins_values_google_login)
  ]
```



**Как получить terraform код из yaml кода?**

Подготавливаете и настраиваете value.yaml, а затем используя вот такую команду переводите в terraform код:

```
echo 'yamldecode(file("value.yaml"))' | terraform console 
```

В моем случае value.yaml имеет другое название:

```
echo 'yamldecode(file("jenkins-values-google-login.yaml"))' | terraform console 
```

Я использую yamlencode, потому что внутри yamlencode кода ссылаюсь на terraform переменные, а так же потому что job лежат отдельно.

Полученный код вставляете в `local.jenkins_values_google_login`

```
locals {
  jenkins_values_google_login = {
  сюда
  }
}
```

Затем вместо ваших данных 

```
"hostName"         = jenkins.mycompany.ru
```

используйте terraform переменные

```
"hostName"         = "${var.jenkins_dns_name}"
```



**Рассмотрим `local.jenkins_values_google_login`.** 

В корне стоит `controller` от которого идут все настройки.

Для настройка jenkins из кода используются  [jcasc](https://www.jenkins.io/projects/jcasc/), [job-dsl](https://github.com/jenkinsci/job-dsl-plugin). Примеры jcasc https://github.com/jenkinsci/configuration-as-code-plugin

```
      "JCasC" = {
        "authorizationStrategy" = <<-EOT
        loggedInUsersCanDoAnything:
          allowAnonymousRead: false
        EOT
```

Более подробно - https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/demos/global-matrix-auth/README.md

В configScripts каждый блок EOT это отдельный файл в контейнере jenkins. `systemMessage` системное сообщение.

```
        "configScripts" = {
          "jenkins-configuration" = <<-EOT
          jenkins:
            systemMessage: This Jenkins is configured and managed 'as code' by Managed Cloud team.
          EOT
```

Указываем файл job

```
          "job-config" = yamlencode({
            jobs = [
              { script = file("${path.module}/job1.groovy") },
              { script = file("${path.module}/job2.groovy") }
            ]
          })
```

Указываем список view

```
          jenkins:
            views:
              - all:
                  name: "all"
              - list:
                  columns:
                  - "status"
                  - "weather"
                  - "jobName"
                  - "lastSuccess"
                  - "lastFailure"
                  - "lastDuration"
                  - "buildButton"
                  jobNames:
                  - "job1"
                  name: "stage"
              - list:
                  columns:
                  - "status"
                  - "weather"
                  - "jobName"
                  - "lastSuccess"
                  - "lastFailure"
                  - "lastDuration"
                  - "buildButton"
                  jobNames:
                  - "job2"
                  name: "test"
            viewsTabBar: "standard"
```

Указываем как мы будет входить в jenkins

```
        "securityRealm" = <<-EOT
        googleOAuth2:
          clientId: "${var.clientId}"
          clientSecret: "${var.clientSecret}"
          domain: "${var.google_domain}"
        EOT
```

Я использую googleOAuth2. Можно еще использовать local, ldap и другие.

Дополнительные плагины jenkins:

```
      "additionalPlugins" = [
        "google-login:1.6",
        "job-dsl:1.81",
        "allure-jenkins-plugin:2.30.2",
        "ws-cleanup:0.42",
        "build-timeout:1.21",
        "timestamper:1.18",
        "google-storage-plugin:1.5.6",
        "permissive-script-security:0.7",
        "ansicolor:1.0.2",
        "google-oauth-plugin:1.0.6",
      ]
```

Настройка Ingress

```
      "ingress" = {
        "annotations" = {
          "cert-manager.io/cluster-issuer" = "letsencrypt-prod"
        }
        "apiVersion"       = "networking.k8s.io/v1"
        "enabled"          = true
        "hostName"         = "${var.jenkins_dns_name}"
        "ingressClassName" = "nginx"
        "tls" = [
          {
            "hosts" = [
              "${var.jenkins_dns_name}",
            ]
            "secretName" = "jenkins-tls"
          },
        ]
      }
```

Если у вас нет letsencrypt, то вы удаляете cert-manager.io/cluster-issuer

Указываем javaOpts чтобы запускать Job-DSL скрипты

```
javaOpts: '-Dpermissive-script-security.enabled=true'
```

**Сравнение настроенный yaml и yamlencode из terraform кода**

Чем плох настроенный yaml в качестве value.yaml для helm чарта относительно yamlencode ?

- Он длинный
- В yaml коде нельзя вынести кусок кода (например job) в отдельный файл 
- Для формирования заполненого настроенного value.yaml необходимо использовать templatefile. Мне кажется это лишнее.

В JCasC.configScripts каждый блок до вертикальной черты (|) будет сохранен как отдельный файл в контейнере Jenkins.

В Yaml формате все job нужно расписывать.  Если сравнить

```
      job-config: |
        jobs:
          - script: >
              pipelineJob('job1') {
                logRotator(120, -1, 1, -1)
                authenticationToken('secret')
                definition {
                  cps {
                    script("""\
                      pipeline {
                        agent any
                        parameters {
                            string(name: 'Variable', defaultValue: '', description: 'Variable', trim: true)
                        }
                        options {
                          timestamps()
                          ansiColor('xterm')  
                          timeout(time: 10, unit: 'MINUTES')
                        }
                        stages {
                          stage ('build') {
                            steps {
                              cleanWs()
                              echo "hello job1"
                            }
                          }
                        }
                      }""".stripIndent())
                    sandbox()
                  }
                }
              }
```

и чтение файла job, то вынос файла лучшее красивее и читабельнее

```
          "job-config" = yamlencode({
            jobs = [
              { script = file("${path.module}/job1.groovy") },
              { script = file("${path.module}/job2.groovy") }
            ]
          })
```



Добавление `kind: ClusterIssuer` в Kubernetes

Если мы будем добавлять `kind: ClusterIssuer` как `resource "kubernetes_manifest"` и добавим как подключатся к Kubernetes используя `provider "kubernetes"`, то получим ошибку

```
cannot create REST client: no client config
```

Мы не можем развернуть `resource "kubernetes_manifest"` в котором ссылаемся на другой ресрус - https://github.com/hashicorp/terraform-provider-kubernetes/issues/1380

Поэтому создадим файл `ClusterIssuer.yaml.tpl` и будем формировать его через `templatefile` передав всего 1 переменную email_letsencrypt.


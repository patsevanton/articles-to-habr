**Snaps** являются кросс-дистрибутивными, независимыми и простыми в установке приложениями, упакованными со всеми их зависимостями для запуска во всех основных дистрибутивах Linux. **Snaps** безопасны — они ограничены и не ставят под угрозу всю систему. Они работают под разными уровнями содержания (то есть степень изоляции от базовой системы и друг от друга). 

**MicroK8s** - это CNCF-сертифицированное развертывание локального кластера Kubernetes, он предназначен для быстрой и легкой установки потока Kubernetes, изолированной от вашей локальной среды. В качестве оснастки он запускает все службы Kubernetes (т.е. без виртуальных машин), упаковывая при этом весь необходимый набор библиотек и файлов. Эта изоляция достигается за счет упаковки всех двоичных файлов для Kubernetes, Docker.io, iptables и CNI в единый пакет Snap.

Преимущества microk8s:

- использует только необходимые ему ресурсы
- кластеризация нескольких microk8s
- легкая и простая установка в Ubuntu через snap, хваленая изоляция snap пакетов, возможность легкого отката на предыдущую версию
- наличие аддонов

<cut />

**Apache Superset** - это веб-приложение для поиска и визуализации данных.

**Helm** — это диспетчер пакетов для Kubernetes, упрощающий для разработчиков и операторов упаковку, настройку и развертывание приложений и служб в кластерах Kubernetes.

Операционная система: Ubuntu 18.04

Устанавливаем Snapd, git

```
sudo apt-get update && sudo apt-get install -y snapd git
```

Устанавливаем microk8s версии 1.18
```
sudo snap install microk8s --classic --channel=1.18/stable && sudo snap install helm --classic
```

Стартуем microk8s
```
sudo microk8s.start
```

Добавляем текущего пользователя в группу microk8s
```
sudo usermod -a -G microk8s $USER
```
Меняем права директории .kube в домашней директории текущего пользователя
```
sudo chown -f -R $USER ~/.kube
```

Выходим из сессии и заходим снова
```
exit
```

Делаем алиал kubectl на microk8s.kubectl
```
alias kubectl=microk8s.kubectl
```

Активируем дополнения microk8s. В опциях дополнения metallb указываем список IP с ваших сетевых карточек. Если у вас 1 сервер, то это два одинаковых IP адреса. Кластеризацию microk8s я не проверял, но по идее нужно указывать IP адреса обоих серверов. Для этого обязательна кластеризация microk8s. IP на сетевой карте - 192.168.22.7. У вас он будет другой.

```
microk8s enable dns ingress storage metallb:192.168.22.7-192.168.22.7 
```

Смотрим что все поды у нас Running
```
kubectl get all --all-namespaces
```

Скачиваем репозиторий superset
```
git clone https://github.com/apache/superset.git
```

Переходим в директорию где хранится helm для superset
```
cd superset/helm/superset
```

Скачиваем зависимиости для текущего helm
```
helm dependency update
```

Сохраняем конфиг для подключения к Kubernetes
```
sudo microk8s.kubectl config view --raw > $HOME/.kube/config
```

Запускаем установку superset с помощью helm используя конфиги в текущей директории
```
helm install --set persistence.enabled=true,service.type=LoadBalancer,ingress.enabled=true,ingress.hosts[0]=superset.192.168.22.7.xip.io  superset ./
```

Если вы перейдет по ссылке superset.192.168.22.7.xip.io - то увидите вот такой экран.

![](https://habrastorage.org/webt/_2/_s/51/_2_s51sdyzxalh_klruws0gtavq.png)

Логин и пароль по умолчанию admin/admin. Superset настроен. Можно пользоваться.


![](https://habrastorage.org/webt/9u/xu/bf/9uxubf90dkzh71hmb95kb1q0ica.png)


Если нужно удалить весь кластер Kubernetes в microk8s, то можно воспользоваться командой reset

```
microk8s reset --destroy-storage

```

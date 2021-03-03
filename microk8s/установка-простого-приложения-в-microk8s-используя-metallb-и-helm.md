**Snaps** являются кросс-дистрибутивными, независимыми и простыми в установке приложениями, упакованными со всеми их зависимостями для запуска во всех основных дистрибутивах Linux. **Snaps** безопасны — они ограничены и не ставят под угрозу всю систему. Они работают под разными уровнями содержания (то есть степень изоляции от базовой системы и друг от друга). 



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

Активируем дополнения microk8s
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



![](https://habrastorage.org/webt/_2/_s/51/_2_s51sdyzxalh_klruws0gtavq.png)


![](https://habrastorage.org/webt/9u/xu/bf/9uxubf90dkzh71hmb95kb1q0ica.png)


Если нужно удалить весь кластер Kubernetes в microk8s, то можно воспользоваться командой reset

```
microk8s reset --destroy-storage

```

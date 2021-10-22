Kubernetes Nginx Ingress: перенаправление трафика с использованием аннотаций

![](https://habrastorage.org/webt/-k/ia/j-/-kiaj-zi-oy4u_ckhqgwautoxuy.jpeg)

Перенаправляйте HTTP-трафик или переписывайте URL-адреса с помощью входных аннотаций Kubernetes и Nginx ingress controller. В этой статье объясняется использование аннотаций и их влияние на результирующий файл конфигурации nginx.conf.

<cut />

Информация взята с [Linux Recruit Blog](https://www.linuxrecruit.co.uk/blog?title=Kubernetes%20Nginx%20Ingress%3A%20Traffic%20Redirect%20Using%20Annotations%20Demystified&id=132).

### nginx.ingress.kubernetes.io/rewrite-target

https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/rewrite

#### Пример 1:

```
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress 
metadata:
   annotations:
     kubernetes.io/ingress.class: "nginx"
     nginx.ingress.kubernetes.io/rewrite-target: /destination$1$2
   name: destination-home
   namespace: myNamespace
 spec:
   rules:
   - host: nginx.redirect
     http:
       paths:
       - backend:
           serviceName: http-svc
           servicePort: 80 
        path: /source(/|$)(.*)
```



Этот делает прозрачный обратный прокси-сервер.
Он не обновляет Location header, поэтому URL-адрес в браузере не меняется.

#### Пример 2:

```
apiVersion: networking.k8s.io/v1beta1
 kind: Ingress
 metadata:
   annotations:
     kubernetes.io/ingress.class: "nginx"
     nginx.ingress.kubernetes.io/rewrite-target: https://nginx.redirect/destination$1$2
   name: destination-home
   namespace: myNamespace
 spec:
   rules:
   - host: nginx.redirect
     http:
       paths:
       - backend:
           serviceName: http-svc
           servicePort: 80
         path: /source(/|$)(.*)
```



Этот изменяет Location header, и URL-адрес в браузере обновляется:

```
HTTP/1.1 302 Moved Temporarily
curl -I  http://nginx.redirect/source
Location: https://nginx.redirect/destination
HTTP/1.1 302 Moved Temporarily
curl -I  http://nginx.redirect/source/bar
Location: https://nginx.redirect/destination/bar
```



Это связано с тем, что строка замены, указанная в аннотации `rewrite-target`, начинается с `https://`.

[Из документации nginx](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html):

```
Кроме того, в качестве единственного параметра можно указать URL для временного перенаправления с кодом 302. Такой параметр должен начинаться со строк “http://”, “https://” или “$scheme”. В URL можно использовать переменные.
..............
Если указанное регулярное выражение соответствует URI запроса, URI изменяется в соответствии со строкой замены. Директивы rewrite выполняются последовательно, в порядке их следования в конфигурационном файле. С помощью флагов можно прекратить дальнейшую обработку директив. Если строка замены начинается с “http://”, “https://” или “$scheme”, то обработка завершается и клиенту возвращается перенаправление.
```



Использование `nginx.ingress.kubernetes.io/permanent-redirect-code : '308'` аннотации не влияет на возвращаемый код, так как это контролируется правилом перезаписи в `nginx.conf`. Директива, добавленная в `nginx.conf`, аналогична:

```
rewrite “(?i)/source(/|$)(.*)” https://nginx.redirect/destination$1$2 break;
```



```
Синтаксис: break;
По умолчанию: —
Контекст: server, location, if

Завершает обработку текущего набора директив модуля ngx_http_rewrite_module.

Если директива указана внутри location, дальнейшая обработка запроса продолжается в этом location.
```



Если вам нужно управлять кодом возврата с помощью правила rewrite, то вам необходимо использовать директиву возврата после директивы перезаписи. Более подробная информация здесь: https://www.nginx.com/blog/creating-nginx-rewrite-rules .

Я предполагаю, что это можно использовать с помощью `nginx.ingress.kubernetes.io/configuration-snippet : |` (Я еще не пробовал).

Этот метод работает очень хорошо: определяет корневой каталог приложения (Application Root), который контроллер должен перенаправить, если он находится в `/` контексте:

```
$ echo "
 apiVersion: extensions/v1beta1
 kind: Ingress
 metadata:
   annotations:
     nginx.ingress.kubernetes.io/app-root: /destination
   name: approot
   namespace: default
 spec:
   rules:
   - host: approot.bar.com
     http:
       paths:
       - backend:
           serviceName: http-svc
           servicePort: 80
         path: /
 " | kubectl create -f -
```



В результирующем файле `nginx.conf` оператор `if` будет добавлен в контекст `server` :

```
if ($uri = /) {
  return 302 /destination
}
```



### 2.nginx.ingress.kubernetes.io/configuration-snippet: |

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
   annotations:
     kubernetes.io/ingress.class: "nginx"
     nginx.ingress.kubernetes.io/configuration-snippet: |
       rewrite ^/source(/?)$ https://nginx.redirect/destination$1 permanent;
       name: destination-home
   namespace: myNamespace
 spec:
   rules:
   - host: nginx.redirect
     http:
       paths:
       - backend:
           serviceName: http-svc
           servicePort: 80
         path: /source
```



Этот вариант выглядит так, как будто он обеспечивает наибольший контроль над redirect/rewrite, поскольку он добавляет дополнительный фрагмент конфигурации в результирующее местоположение `nginx.conf`.

```
permanent: returns a permanent redirect with the 301 code.
```



Дополнительная документация [здесь](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#configuration-snippet).

### 3. nginx.ingress.kubernetes.io/server-snippet :|

Используйте осторожно. Хотя его можно использовать аналогично приведенному выше (только аннотация немного отличается), он добавит вашу пользовательскую конфигурацию в серверный блок в результирующем файле `nginx.conf`, таким образом, вступив в силу для всего сервера. Перенаправление/перезапись, размещенные здесь, будут обработаны перед любым другим оператором в директиве о местоположении (управляемой входящим ресурсом kubernetes), поэтому это может привести к нежелательному поведению.

Дополнительная документация [здесь](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#server-snippet).

### 4. nginx.ingress.kubernetes.io/permanent-redirect

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
   annotations:
     nginx.ingress.kubernetes.io/permanent-redirect: https://nginx.redirect/destination
     nginx.ingress.kubernetes.io/permanent-redirect-code: '308'
   name: destination-home
   namespace: myNamespace
 spec:
   rules:
   - host: nginx.redirect
     http:
       paths:
       - backend:
           serviceName: http-svc
           servicePort: 80
         path: /source
```



Довольно понятно, работает отлично

```
curl -I  http://nginx.redirect/source
HTTP/1.1 308 
Permanent Redirect 
Location: https://nginx.redirect/destination
curl -I  http://nginx.redirect/source/bar 
HTTP/1.1 308 
Permanent Redirect 
Location: https://nginx.redirect/destination
```



Он добавляет оператор if в файл `nginx.conf` в разделе `/source` следующим образом:

```
if ($uri ~* /source) {
     return 308 https://nginx.redirect/destination;
    }
```



Дополнительная документация: [annotations.md#permanent-redirect](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#permanent-redirect) и [здесь](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#permanent-redirect).

### Permanent Redirect 

Эта аннотация позволяет возвращать постоянное перенаправление вместо отправки данных в вышестоящий канал. Например:

```
nginx.ingress.kubernetes.io/permanent-redirect: 
https://www.google.com 
```

перенаправил бы все в Google.

### Permanent Redirect Code

Эта аннотация позволяет изменять код состояния, используемый для постоянных перенаправлений. Например `nginx.ingress.kubernetes.io/permanent-redirect-code : '308' `вернет вам постоянное перенаправление (permanent-redirect) с кодом 308.

### Temporal Redirect

Эта аннотация позволяет вам возвращать временное перенаправление (код возврата 302) вместо отправки данных в вышестоящий канал. Например `nginx.ingress.kubernetes.io/temporal-redirect : https://www.google.com` перенаправил бы все в Google с кодом возврата 302 (Временно перемещен)

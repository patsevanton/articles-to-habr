# Sentry — мониторинг java exception в Java 

Стандартно Java разработчики мониторят ошибки, exception через логи. Но есть и другой способ, а именно отправка exception в Sentry.

**Sentry** — инструмент мониторинга исключений (exception), ошибок в ваших приложениях.

**Преимущества использования Sentry:**

- не нервничать при размещении приложений на боевом сервере,
- быстро находить причины возникших проблем,
- устранять баги раньше, чем о них вам сообщат тестировщики, коллеги из саппорта, пользователи, ПМ или директор,
- выявлять незаметные остальной команде проблемы, которые портят жизнь пользователям и снижают эффективность вашего продукта,
- бесплатен,
- легко интегрируется в проект,
- ловит ошибки и в браузере пользователя, и на вашем сервере.
- Если в ELK один и тот же exception происходит несколько раз, то они идут как отдельные записи, занимают место на диске и в ОЗУ. Если в Sentry один и тот же exception происходит несколько раз, то поле EVENTS увеличивается, тем самым экономя место на диске и в ОЗУ.

**Основные возможности:**

- Список ошибок обновляется в режиме реального времени,
- Если ошибка была помечена как решенная и появилась снова, то она снова создается и учитывается в отдельном потоке,
- Ошибки группируются и отображаются в порядке частоты появления,
- Ошибки можно фильтровать по статусам, источнику логгирования, уровню логгирования, имени сервера и т.д.

Sentry поддерживает большую часть языков программирования. Подробнее [здесь](https://sentry.io/platforms/).

Устанавливаем Sentry

- через скрипт, который в docker-compose поднимет все компоненты (скрипт и репо находятся здесь https://github.com/getsentry/onpremise/)
- собираем RPM пакеты все зависимостей Sentry и устанавливаем через RPM пакеты https://habr.com/ru/post/500632/

После установки Sentry у вас должен быть DSN

![](https://habrastorage.org/webt/ss/h1/rp/ssh1rpfl6rwbsxd2xyskrz3vqiw.png)

Здесь будет обзор основных примеров отправки java exception в Sentry. Примеры будем брать из официального репозитория https://github.com/getsentry/examples/tree/master/java.

## Зависимости

На машине где будете запускать эти примеры необходимо иметь установлеными JDK и Maven.

```
# Пример для CentOS
wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
yum install -y apache-maven java-1.8.0-openjdk-devel git
```

## Скачиваем репозиторий 
```
git clone https://github.com/getsentry/examples.git
cd examples/java/
```

## Sentry Basic Example

Переходим к https://github.com/getsentry/examples/tree/master/java/basic
```
cd basic
```

Запускаем компилирование проекта

```
mvn compile exec:java
```

Запускаем Java приложение и передеаем ему SENTRY_DSN

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn exec:java
```

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/to/7z/it/to7zitklsjuc-q2qqc9aciyq2zk.png)

![](https://habrastorage.org/webt/hs/mh/tg/hsmhtg-1ydnepr90zn4cpq36emk.png)

![](https://habrastorage.org/webt/dt/yr/68/dtyr68czpmmdbjm9xtil11p2uoa.png)

![](https://habrastorage.org/webt/st/fx/rk/stfxrkmtgmef0lo2od_u_y94pno.png)

![](https://habrastorage.org/webt/g7/tx/4k/g7tx4ktv5nd8xv139wkmrvsgzv4.png)

![](https://habrastorage.org/webt/yw/rt/e9/ywrte9ibfefsmn1y4l35hrrqj1e.png)

Запустим приложение несколько раз:

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn exec:java
```

Видно что счетчик EVENTS для этого конкретного exception увеличился.

![](https://habrastorage.org/webt/xp/9r/wq/xp9rwqnnr0ha4jnqqqt9pmhoqo0.png)

## Sentry Grails Example

Переходим к https://github.com/getsentry/examples/tree/master/java/grails-3.x

Устанавливаем зависимости для SDK

```
yum install -y unzip zip
```

Устанавливаем grails используя SDK https://grails.org/download.html

```
curl -s https://get.sdkman.io | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install grails
```

Запускаем grails c SENTRY_DSN

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 grails run-app
```

Открываем в браузере http://localhost:8080/hello/index и exception отправляются в Sentry

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/2k/um/1e/2kum1eatxhwpektljhii87wtso0.png)

![](https://habrastorage.org/webt/ic/1i/1i/ic1i1i6taxqgccow5i5dmz9ieh0.png)

![](https://habrastorage.org/webt/jm/ce/9n/jmce9nraz4datevczdofshtbeik.png)

![](https://habrastorage.org/webt/ma/dl/db/madldbpyzrqf_vi_io-tgk9xe2s.png)

![](https://habrastorage.org/webt/n2/st/k1/n2stk1a1cgteegas0naczl_w_t0.png)

![](https://habrastorage.org/webt/ir/a7/uj/ira7uju9mpnxf4xuvlahhloarng.png)

![](https://habrastorage.org/webt/e_/sp/dj/e_spdjswzsrbczm_jlapf9c5j70.png)

![](https://habrastorage.org/webt/jk/kx/_c/jkkx_c8ncxj0xoofx2ixt3xenmq.png)

![](https://habrastorage.org/webt/06/am/kk/06amkklfkdpccwwarzszci-8p20.png)

![](https://habrastorage.org/webt/ia/ym/ht/iaymhtrucnntixjlgxkaxopging.png)

Запустим приложение несколько раз:

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 grails run-app
```

Видно что счетчик EVENTS для этого конкретного exception увеличился.

## Sentry java.util.logging Example

Переходим к https://github.com/getsentry/examples/tree/master/java/java.util.logging

Собираем пакет JAR

```
mvn clean package
```

Запускаем JAR

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 \
java \
-Djava.util.logging.config.file=src/main/resources/logging.properties \
-cp ./target/sentry-java-jul-example-1.0-SNAPSHOT-jar-with-dependencies.jar \
io.sentry.example.Application
```

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/ga/_t/yf/ga_tyf8lq11hchmclnn7lq8enyw.png)

![](https://habrastorage.org/webt/oa/zv/hq/oazvhqcs2tpwooowp2zqtcpmq9c.png)

![](https://habrastorage.org/webt/ts/v1/53/tsv153i9ediv8cucjngjafarzum.png)

![](https://habrastorage.org/webt/of/f0/wy/off0wyhqolxcyd08_snalamqjdu.png)

![](https://habrastorage.org/webt/7a/mj/44/7amj441my4kopy5gogv7j5ptvh4.png)

## Sentry Log4j 1.x Example

Переходим к https://github.com/getsentry/examples/tree/master/java/log4j-1.x

Компилируем

```
mvn compile exec:java
```

Запускаем 

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn exec:java
```

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/ie/3n/ej/ie3nej4fqpp63ubp5j964bpisby.png)

![](https://habrastorage.org/webt/oi/cw/3j/oicw3jhaxqu9gnbq_ao6mtykt6w.png)

![](https://habrastorage.org/webt/et/6q/35/et6q35xo5pylbzoajjdpjuunu5m.png)

![](https://habrastorage.org/webt/vx/ec/xs/vxecxsejx_zal6cu-88nvvbouei.png)

![](https://habrastorage.org/webt/gi/gc/0r/gigc0ruliax6keyywli6taeiaoo.png)

## Sentry Log4j 2.x Example

Переходим к https://github.com/getsentry/examples/tree/master/java/log4j-2.x

Компилируем

```
mvn compile exec:java
```

Запускаем

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn exec:java
```

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/__/s_/pq/__s_pq2culw5hahmigmid9xyqva.png)

![](https://habrastorage.org/webt/i9/kj/fd/i9kjfd8y1oijq-zizre1xw4sjtq.png)

![](https://habrastorage.org/webt/f_/oc/jw/f_ocjwlirs43xco3prsa_ksvmcq.png)

![](https://habrastorage.org/webt/s1/8d/bt/s18dbtuf8v4v13vwjjb4sqpa-p0.png)

![](https://habrastorage.org/webt/ho/wc/xx/howcxx8gxagkvpwvscr0qabx9ac.png)

## Sentry Logback Example

Переходим к https://github.com/getsentry/examples/tree/master/java/logback

Компилируем

```
mvn compile exec:java
```

Запускаем

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn exec:java
```

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/ck/4a/v_/ck4av_6r59eh9qp5bksh51ozyie.png)

![](https://habrastorage.org/webt/n7/oo/w6/n7oow6roc5ifb7nv33erexyzzi4.png)

![](https://habrastorage.org/webt/xd/dg/fp/xddgfpglbgefxncxf-qsg5rjwcw.png)

![](https://habrastorage.org/webt/qq/m8/gf/qqm8gfadydvofofeiivltnprrwc.png)

![](https://habrastorage.org/webt/3t/-p/j0/3t-pj0_uz8umiwilf0hr3d9et_w.png)

## Sentry Spring Boot Example

Переходим к https://github.com/getsentry/examples/tree/master/java/spring-boot

Запускаем

```
SENTRY_DSN=http://31bc93d7d64e4fd6b2e77d6d7780be6c@172.26.10.64:9000/1 mvn spring-boot:run
```

Открываем в браузере http://localhost:8080/ и exception отправляются в Sentry

В логах мы видим

```
2020-06-20 12:35:47.249 ERROR 13939 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.ArithmeticException: / by zero] with root cause

java.lang.ArithmeticException: / by zero
	at io.sentry.example.Application.home(Application.java:44) ~[classes/:na]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_252]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_252]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_252]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_252]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:133) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:116) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:963) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:897) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:622) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846) ~[spring-webmvc-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:729) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:230) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52) ~[tomcat-embed-websocket-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:105) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:81) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:197) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-4.3.7.RELEASE.jar:4.3.7.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:192) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:165) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198) ~[tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:474) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:349) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:783) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:798) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1434) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_252]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_252]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-8.5.11.jar:8.5.11]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_252]
```

В http://localhost:8080/ мы видим:

![](https://habrastorage.org/webt/8e/yh/54/8eyh54kavbseysf8p0scvecd3bq.png)

Как выглядят exception в Sentry

![](https://habrastorage.org/webt/nv/pn/vx/nvpnvxfbmphntecue6r5kfwyxds.png)

![](https://habrastorage.org/webt/ox/d6/ey/oxd6eygzocdgc-2xhnpm7qfe37g.png)

![](https://habrastorage.org/webt/ss/ix/wg/ssixwgzrpsgi4htnxodnrlkqfae.png)

![](https://habrastorage.org/webt/nt/3c/1j/nt3c1jc98iufu3xfmagfbv2vprq.png)

![](https://habrastorage.org/webt/vw/4s/of/vw4sofgypw0dahzubw9ay0rkahg.png)

![](https://habrastorage.org/webt/8m/qh/dc/8mqhdcxo8stjzknjyhl8rdojgzw.png)

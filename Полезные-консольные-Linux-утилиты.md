В этой подборке представлены полезные малоизвестные консольные Linux утилиты. В списке не представлены Kubernetes утилиты, так как у них есть своя подборка.

Осторожно много скриншотов. Добавил до ката утилиту binenv.

[**binenv**](https://github.com/devops-works/binenv) - cамая интересная утилита для установки новых популярных программ в linux, но которых нет в пакетном менеджере.

<cut />

[**Bat**](https://github.com/sharkdp/bat) — это клон команды cat с дополнительной разметкой и подсветкой синтаксиса для большого количества языков программирования, а также интеграцией Git для отображения изменений файлов. [Подробнее на русском](https://devsimple.ru/posts/bat/).

![](https://habrastorage.org/webt/q7/8y/0_/q78y0_x3luaubqln0wnbkuq3bas.png)

[**Exa**](https://github.com/ogham/exa) – это изящный инструмент командной строки, получивший множество похвал за то, что он является современной заменой старой доброй команде ls. И это справедливо, учитывая его способность использовать разные цвета при отображении различных типов файлов, прав доступа к файлам и прав собственности, блоков и информации inode, чтобы упомянуть лишь некоторые из них. [Подробнее на русском](https://habr.com/ru/company/macloud/blog/549988/).

![](https://habrastorage.org/webt/iq/nn/oq/iqnnoqvkfevizecfmw_btjy-gxg.png)

[**Fd**](https://github.com/sharkdp/fd) - это простой, быстрый и удобный инструмент, предназначенный для более простой и быстрой работы по сравнению с командой find. [Подробнее на русском](https://blog.sedicomm.com/2019/05/11/fd-prostaya-i-bystraya-alternativa-komande-find/).

![](https://github.com/sharkdp/fd/raw/master/doc/screencast.svg)

[**Procs**](https://github.com/dalance/procs) - это современная замена ps, программы командной строки по умолчанию в Unix / Linux для получения информации о процессах. По умолчанию он обеспечивает удобный, понятный для человека (и цветной) формат вывода. [Подробнее на русском](https://zenway.ru/page/procs).

![](https://habrastorage.org/webt/yz/ki/1i/yzki1im8mvuaish4b7wnwmjd02y.png)

[**Sd**](https://github.com/chmln/sd) - это интуитивно понятный инструмент командной строки для поиска и замены, он является альтернативой sed. sd имеет более простой синтаксис для замены всех вхождений и использует удобный синтаксис регулярных выражений, который вы уже знаете из JavaScript и Python. Sd также в 2-11 раз быстрее, чем sed.

![](https://habrastorage.org/webt/zr/va/po/zrvapo_o1na0t8anwwfgm9q5gyu.png)

[**Dust**](https://github.com/bootandy/dust) - опрятная версия дефолтного du, c удобной записью памяти, цветом и отступами. 

![](https://habrastorage.org/webt/en/f2/wp/enf2wpflcl_auokns8w7xsplkuk.png)

[**Starship**](https://starship.rs/guide/) - очень приятный prompt который легко накатывается поверх zsh, fish, bash и прочего. 

Легкая настройка через Toml файл (https://github.com/toml-lang/toml) с кучей уже поддерживаемых форматов и конфигов (https://starship.rs/config/#prompt). [Подробнее на русском](https://ubunlog.com/ru/starship-prompt-personalizable-escrito-en-rust/).

![](https://habrastorage.org/webt/a9/zf/2t/a9zf2tdwle1sstmiarfvjtbriiu.gif)

[**Ripgrep**](https://github.com/BurntSushi/ripgrep) - быстрый поиск с возможностью замены по содержимому в файлах, аналог [GREP](https://www.gnu.org/savannah-checkouts/gnu/grep/), [ASK](https://beyondgrep.com/), написан на [RUST](https://www.rust-lang.org/), понимает регулярные выражения, игнорирует ресурсы указанные в .gitignore, автоматически пропускает бинарные, скрытые файлы. [Подробнее на русском](https://www.setup.pp.ua/2020/06/ripgrep-linux-macos.html).

![](https://habrastorage.org/webt/5z/vi/yn/5zviynxccn58ruxjqvy5vsqtfey.gif)

[**Ripgrep-all**](https://github.com/phiresky/ripgrep-all) - Инструмент поиска, ориентированный на строки, который позволяет вам искать по регулярному выражению во множестве типов файлов. Ripgrep-all является оберткой над [ripgrep](https://github.com/BurntSushi/ripgrep) и позволяет ему искать в pdf, docx, sqlite, jpg, субтитрах фильмов (mkv, mp4) и т. д.

![](https://habrastorage.org/webt/rk/k_/wl/rkk_wltuhvuvjlti0ncsf1cimdu.png)

[**Grex**](https://github.com/pemistahl/grex) - это инструмент командной строки и библиотека для генерации регулярных выражений из предоставленных пользователем тестовых примеров. Созданное регулярное выражение имеет синтаксис PCRE.

![](https://habrastorage.org/webt/gh/yy/dd/ghyyddxgzr2e8tm51lmmy1ej6zo.gif)

[**Fzf**](https://github.com/junegunn/fzf) - инструмент для "фильтрации" любых списковых данных в консоли. Может использоваться для фильтрации списка файлов, истории команд, процессов, hostnames, коммитов и прочего. [Подробнее на русском](http://distrland.blogspot.com/2020/03/fzf-linux.html).

![](https://habrastorage.org/webt/-3/-n/df/-3-ndfcdqc6sjta-lonay3ymb24.gif)

[**Jq**](https://github.com/stedolan/jq) - это легкий и гибкий JSON-процессор командной строки. [Подробнее на русском](https://zalinux.ru/?p=4744).

![](https://habrastorage.org/webt/6z/rm/ta/6zrmtap1fpbibmiia7dswqzjbka.gif)

[**Peco**](https://github.com/peco/peco) — инструмент, способный сильно облегчить навигацию, а также поиск. Это небольшая утилита, которая принимает на вход список строк и выводит на экран псевдографическое меню, с помощью которого можно выбрать или найти нужную строку. Далее peco отдает эту строку на выход и завершает свою работу (по сути, это консольный аналог dmenu). [Подробнее на русском](https://onstartup.ru/razrabotka/peco/).

![](https://habrastorage.org/webt/fc/6g/wv/fc6gwvvmugi8ijog-6clasijtpm.gif)

[**HTTPie**](https://github.com/httpie/httpie)  - HTTP клиент для командной строки, с поддержкой json, понятным интерфейсом, подсветкой синтаксиса и прочим. [Подробнее на русском](https://blog.sedicomm.com/2019/02/25/httpie-sovremennyj-http-klient-pohozhij-na-komandy-curl-i-wget/).

![](https://habrastorage.org/webt/bf/be/ou/bfbeouampveoiioy4eqcn3fxx80.gif)

[**xh**](https://github.com/ducaale/xh) - удобный и быстрый инструмент для отправки HTTP-запросов. Он в максимально возможной степени воплощает превосходный дизайн [HTTPie](https://github.com/httpie/httpie).

![](https://habrastorage.org/webt/eu/fp/m5/eufpm5jhzfuhm2h8kn14bcxst9a.gif)

[**Rebound**](https://github.com/shobrook/rebound) - это инструмент командной строки, который мгновенно извлекает результаты Stack Overflow при возникновении исключения. Просто используйте команду rebound для запуска вашего исполняемого файла.

![](https://habrastorage.org/webt/sh/yz/-u/shyz-ujjbgphyu_mwxjduqu3kie.gif)

[**HTTP Prompt**](https://github.com/httpie/http-prompt) – это интерактивный HTTP-клиент командной строки, созданный на основе prompt_toolkit и HTTPie с более чем 20 темами. Его основные функции включают в себя автоматическое заполнение, подсветку синтаксиса, автоматические куки, Unix-подобные конвейеры, совместимость с [HTTpie](https://github.com/httpie/httpie), http-подсказка, которая  сохраняется между сеансами и интеграцию OpenAPI / Swagger. [Подробнее на русском](https://code.tutsplus.com/ru/tutorials/httpie-a-human-friendly-curl-like-tool--cms-27310).

![](https://habrastorage.org/webt/i9/wi/cr/i9wicraqqlvweua03weu3_nw2qw.png)

[**shell2http**](https://github.com/msoap/shell2http) - Простой веб сервер для удаленного выполнения команд.

[**reachable**](https://github.com/italolelis/reachable) - инструмент, который поможет вам проверить, работает ли домен или нет.

![](https://habrastorage.org/webt/xk/tc/zq/xktczqnxh4vioixvfjb4g4-2rm8.png)

[**Lazydocker**](https://github.com/jesseduffield/lazydocker) — пользовательский интерфейс для управления докером. С его помощью больше не нужно запоминать команды, алиасы и следить за контейнерами через несколько терминальных окон. Всё это теперь собрано в одном окне. Просмотр состояния, логов, метрик; перезапуск, удаление, восстановление контейнеров и сервисов. [Подробнее на русском](https://habr.com/ru/company/flant/blog/446700/).

[**Clog-cli**](https://github.com/clog-tool/clog-cli) - утилита для создания changelogs из истории коммитов Git.

[**Gotty**](https://github.com/yudai/gotty) - программа позволяет организовывать общий доступ к приложениям командной строки через браузер. [Подробнее на русском](https://habr.com/ru/company/ruvds/blog/529836/).

![](https://habrastorage.org/webt/qn/7n/it/qn7nit1xrmvz0b_pjnqql-_6zaq.gif)

[**mosh**](https://mosh.org/) - утилита Mosh (сокращение от mobile shell), обладающая всеми преимуществами безопасности SSH, является более устойчивой в условиях плохих сетевых и мобильных соединений. Кроме того, данное приложение увеличивает способность к реагированию и снижает использование полосы пропускания. Создание подключения и авторизация в Mosh происходят через обычное соединение SSH, что значит, что для безупречной работы каких-либо механизмов безопасности на основе ключей нужно внести всего несколько дополнительных конфигураций. После проверки подлинности ключа Mosh начинает взаимодействие через зашифрованные датаграммы UDP, что делает сессию более устойчивой к изменяющимся клиентским IP-адресам и перебоям соединений, которые часто случаются при использовании мобильных устройств. [Подробнее на русском](https://habr.com/ru/post/243651).

[**ngrok**](https://ngrok.com/) - Безопасные интроспективные туннели к localhost.

[**teleconsole**](https://www.teleconsole.com/) - поделитесь своим терминалом UNIX.

[**tmate**](https://tmate.io/) - Мгновенный доступ к терминалу (tmux).

[**Lazygit**](https://github.com/jesseduffield/lazygit) — псевдографический консольный клиент для Git. Если у вас есть проблемы с восприятием основных моментов в работе с Git-репозиторием из консоли, то вы всегда можете воспользоваться графическим клиентом. Один из них - **Lazygit**, псевдографический клиент, написанный на языке **Go** с использованием библиотеки **gocui**. В официальном описании программы автор описывает, как трудно бывает понять, что и как нужно сделать в гите, если там больше одной ветви или коммита, и как хорошо при этом помогает разобраться его клиент. Думаю, что описывать все возможности программы нет смысла, так как что еще можно сказать про гит, кроме как «позволяет коммитить, мержить и так далее»?

![](https://habrastorage.org/webt/jd/_p/g8/jd_pg8mnpjejn4gkppturp83gq8.gif)

[**GNU parallel**](https://ru.wikipedia.org/wiki/GNU_parallel) — это инструмент оболочки для параллельного выполнения работ используя один или более компьютер. В качестве работы может быть единичная команда или небольшой скрипт, который должен быть запущен для каждой строки из полученного ввода. Типичным вводом является список файлов, список хостов, список пользователей, список URL, список таблиц. В качестве работы может быть команда, которая считывает по трубе (pipe). GNU parallel затем может разбить ввод на блоки и передать блоки по трубе параллельно в каждую команду. GNU parallel может заменить вам программы **xargs** и **tee**. А также не только заменить **циклы (loops)**, но и сделать их выполнение более быстрым за счёт параллельного выполнения нескольких работ. [Подробнее на русском](https://zalinux.ru/?p=2623).

[**Bottom**](https://github.com/ClementTsang/bottom) — консольное приложение для мониторинга процессов и загрузки системы. [Подробнее на русском](https://zenway.ru/page/bottom).

![](https://habrastorage.org/webt/pu/xi/yv/puxiyvdq2vcchcmec04lsdruteu.gif)

[**Bandwhich**](https://github.com/imsnif/bandwhich) — net monitor с раскладкой по процессам, который работает и на FreeBSD.

![](https://habrastorage.org/webt/jb/qf/df/jbqfdfuz3dgit1biqqni2mrak_y.gif)



[**Delta**](https://github.com/dandavison/delta) - Средство просмотра для вывода git и diff. [Подробнее на английском.](https://dev.to/cloudx/delta-a-new-git-diff-tool-to-rock-your-productivity-2773)

![](https://habrastorage.org/webt/5o/ym/9a/5oym9aolhdg1ysuwg-eflw9s0uw.png)

![](https://habrastorage.org/webt/lv/dg/cl/lvdgcllzqjqxczn6wwggfcf3oko.png)

[**mtr**](https://github.com/traviscross/mtr) — MyTraceRoute Великолепная замена traceroute и аналогам

[**gdu**](https://github.com/dundee/gdu) — Более шустрый и фичастый аналог ncdu (ncurses du), на Go. Удобнее штатного du, при разборах «куда же делось свободное место».

[**Dog**](https://github.com/ogham/dog) - это красивый DNS-клиент командной строки для поиска DNS, который работает как dig. Он имеет красочный вывод, понимает обычный синтаксис аргументов командной строки, поддерживает протоколы DNS-over-TLS и DNS-over-HTTPS и может генерировать JSON.

![](https://habrastorage.org/webt/5f/cz/q_/5fczq__privkui0qhlfwhizt5ve.png)

[**dnsmeter**](https://github.com/DNS-OARC/dnsmeter) - это инструмент для тестирования производительности сервера имен и инфраструктуры вокруг него. Он генерирует DNS-запросы и отправляет их через UDP на целевой сервер имен и считает ответы.

[**Gitleaks**](https://github.com/zricethezav/gitleaks) - это инструмент SAST для обнаружения жестко закодированных секретов, таких как пароли, ключи API и токены в репозиториях git. Gitleaks - это простое в использовании универсальное решение для поиска секретов прошлого или настоящего в вашем коде.

[**localtls**](https://github.com/Corollarium/localtls) - DNS-сервер для предоставления TLS веб-сервисам на локальных адресах

[**fx**](https://github.com/antonmedv/fx) — альтернатива jq для обработки JSON из командной строки. [Подробнее на русском](https://habr.com/ru/post/347808/).

![](https://habrastorage.org/webt/bq/ek/im/bqekim3ksvvid772ndoibrozui8.gif)

[**dnspeep**](https://github.com/jvns/dnspeep) - простая утилита, которая позволяет просмотреть DNS запросы.

![](https://habrastorage.org/webt/jc/gi/pc/jcgipctvzyzz_lp6ptflpavuogc.jpeg)

[**Dive**](https://github.com/wagoodman/dive) - инструмент для изучения образа Docker, содержимого слоев и поиска способов уменьшить размер вашего образа Docker/OCI.

![](https://habrastorage.org/webt/z1/jz/sr/z1jzsrmbxqgtokwglyzz4fhwwrw.gif)

[datanymizer](https://github.com/datanymizer/datanymizer) - Мощный анонимайзер базы данных с гибкими правилами. [Подробнее на русском](https://evrone.ru/datanymizer).

[**termshark**](https://github.com/gcla/termshark) - консольный интерфейс терминала для tshark, вдохновленный Wireshark

![](https://habrastorage.org/webt/y0/aw/vp/y0awvpo97t_ifaxyawh6aeyou9g.gif)

[**sysinfo**](https://github.com/peterbay/sysinfo) - Скрипт на основе Python для получения системной информации из Linux.

**[SSH-Attack-Stats](https://github.com/Sodium-Hydrogen/SSH-Attack-Stats)** - Простой скрипт, который будет запущен в MOTD на сервере Linux и сообщит вам статистику атак.

![](https://habrastorage.org/webt/kb/lh/by/kblhbym1hikagrw0bzmtsbjozwu.png)



[**dry**](https://github.com/moncho/dry) — менеджер для Docker, по ощущениям гораздо быстрее и отзывчивее чем «LazyDocker»

[**gh**](https://github.com/cli/cli) — утилита для работы с GitHub из консоли, например можно создать Pull Request

[**gitlab**](https://github.com/NARKOZ/gitlab/) — аналогичная утилита для работы с GitLab (неофициальная)

**watch** — запуск любой команды каждые N секунд, позволяет на раз-два сделать реалтайм мониторинг в консоли

**runnel** — автоматический запуск туннелей SSH с переподключением при обрыве соединения

Ниже утилиты и краткое описание со статьи [Sysadmin-util: полезные инструменты для системных администраторов Linux](https://xakinfo.ru/network_administration/sysadmin-tools/). Подробнее об этих утилитах вы найдете в этой статье.

**Ago** - Данный инструмент выводит в удобочитаемом формате информацию, как давно файл или каталог были изменены.

Cronic – инструмент запускает команду тихо, пока не завершится неудачей, т.е. он запускает команду и скрывает STDOUT и STDERR, если она успешно завершается. Это полезно для заданий cron.

**cidr2ip** - Он преобразует блоки CIDR в составляющие их IP-адреса.

**collapse** - Инструмент collapse удаляет пустые строки и строки, содержащие пробелы, из заданных файлов.

**dupes** - Инструмент dupes сообщит о идентичных файлах. Это поможет вам найти дубликаты файлов, которые содержат то же самое содержимое. Утилита сравнивает у файлов хэш SHA1 .

**empty-dir** - Этот инструмент проверит, является ли данный каталог пустым или нет.

**expand-ipv6** - Этот инструмент расширяет указанные сокращенные / сжатые адреса IPv6 до их полной формы. Это может быть полезно при настройке DNS.

multi-ping - Это многопротокольная оболочка ping. Он используется для проверки подключения удаленного хоста независимо от того, является ли он хостом IPv6 или IPv4. Значение: если пульт использует IPv4, он вызывает команду ping для проверки возможности подключения. Если удаленный хост использует IPv6, он вызовет команду «ping6».

**pyhttpd** - Это простой  HTTP-сервер на Python, который позволяет мгновенно настроить базовый web-сервер.

**randpass** - Утилита randpass используется для генерации случайного пароля из командной строки.

**since** - Он показывает любой новый контент с момента последнего чтения файла. Это полезно для отслеживания файлов журнала.

**ssl-expiry-date** - Отображает дату истечения срока действия сертификата SSL данного домена или хоста.

**timeout** - Это позволяет пользователю выполнить команду для определенного интервала и уничтожить ее.

**until-error & until-success** –  повторять команды до тех пор, пока не произойдет сбой или успешно выполнится

**when-down & when-up** – ждет, пока хост не упадет/ поднимется

**mysql-slave-check** – выяснить, является текущий хост – ведомым или нет

**which-shell** – определить оболочку, под которой мы работаем, и т. д.

Утилиты из пакета Moreutils:

**sponge** - «губка» для стандартного ввода. [Подробнее на русском](https://habr.com/ru/post/178141/)

**Утилиты ниже взял из канала** https://t.me/SysadminNotes

[**ssh-config**](https://github.com/dbrady/ssh-config) - простая штуковина, позволяющая работать с конфигом SSH.

[**ssh-tools**](https://github.com/vaporup/ssh-tools) - набор из нескольких утилит. Проверяем удалённые хосты, получаем информацию о них и т. п.

[**assh**](https://github.com/moul/assh) - серьёзный инструмент, который позволяет иначе взглянуть на привычную работу с SSH. Тут вам и регулярки, и возможность использования шаблонов, хуки, работа с конфигом SSH клиента. Причём, lib-ssh запускает assh через ProxyCommand, а значит что весь этот функционал мы можем использовать и с scp, и c rcync'ом, и с git'ом, например.

**[domain-check-2](https://github.com/click0/domain-check-2)** - Простой скрипт для проверки срока истечения важных для нас доменных имён. [Подробнее на русском](https://sysadmin.pm/need-renew-domains/) 

[**pingtop**]() - интересная *top* утилита, с помощью которой можно пинговать несколько сайтов одновременно. [Подробнее на русском](https://sysadmin.pm/pingtop/)  

[**red**](https://github.com/antonmedv/red) - весьма интересный анализатор логов. Пайпим ему json выхлоп и получаем статистику по зафиксированным там событиям.

![](https://habrastorage.org/webt/w5/sg/s0/w5sgs0tufc1qz9vpreevkaa75o4.gif)

[**Watchman**](https://github.com/facebook/watchman) - Позволяет отслеживать состояние файлов, и по мере их изменения триггерить нужные действия, если потребуется (запускаем что-то, собираем что-то, уведомляем и т. п.).

[**localstack**](https://github.com/localstack/localstack) - протестировать что-то предназначенное для AWS локально

[**txeh**](https://github.com/txn2/txeh) - Библиотека Go и утилита CLI для управления /etc/hosts.

[**mkchain**](https://github.com/trimstray/mkchain) - Утилита, с помощью которой можно составить цепочку сертификатов, из присланных от центра сертификации файлов.

 [**Asn**](https://github.com/nitefood/asn) - удобная утилита для отображения информации об IP и домене.

![](https://habrastorage.org/webt/-n/zs/em/-nzsemwztzcflmrnu0qcmdlofog.png)

[**tlog**](https://github.com/Scribery/tlog) - Инструмент для записи / воспроизведения терминального ввода-вывода и сессий. [Подробнее на английском](https://linoxide.com/tlog-tool-record-play-terminal-io-sessions/)

[**frp**](https://github.com/fatedier/frp) - Быстрый обратный прокси-сервер, который поможет вам открыть доступ в Интернет к локальному серверу за NAT или брандмауэром.

[**tunnelto**](https://github.com/agrinman/tunnelto) - Разместите свой локальный веб-сервер в Интернете с помощью общедоступного URL-адреса.

[**pd.sh**](https://github.com/ketancmaheshwari/pd) - интересная штука для дебага пйплайнов. Подсовываем скрипту наш набор команд, и получаем информацию о том, как пайплайн отрабатывает и в каком месте возникает ошибка, если такое происходит.

[**logtop**](https://github.com/JulienPalard/logtop) - утилита, которая выводит top IP адресов, анализируя логи веб-сервера. Это для тех, кому не очень хочется однострочник на bash писать c awk, sort и uniq.

[**Choose**](https://github.com/theryangeary/choose) - аналог cut и awk '{print $N}'. Авторы, к слову, утверждают что работает штука значительно быстрее перечисленных аналогов.

![](https://habrastorage.org/webt/kj/_a/_l/kj_a_lmp6wyxfu7a8_yhklavyqe.png)

[**whatfiles**](https://github.com/spieglt/whatfiles)  - простая и удобная утилита, которая покажет, с какими файлами работает тот или иной процесс в системе. Этакий упрощённый вариант strace'а.

[**topngx**](https://github.com/gsquire/topngx) - этакий top для Nginx. Утилита анализирует логи и выдаёт администратору итоговую статистику. По сути, это аналог другой утилиты - ngxtop, правда эта уже давно не поддерживается.

[**SSHHeatmap**](https://github.com/meesaltena/SSHHeatmap) - Создает тепловую карту IP-адресов, которые сделали неудачные попытки входа в систему по SSH.

[**desed**](https://github.com/SoptikHa2/desed/) - Отладчик для Sed: отлаживайте сценарии sed, не выходя из вашего терминала.

[**3mux**](https://github.com/aaronjanse/3mux) - Терминальный мультиплексор, вдохновленный i3

**[frontail](https://github.com/mthenw/frontail)** — утилита для трансляции логов по http\https. Удобно использовать в случаях, когда нужно быстро, в браузере, показать содержимое какого-либо лога и его изменения в реальном времени. [Подробнее на русском](https://sysadmin.pm/frontail/) 

[**critic.sh**](https://github.com/Checksum/critic.sh) - Суровое тестирование bash скриптов. Выглядит не менее интересным даже в сравнении с shellcheck

[**jql**](https://github.com/cube2222/jql) - Вместо jq можно попробовать jql для работы с JSON.

**asciicast2gif** - утилита для конвертации ascii кастов, в обычную gif анимацию. [Подробнее на русском](https://sysadmin.pm/asciicast2gif/)

[**Nginx-builder**](https://github.com/TinkoffCreditSystems/Nginx-builder) - инструмент, который автоматизирует сборку Nginx с нужными модулями. Описываем всё в yaml, запускаем сборку и получаем готовый rpm или deb пакет

**[linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester)** - Быстрый аудит сервера на возможность применения эксплойтов

![](https://habrastorage.org/webt/f4/ok/qh/f4okqhjwfm2vpj09pmwny8plowq.jpeg)

[**linux-exploit-suggester-2**](https://github.com/jondonas/linux-exploit-suggester-2) - Подсказка эксплойтов ядра Linux нового поколения

![](https://habrastorage.org/webt/3i/-u/qw/3i-uqwfubldia6zeo7h16n7zq_4.jpeg)

[**ssl-cert-check**](https://github.com/Matty9191/ssl-cert-check) - делаем простой мониторинг SSL с уведомлениями на email о необходимости продления сертификата. [Подробнее на русском](https://sysadmin.pm/ssl-cert-check/)

**[jl](https://github.com/koenbollen/jl)** - Удобная утилита для парсинга\чтения JSON логов

**[Wondershaper](https://github.com/magnific0/wondershaper.git)** — утилита для настройки ограничений скорости сетевого интерфейса на сервере. [Подробнее на русском](https://sysadmin.pm/wondershaper/)

[**mkcert**](https://github.com/FiloSottile/mkcert) - удобная штука, с помощью которой можно у себя на локалхосте для часто употребляемых тестовых имён выписать валидный сертификат.

![](https://habrastorage.org/webt/rq/wd/yx/rqwdyx_zgc9yqzn7nbm_arfx6cm.png)

[**https-portal**](https://github.com/SteveLTN/https-portal) - Полностью автоматизированный сервер HTTPS на базе Nginx, Let's Encrypt и Docker.

**httpsat** — утилита, которая позволяет получить статистику доступа к нужному сайту. Скрипт работает с cURL’ом, и поддерживает многие доступные в нём опции при соединении. [Подробнее на русском](https://sysadmin.pm/httpstat/) 

[GeoLogonalyzer](https://github.com/fireeye/GeoLogonalyzer) - И вот ещё занятный анализатор логов, который на основе asn, гео, и некоторых сопутствующих данных выявляет аномалии и сообщает о них. Например, если пользователь много раз заходил с одного IP адреса, но внезапно зашёл с другого (причём из другой подсети вовсе, либо из другой географической точки), анализатор сформирует отчёт об этом.

**[graphpath](https://github.com/ocochard/graphpath)** - Утилита для быстрого построения ASCII схемы сети прямо в терминале.

![](https://habrastorage.org/webt/bc/lg/mb/bclgmbzjcnmudnmag43825hxjq0.jpeg)

**Sosreport** — утилита для сбора информации о текущем состоянии системы. Удобно использовать самому в рамка первичного осмотра, удобно попросить клиента выполнить команду и прислать отчёт, готовый для изучения. [Подробнее на русском](https://sysadmin.pm/sosreport-xsos/)

![](https://habrastorage.org/webt/sd/pi/-h/sdpi-hcgucarayut4kl8izphhk0.png)

**Fatrace** (file access trace) — простая и удобная в использовании утилита для мониторинга текущей активности в системе. Отлично подойдёт для тех случаев, когда strace или blktrace оказываются избыточны. [Подробнее на русском](https://sysadmin.pm/fatrace/)

![](https://habrastorage.org/webt/fv/0k/wn/fv0kwn_xa1cleeaxeafmhemdmuk.png)

**httping** — простая утилита, с помощью которой можно «пинговать» сайт по определённому адресу и получать общую (а при необходимости подробную) статистику по ответу и доступности. [Подробнее на русском](https://sysadmin.pm/httping/)

![](https://habrastorage.org/webt/3o/yy/ie/3oyyie21wikpzpqyv3uigc0auss.png)

**Bashdb** — утилита-дебаггер для bash скриптов. Дебаг скриптов задача не частая, но если она стоит, то на bashdb имеет смысл обратить внимание. [Подробнее на русском](https://sysadmin.pm/bashdb/)

![](https://habrastorage.org/webt/uy/dm/2r/uydm2r9nzzgdp2u1uf9dnut0gl4.png)

**Goaccess** — функциональный анализатор логов, который может работать с ними в реальном времени, визуализировать информацию и отдавать её в разных форматах. Доступен для установки на большинстве популярных дистрибутивов прямо из репозиториев. [Подробнее на русском](https://sysadmin.pm/goaccess/)

![](https://habrastorage.org/webt/iq/rn/0h/iqrn0haij0su3fl1lf7cd_jijp8.png)

**[testssl.sh](https://github.com/drwetter/testssl.sh)** — это скрипт, который позволяет протестировать SSL сервис и вывести подробную статистику по нему. Это как SSLlabs, только имеется возможность проведения проверки разных сервисов. Кроме того, для скрипта доступен простой веб-интерфейс. [Подробнее на русском](https://sysadmin.pm/testssl/)

![](https://habrastorage.org/webt/cn/ex/zf/cnexzfx9n3a9ejxbozyiiif7nkk.png)

**[iprange](https://github.com/firehol/iprange/)** - Обнаружилась очень удобная утилита для работы с подсетями и IP адресами. Умеет вычислять нужные подсети из разных диапазонов, что порой бывает необходимо, например, при работе с ipset и фаерволом на сервере. 

[**container-diff**](https://github.com/GoogleContainerTools/container-diff) - утилита для сравнения образов контейнеров. Сontainer-diff умеет анализировать и учитывать при сравнении историю образа, файловую систему, пакеты apt, pip и npm. Выглядит очень удобно для отслеживания изменений в рамках одного контейнера, либо для сравнения двух разных образов.

**https://gist.github.com/agarzon/5554490** - Скрипт для проверки присутствия IP в различных DNSBL

[**Prochunter**](https://gitlab.com/nowayout/prochunter.git) — хорошая утилита для поиска скрытых процессов, запущенных в системе. [Подробнее на русском](https://sysadmin.pm/prochunter/)

[**netutils-linux**](https://github.com/strizhechenko/netutils-linux) - это набор утилит для мониторинга и тюнинга сети. [Подробнее на русском.](https://habr.com/ru/post/331720/)

![](https://habrastorage.org/web/2ae/dda/a7f/2aeddaa7fffb494dbfb0a2ecb40018ba.gif)

**Sysdig** — удобный и функциональный инструмент, дающий администратору широкие возможности для сбора информации о работающей системе. [Подробнее на русском](https://sysadmin.pm/sysdig/)

**ioping** — очень простая, но при этом удобная утилита для быстрой проверки отзывчивости дисковой подсистемы. В CentOS ставится из репозитория EPEL. [Подробнее на русском](https://sysadmin.pm/ioping/)

[**ctop**](https://github.com/bcicen/ctop) - полезная *top утилита для мониторинга работы контейнеров.

![](https://habrastorage.org/webt/mc/fk/bs/mcfkbsi_a28_lrlnoy52hynnyts.gif)

**[hstr](https://github.com/dvorka/hstr)** - от англ. history, инструмент для просмотра истории команд. 

![](https://habrastorage.org/webt/a9/0h/iq/a90hiqlc0tnmwbpyuztmgy811si.gif)

**pastebinit** - инструмент командной строки в Deb-системах для отправки данных на "pastebin", сайт, который позволяет пользователям размещать отрывки текста для публичного просмотра. [Подробнее на русском](https://habr.com/ru/post/82389/)

**fpaste** - инструмент командной строки в RedHat-системах для отправки данных на http://fpaste.org/, сайт, который позволяет пользователям размещать отрывки текста для публичного просмотра. 

[**ShellCheck**](https://github.com/koalaman/shellcheck) - инструмент статического анализа shell скриптов. [Подробнее на русском](https://habr.com/ru/company/ruvds/blog/342580/)

[Eternal Terminal](https://github.com/MisterTea/EternalTerminal) - удаленная оболочка, которая автоматически переподключается. [Подробнее на русском](https://blog.desdelinux.net/ru/eternal-terminal-un-shell-remoto-que-se-vuelve-a-conectar-automaticamente/)



P.S. Пишите утилиты, которые стоит добавить в список. 

P.S2. На написание этого поста навеял телеграм канал https://t.me/SysadminNotes где публикуются подобные интересные утилиты

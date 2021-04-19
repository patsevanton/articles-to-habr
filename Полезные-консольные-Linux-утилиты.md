В этой подборке представлены полезные малоизвестные консольные Linux утилиты. В списке не представлены Kubernetes утилиты, так как у них есть своя подборка.

Осторожно много скриншотов.
<cut />
[Bat](https://github.com/sharkdp/bat) — это клон команды cat с дополнительной разметкой и подсветкой синтаксиса для большого количества языков программирования, а также интеграцией Git для отображения изменений файлов. [Подробнее на русском](https://devsimple.ru/posts/bat/).

![](https://habrastorage.org/webt/q7/8y/0_/q78y0_x3luaubqln0wnbkuq3bas.png)

[Exa](https://github.com/ogham/exa) – это изящный инструмент командной строки, получивший множество похвал за то, что он является современной заменой старой доброй команде ls. И это справедливо, учитывая его способность использовать разные цвета при отображении различных типов файлов, прав доступа к файлам и прав собственности, блоков и информации inode, чтобы упомянуть лишь некоторые из них. [Подробнее на русском](https://habr.com/ru/company/macloud/blog/549988/).

![](https://habrastorage.org/webt/iq/nn/oq/iqnnoqvkfevizecfmw_btjy-gxg.png)

[Fd](https://github.com/sharkdp/fd) - это простой, быстрый и удобный инструмент, предназначенный для более простой и быстрой работы по сравнению с командой find. [Подробнее на русском](https://blog.sedicomm.com/2019/05/11/fd-prostaya-i-bystraya-alternativa-komande-find/).

![](https://github.com/sharkdp/fd/raw/master/doc/screencast.svg)

[Procs](https://github.com/dalance/procs) - это современная замена ps, программы командной строки по умолчанию в Unix / Linux для получения информации о процессах. По умолчанию он обеспечивает удобный, понятный для человека (и цветной) формат вывода. [Подробнее на русском](https://zenway.ru/page/procs).

![](https://habrastorage.org/webt/yz/ki/1i/yzki1im8mvuaish4b7wnwmjd02y.png)

[Sd](https://github.com/chmln/sd) - это интуитивно понятный инструмент командной строки для поиска и замены, он является альтернативой sed. sd имеет более простой синтаксис для замены всех вхождений и использует удобный синтаксис регулярных выражений, который вы уже знаете из JavaScript и Python. Sd также в 2-11 раз быстрее, чем sed.

![](https://habrastorage.org/webt/zr/va/po/zrvapo_o1na0t8anwwfgm9q5gyu.png)

[Dust](https://github.com/bootandy/dust) - опрятная версия дефолтного du, c удобной записью памяти, цветом и отступами. 

![](https://habrastorage.org/webt/en/f2/wp/enf2wpflcl_auokns8w7xsplkuk.png)

[Starship](https://starship.rs/guide/) - очень приятный prompt который легко накатывается поверх zsh, fish, bash и прочего. 

Легкая настройка через Toml файл (https://github.com/toml-lang/toml) с кучей уже поддерживаемых форматов и конфигов (https://starship.rs/config/#prompt). [Подробнее на русском](https://ubunlog.com/ru/starship-prompt-personalizable-escrito-en-rust/).

![](https://habrastorage.org/webt/a9/zf/2t/a9zf2tdwle1sstmiarfvjtbriiu.gif)

[Ripgrep](https://github.com/BurntSushi/ripgrep) - быстрый поиск с возможностью замены по содержимому в файлах, аналог [GREP](https://www.gnu.org/savannah-checkouts/gnu/grep/), [ASK](https://beyondgrep.com/), написан на [RUST](https://www.rust-lang.org/), понимает регулярные выражения, игнорирует ресурсы указанные в .gitignore, автоматически пропускает бинарные, скрытые файлы. [Подробнее на русском](https://www.setup.pp.ua/2020/06/ripgrep-linux-macos.html).

![](https://habrastorage.org/webt/5z/vi/yn/5zviynxccn58ruxjqvy5vsqtfey.gif)

[Ripgrep-all](https://github.com/phiresky/ripgrep-all) - Инструмент поиска, ориентированный на строки, который позволяет вам искать по регулярному выражению во множестве типов файлов. Ripgrep-all является оберткой над [ripgrep](https://github.com/BurntSushi/ripgrep) и позволяет ему искать в pdf, docx, sqlite, jpg, субтитрах фильмов (mkv, mp4) и т. д.

![](https://habrastorage.org/webt/rk/k_/wl/rkk_wltuhvuvjlti0ncsf1cimdu.png)

[Grex](https://github.com/pemistahl/grex) - это инструмент командной строки и библиотека для генерации регулярных выражений из предоставленных пользователем тестовых примеров. Созданное регулярное выражение имеет синтаксис PCRE.

![](https://habrastorage.org/webt/gh/yy/dd/ghyyddxgzr2e8tm51lmmy1ej6zo.gif)

[Fzf](https://github.com/junegunn/fzf) - инструмент для "фильтрации" любых списковых данных в консоли. Может использоваться для фильтрации списка файлов, истории команд, процессов, hostnames, коммитов и прочего. [Подробнее на русском](http://distrland.blogspot.com/2020/03/fzf-linux.html).

![](https://habrastorage.org/webt/-3/-n/df/-3-ndfcdqc6sjta-lonay3ymb24.gif)

[Jq](https://github.com/stedolan/jq) - это легкий и гибкий JSON-процессор командной строки. [Подробнее на русском](https://zalinux.ru/?p=4744).

![](https://habrastorage.org/webt/6z/rm/ta/6zrmtap1fpbibmiia7dswqzjbka.gif)

[Peco](https://github.com/peco/peco) — инструмент, способный сильно облегчить навигацию, а также поиск. Это небольшая утилита, которая принимает на вход список строк и выводит на экран псевдографическое меню, с помощью которого можно выбрать или найти нужную строку. Далее peco отдает эту строку на выход и завершает свою работу (по сути, это консольный аналог dmenu). [Подробнее на русском](https://onstartup.ru/razrabotka/peco/).

![](https://habrastorage.org/webt/fc/6g/wv/fc6gwvvmugi8ijog-6clasijtpm.gif)

[HTTPie](https://github.com/httpie/httpie)  - HTTP клиент для командной строки, с поддержкой json, понятным интерфейсом, подсветкой синтаксиса и прочим. [Подробнее на русском](https://blog.sedicomm.com/2019/02/25/httpie-sovremennyj-http-klient-pohozhij-na-komandy-curl-i-wget/).

![](https://habrastorage.org/webt/bf/be/ou/bfbeouampveoiioy4eqcn3fxx80.gif)

[xh](https://github.com/ducaale/xh) - удобный и быстрый инструмент для отправки HTTP-запросов. Он в максимально возможной степени воплощает превосходный дизайн [HTTPie](https://github.com/httpie/httpie).

![](https://habrastorage.org/webt/eu/fp/m5/eufpm5jhzfuhm2h8kn14bcxst9a.gif)

[Rebound](https://github.com/shobrook/rebound) - это инструмент командной строки, который мгновенно извлекает результаты Stack Overflow при возникновении исключения. Просто используйте команду rebound для запуска вашего исполняемого файла.

![](https://habrastorage.org/webt/sh/yz/-u/shyz-ujjbgphyu_mwxjduqu3kie.gif)

[HTTP Prompt](https://github.com/httpie/http-prompt) – это интерактивный HTTP-клиент командной строки, созданный на основе prompt_toolkit и HTTPie с более чем 20 темами. Его основные функции включают в себя автоматическое заполнение, подсветку синтаксиса, автоматические куки, Unix-подобные конвейеры, совместимость с [HTTpie](https://github.com/httpie/httpie), http-подсказка, которая  сохраняется между сеансами и интеграцию OpenAPI / Swagger. [Подробнее на русском](https://code.tutsplus.com/ru/tutorials/httpie-a-human-friendly-curl-like-tool--cms-27310).

![](https://habrastorage.org/webt/i9/wi/cr/i9wicraqqlvweua03weu3_nw2qw.png)

[shell2http](https://github.com/msoap/shell2http) - Простой веб сервер для удаленного выполнения команд.

[reachable](https://github.com/italolelis/reachable) - инструмент, который поможет вам проверить, работает ли домен или нет.

![](https://habrastorage.org/webt/xk/tc/zq/xktczqnxh4vioixvfjb4g4-2rm8.png)

[Lazydocker](https://github.com/jesseduffield/lazydocker) — пользовательский интерфейс для управления докером. С его помощью больше не нужно запоминать команды, алиасы и следить за контейнерами через несколько терминальных окон. Всё это теперь собрано в одном окне. Просмотр состояния, логов, метрик; перезапуск, удаление, восстановление контейнеров и сервисов. [Подробнее на русском](https://habr.com/ru/company/flant/blog/446700/).

[Clog-cli](https://github.com/clog-tool/clog-cli) - утилита для создания changelogs из истории коммитов Git.

[Gotty](https://github.com/yudai/gotty) - программа позволяет организовывать общий доступ к приложениям командной строки через браузер. [Подробнее на русском](https://habr.com/ru/company/ruvds/blog/529836/).

![](https://habrastorage.org/webt/qn/7n/it/qn7nit1xrmvz0b_pjnqql-_6zaq.gif)

[mosh](https://mosh.org/) - утилита Mosh (сокращение от mobile shell), обладающая всеми преимуществами безопасности SSH, является более устойчивой в условиях плохих сетевых и мобильных соединений. Кроме того, данное приложение увеличивает способность к реагированию и снижает использование полосы пропускания. Создание подключения и авторизация в Mosh происходят через обычное соединение SSH, что значит, что для безупречной работы каких-либо механизмов безопасности на основе ключей нужно внести всего несколько дополнительных конфигураций. После проверки подлинности ключа Mosh начинает взаимодействие через зашифрованные датаграммы UDP, что делает сессию более устойчивой к изменяющимся клиентским IP-адресам и перебоям соединений, которые часто случаются при использовании мобильных устройств. [Подробнее на русском](https://habr.com/ru/post/243651).

[ngrok](https://ngrok.com/) - Безопасные интроспективные туннели к localhost.

[teleconsole](https://www.teleconsole.com/) - поделитесь своим терминалом UNIX.

[tmate](https://tmate.io/) - Мгновенный доступ к терминалу (tmux).

[Lazygit](https://github.com/jesseduffield/lazygit) — псевдографический консольный клиент для Git. Если у вас есть проблемы с восприятием основных моментов в работе с Git-репозиторием из консоли, то вы всегда можете воспользоваться графическим клиентом. Один из них - **Lazygit**, псевдографический клиент, написанный на языке **Go** с использованием библиотеки **gocui**. В официальном описании программы автор описывает, как трудно бывает понять, что и как нужно сделать в гите, если там больше одной ветви или коммита, и как хорошо при этом помогает разобраться его клиент. Думаю, что описывать все возможности программы нет смысла, так как что еще можно сказать про гит, кроме как «позволяет коммитить, мержить и так далее»?

![](https://habrastorage.org/webt/jd/_p/g8/jd_pg8mnpjejn4gkppturp83gq8.gif)

[GNU parallel](https://ru.wikipedia.org/wiki/GNU_parallel) — это инструмент оболочки для параллельного выполнения работ используя один или более компьютер. В качестве работы может быть единичная команда или небольшой скрипт, который должен быть запущен для каждой строки из полученного ввода. Типичным вводом является список файлов, список хостов, список пользователей, список URL, список таблиц. В качестве работы может быть команда, которая считывает по трубе (pipe). GNU parallel затем может разбить ввод на блоки и передать блоки по трубе параллельно в каждую команду. GNU parallel может заменить вам программы **xargs** и **tee**. А также не только заменить **циклы (loops)**, но и сделать их выполнение более быстрым за счёт параллельного выполнения нескольких работ. [Подробнее на русском](https://zalinux.ru/?p=2623).

[Bottom](https://github.com/ClementTsang/bottom) — консольное приложение для мониторинга процессов и загрузки системы. [Подробнее на русском](https://zenway.ru/page/bottom).

![](https://habrastorage.org/webt/pu/xi/yv/puxiyvdq2vcchcmec04lsdruteu.gif)

[Bandwhich](https://github.com/imsnif/bandwhich) — net monitor с раскладкой по процессам, который работает и на FreeBSD.

![](https://habrastorage.org/webt/jb/qf/df/jbqfdfuz3dgit1biqqni2mrak_y.gif)

P.S. Пишите утилиты, которые стоит добавить в список.

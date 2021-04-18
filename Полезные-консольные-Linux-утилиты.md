Полезные консольные Linux утилиты

[Bat](https://github.com/sharkdp/bat) — это клон команды cat с дополнительной разметкой и подсветкой синтаксиса для большого количества языков программирования, а также интеграцией Git для отображения изменений файлов.

![](https://camo.githubusercontent.com/7b7c397acc5b91b4c4cf7756015185fe3c5f700f70d256a212de51294a0cf673/68747470733a2f2f696d6775722e636f6d2f724773646e44652e706e67)

[Exa](https://github.com/ogham/exa) – это изящный инструмент командной строки, получивший множество похвал за то, что он является современной заменой старой доброй команде ls. И это справедливо, учитывая его способность использовать разные цвета при отображении различных типов файлов, прав доступа к файлам и прав собственности, блоков и информации inode, чтобы упомянуть лишь некоторые из них.

![](https://github.com/ogham/exa/raw/master/screenshots.png)

[Fd](https://github.com/sharkdp/fd) - это простой, быстрый и удобный инструмент, предназначенный для более простой и быстрой работы по сравнению с командой find. 

![](https://github.com/sharkdp/fd/raw/master/doc/screencast.svg)

[Procs](https://github.com/dalance/procs) - это современная замена ps, программы командной строки по умолчанию в Unix / Linux для получения информации о процессах. По умолчанию он обеспечивает удобный, понятный для человека (и цветной) формат вывода.

![](https://user-images.githubusercontent.com/4331004/55446704-ab43a480-55fb-11e9-81dc-e3ac1a1e2507.png)

[Sd](https://github.com/chmln/sd) - это интуитивно понятный инструмент командной строки для поиска и замены, он является альтернативой sed. sd имеет более простой синтаксис для замены всех вхождений и использует удобный синтаксис регулярных выражений, который вы уже знаете из JavaScript и Python. Sd также в 2-11 раз быстрее, чем sed.

![](https://user-images.githubusercontent.com/200613/90223698-d6a0eb00-de0e-11ea-85e7-7bf590794ac0.png)

[Dust](https://github.com/bootandy/dust) - опрятная версия дефолтного du, c удобной записью памяти, цветом и отступами. 

![](https://github.com/bootandy/dust/raw/master/media/snap.png)

[Starship](https://starship.rs/guide/) - очень приятный prompt который легко накатывается поверх zsh, fish, bash и прочего. 
Легкая настройка через Toml файл (https://github.com/toml-lang/toml) с кучей уже поддерживаемых форматов и конфигов (https://starship.rs/config/#prompt). 

![](https://raw.githubusercontent.com/starship/starship/master/media/demo.gif)

[Ripgrep](https://github.com/BurntSushi/ripgrep) - быстрый поиск с возможностью замены по содержимому в файлах, аналог [GREP](https://www.gnu.org/savannah-checkouts/gnu/grep/), [ASK](https://beyondgrep.com/), написан на [RUST](https://www.rust-lang.org/), понимает регулярные выражения, игнорирует ресурсы указанные в .gitignore, автоматически пропускает бинарные, скрытые файлы.

![](https://res.cloudinary.com/practicaldev/image/fetch/s--zRcFJ8zW--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://www.santoshsrinivas.com/content/images/2017/09/demo-ripgrep.gif)

[Ripgrep-all](https://github.com/phiresky/ripgrep-all) - Инструмент поиска, ориентированный на строки, который позволяет вам искать по регулярному выражению во множестве типов файлов. Ripgrep-all является оберткой над [ripgrep](https://github.com/BurntSushi/ripgrep) и позволяет ему искать в pdf, docx, sqlite, jpg, субтитрах фильмов (mkv, mp4) и т. д.

![](https://github.com/phiresky/ripgrep-all/raw/master/doc/demodir.png)

[Grex](https://github.com/pemistahl/grex) - это инструмент командной строки и библиотека для генерации регулярных выражений из предоставленных пользователем тестовых примеров.

![](https://github.com/pemistahl/grex/raw/main/demo.gif)

[Fzf](https://github.com/junegunn/fzf) - инструмент для "фильтрации" любых списковых данных в консоли. 
Может использоваться для фильтрации списка файлов, истории команд, процессов, hostnames, коммитов и прочего.

![](https://res.cloudinary.com/practicaldev/image/fetch/s--v3bl4Az1--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://cdn-media-1.freecodecamp.org/images/YrD36K2ENGCVEPlslYvMzSwTG1xTsZRiIlHS)

[Jq](https://github.com/stedolan/jq) - это легкий и гибкий JSON-процессор командной строки.

![](https://res.cloudinary.com/practicaldev/image/fetch/s--2GvyaJmF--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://raw.githubusercontent.com/rike422/kirinuki-cli/master/assets/sample.gif)

[Peco](https://github.com/peco/peco) — инструмент, способный сильно облегчить навигацию, а также поиск. Это небольшая утилита, которая принимает на вход список строк и выводит на экран псевдографическое меню, с помощью которого можно выбрать или найти нужную строку. Далее peco отдает эту строку на выход и завершает свою работу (по сути, это консольный аналог dmenu).

![](https://camo.githubusercontent.com/00ac9dd8fa29725a926bc8eef15f0108ecd5c9f3b355e2a921df4deaa4b16628/687474703a2f2f7065636f2e6769746875622e696f2f696d616765732f7065636f2d64656d6f2d72616e67652d6d6f64652e676966)

[HTTPie](https://github.com/httpie/httpie)  - HTTP клиент для командной строки, с поддержкой json, понятным интерфейсом, подсветкой синтаксиса и прочим.

![](https://res.cloudinary.com/practicaldev/image/fetch/s--DHOT_1Db--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://httpie.io/static/img/httpie.gif%3Fv%3D70bc5a5b7fdf2b4982ed18b364c32b11)

[Rebound](https://github.com/shobrook/rebound) - это инструмент командной строки, который мгновенно извлекает результаты Stack Overflow при возникновении исключения. Просто используйте команду rebound для запуска вашего исполняемого файла.

![](https://github.com/shobrook/rebound/raw/master/docs/demo.gif)

[HTTP Prompt](https://github.com/httpie/http-prompt) – это интерактивный HTTP-клиент командной строки, созданный на основе prompt_toolkit и HTTPie с более чем 20 темами. Его основные функции включают в себя автоматическое заполнение, подсветку синтаксиса, автоматические куки, Unix-подобные конвейеры, совместимость с [HTTpie](https://github.com/httpie/httpie), http-подсказка, которая  сохраняется между сеансами и интеграцию OpenAPI / Swagger.

![](https://camo.githubusercontent.com/b526c6f37cbae7a3d44526a195d68fa3e524e691ac40ae9c004e69485c8c14c0/68747470733a2f2f61736369696e656d612e6f72672f612f39363631332e706e67)



[shell2http](https://github.com/msoap/shell2http) - Простой веб сервер для удаленного выполнения команд.

[reachable](https://github.com/italolelis/reachable) - инструмент, который поможет вам проверить, работает ли домен или нет.

![](https://camo.githubusercontent.com/900e0b9b4af73c486b035b0e821a9f59fcd4a17ee2597cb11dc2cf816132525c/68747470733a2f2f61736369696e656d612e6f72672f612f4c4a536f6f5661686f696f707039567836544854456e776e502e706e67)

[Lazydocker](https://github.com/jesseduffield/lazydocker) — пользовательский интерфейс для управления докером. С его помощью больше не нужно запоминать команды, алиасы и следить за контейнерами через несколько терминальных окон. Всё это теперь собрано в одном окне. Просмотр состояния, логов, метрик; перезапуск, удаление, восстановление контейнеров и сервисов.

[Clog-cli](https://github.com/clog-tool/clog-cli) - утилита для создания changelogs из истории коммитов Git.

[Gotty](https://github.com/yudai/gotty) - программа позволяет организовывать общий доступ к приложениям командной строки через браузер. 

![](https://raw.githubusercontent.com/yudai/gotty/master/screenshot.gif)

[mosh](https://mosh.org/) - удаленный SSH-клиент, который позволяет роуминг с прерывистой связью.
[ngrok](https://ngrok.com/) - Безопасные интроспективные туннели к localhost.
[teleconsole](https://www.teleconsole.com/) - поделитесь своим терминалом UNIX.
[tmate](https://tmate.io/) - Мгновенный доступ к терминалу (tmux).


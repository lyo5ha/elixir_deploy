
# Table of Contents

1.  [Настраиваем фаервол](#org76ef55a)
2.  [Добавляем пользователя для деплоя](#orge4d9155)
3.  [Устанавливаем NGNIX](#org0bafadc)
    1.  [Установка, запуск Ngnix](#org5c18c50)
    2.  [Конфигурация Ngnix](#org9671d79)
4.  [SSL-сертификат](#orgfeb88f8)
5.  [Postgresql](#org2a090fa)
6.  [Установка elixir, erlang и node.js](#org0ea63b8)
    1.  [Mенеджер версий asdf](#org5843370)
    2.  [Установка Эрланга](#org846bc83)
    3.  [Установка Эликсира](#org4b73d12)
    4.  [Установка Node.js](#orgf1d20cb)
7.  [Конфигурация проекта](#org43bbacc)
    1.  [config/prod.exs](#org43b3cc6)
    2.  [Хранение prod.secret.exs](#orgda796d8)
    3.  [Distillery, Edeliver](#org50597e4)
        1.  [Добавляем в зависимости в `mix.exs`](#org7737b1b)
        2.  [Создаем релиз(конфиг edeliver)](#orgb2ad85f)
8.  [Управление релизами](#org5eab61c)
    1.  [Ngnix и reverse proxy](#org2d20e2a)
    2.  [Деплой, администрирование релизов](#org2613c6b)
        1.  [Команды деплоя](#orga1fa803)
        2.  [Логи](#orge955312)
        3.  [Компилирование ассетов при деплое](#orge576074)
    3.  [.edeliver/config - финальный вид](#org2e39329)
9.  [Возможные проблемы](#org2da379c)


<a id="org76ef55a"></a>

# Настраиваем фаервол

Осторожно! Можно самому себе забокировать доступ на сервер!
Дальнейшее подразумевает, что вы зашли(уже зашли) по ssh ключу и он
добавлен(уже лежит) в `/home/ubuntu/.ssh/authorized_keys`

    
    $ sudo ufw app list
    
    # output:
    
    Available applications:
    OpenSSH
    
    $ sudo ufw allow OpenSSH
    
    $ sudo ufw enable
    
    $ sudo ufw status
    
    # output:
    
    Status: active
    
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere
    OpenSSH (v6)               ALLOW       Anywhere (v6)
    
    # добавить порт для Феникса
    
    $ sudo ufw allow 4000


<a id="orge4d9155"></a>

# Добавляем пользователя для деплоя

И даем ему права.

    
    $ adduser deploy
    $ sudo usermod -a -G sudo deploy

Не забудьте пароль записать/запомнить.

Делаем возможность заходить под `deploy` через ssh
и копируем права.
(внимательно с пробелами!)

    
    $ sudo rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy

Проверить:

-   зайти в новом терминале `ssh deploy@<ip-адрес сервера>`

Нам необходимо будет заходить таким образом: `ssh <домен.com>`,
для этого добавить в `~/.ssh/config` на локальной машине:

    Host example.com 
        HostName example.com
        User deploy
        IdentityFile ~/.ssh/<файл_с_ключом>


<a id="org0bafadc"></a>

# Устанавливаем NGNIX


<a id="org5c18c50"></a>

## Установка, запуск Ngnix

Заходим на сервер через ssh под юзером `deploy`

    
    $ sudo apt update
    $ sudo apt install nginx

Апдэйтим файервол:

    
    $ sudo ufw allow 'Nginx HTTP'
    $ sudo ufw status
    
    # output:
    
    Status: active
    
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere                  
    4000                       ALLOW       Anywhere                  
    Nginx HTTP                 ALLOW       Anywhere                  
    OpenSSH (v6)               ALLOW       Anywhere (v6)             
    4000 (v6)                  ALLOW       Anywhere (v6)             
    Nginx HTTP (v6)            ALLOW       Anywhere (v6)        

После установки nginx должен сам запуститься и работать.
Проверить:

    
    $ systemctl status nginx
    
    # output:
    
    ● nginx.service - A high performance web server and a reverse proxy server
    Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
    Active: active (running) since Fri 2018-04-20 16:08:19 UTC; 3 days ago
    Docs: man:nginx(8)
    Main PID: 2369 (nginx)
    Tasks: 2 (limit: 1153)
    CGroup: /system.slice/nginx.service
    ├─2369 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
    └─2380 nginx: worker process

Зайти по ip через браузер. (`http://your_server_ip`) Должа быть старничка `Welcome to nginx!`
Узнать ip можно: `curl -4 icanhazip.com`

Ngnix автозапускается при перезагрузке сервера. Управление:

    
    $ sudo systemctl stop nginx
    $ sudo systemctl start nginx
    $ sudo systemctl restart nginx
    
    # при изменеиии конфигов перезапускать не обязательно:
    
    $ sudo systemctl reload nginx


<a id="org9671d79"></a>

## Конфигурация Ngnix

    
     $ sudo mkdir -p /var/www/rbk.pay.amarkets.net/html
     $ sudo chown -R $USER:$USER /var/www/rbk.pay.amarkets.net/html
     $ sudo chmod -R 755 /var/www/rbk.pay.amarkets.net
    
     # отредактировать home-страницу для нового блока
    
     $ vim /var/www/rbk.pay.amarkets.net/html/index.html
    
     # добавить что-то типа:
     <html>
       <head>
         <title>Welcome to Example.com!</title>
     </head>
     <body>
     <h1>Success!  The example.com server block is working!</h1>
         </body>
     </html>
    
    # завести новый конфиг-файл для нового блока
    
    $ sudo vim /etc/nginx/sites-available/rbk.pay.amarkets.net
    
    # новая конфигурация (вставить)
    
    server {
          listen 80;
          listen [::]:80;
    
          root /var/www/rbk.pay.amarkets.net/html;
          index index.html index.htm index.nginx-debian.html;
    
          server_name rbk.pay.amarkets.net;
    
          location / {
                      try_files $uri $uri/ =404;
          }
    }

    # подключить новый конфиг (пробел после .net)
    
    $ sudo ln -s /etc/nginx/sites-available/rbk.pay.amarkets.net /etc/nginx/sites-enabled/
    
    # подправить кофиг
    
    $ sudo vim /etc/nginx/nginx.conf
    
    # раскомментить строчку:
    
    ...
    http {
    ...
        server_names_hash_bucket_size 64;
    ...
    }
    ...
    
    # проверить, что конфигурация без ошибок:
    
    $ sudo nginx -t
    
    # output:
    
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    
    # перезапустить nginx:
    
    $ sudo systemctl restart nginx

Подразумевается, что у нас будет работать только `<domen_name.com>`.
Если надо, чтобы работал и `<www.domen_name.com>`, то:
надо добавить CNAME настройку в DNS-настройках сервера,
что-то типа `CNAME www.rbk.pay.amarkets.net  is an alias of rbk.pay.amarkets.net`
вставить `<www.domen_name.com>` в строчку конфигурации ngnix `server_name` через пробел, без запятых.

По адресу веб-адресу сервера должна быть надпись:
`Success! The rbk.pay.amarkets.net server block is working!`

Дальнейшая (обязательная) конфигурация nginx и файервола в разделе "Управление релизами"


<a id="orgfeb88f8"></a>

# SSL-сертификат

    
    $ sudo add-apt-repository ppa:certbot/certbot
    $ sudo apt-get update
    $ sudo apt-get install python-certbot-nginx

Апдэйт файервола:

    
    $ sudo ufw allow 'Nginx Full'
    $ sudo ufw delete allow 'Nginx HTTP'

Получение сертификата

    
    $ sudo certbot --nginx -d rbk.pay.amarkets.net
    
    # если нужен еще и <www.domen_name.com>, то команда выглядит так
    $ sudo certbot --nginx -d rbk.pay.amarkets.net -d www.rbk.pay.amarkets.net
    
    # будет ошибка, если <www.domen_name.com> не настроен, как alias в CNAME - поле настройки DNS.


<a id="org2a090fa"></a>

# Postgresql

Подробней - <https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04>

    
    $ sudo apt update
    $ sudo apt install postgresql postgresql-contrib
    
    # создать юзера с таким же именем, как и юзер, под
    # которым зашли на сервер (deploy).
    $ sudo -u postgres createuser --interactive
    
    # сделать одноименную базу
    $ sudo -u postgres createdb deploy
    
    # и тогда можно заходить в консоль постгреса просто:
    psql

Это был юзер и база данных для администрирования и легкого
попадания в консоль postgres.
Создаем теперь юзеров и базы данных для приложения в консоли psql
по следубщему принципу:

    
    # создаем базу данных (нам нужна только для прода)
    postgres=# create database <database_name>;
    
    # создаем пользователя для нее
    postgres=# create user <user_name> with encrypted password '<password>';
    
    # даем пользователю нормальные права
    postgres=# alter user <user_name> with superuser ;
    
    # даем пользователю права на базу
    postgres=# grant all privileges on database <database_name> to <user_name>;


<a id="org0ea63b8"></a>

# Установка elixir, erlang и node.js

Устанавливаем под юзером `deploy`, которого создали ранее. 
`ASDF` будет работать только в папке этого юзера, т.е. если зайти
под другим юзером, эрланга и эликсира не будет.


<a id="org5843370"></a>

## Mенеджер версий asdf

Аналог `rbenv`  для эликсира/эрланга
<https://github.com/asdf-vm/asdf>

    
    $ git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.6.2
    
    $ echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
    $ echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc

Не забыть выйти и заново зайти на сервер (перезагрузать терминал).

    
    $ asdf plugin-add erlang
    $ asdf plugin-add elixir
    $ asdf plugin-add nodejs


<a id="org846bc83"></a>

## Установка Эрланга

    
    $ asdf install erlang 21.1.1

Если в процессе установки есть ошибки такого вида:
`WARNING: It appears that a required development package 'libssl-dev' is not installed.`

Установить недостающие библиотеки:

    
    $ sudo apt-get update && sudo apt-get install libssl-dev

И перезапустить `asdf install erlang 21.1.1`
Эрланг компилируется довольно долго, ± 10 минут.
По завершению должно быть сообщение:

    
    Erlang 21.1.1 has been installed. Activate globally with:
    
    asdf global erlang 21.1.1
    
    Activate locally in the current folder with:
    
    asdf local erlang 21.1.1


<a id="org4b73d12"></a>

## Установка Эликсира

Уставливаем эликсир:

    
    $ asdf install elixir 1.7.4

Проверяем, что все установилось:

    
    $ asdf list
    
    # output:
    
    elixir
     1.7.4
    erlang
     21.1.1

Устанавливаем глобально:

    
    asdf glibal erlang 21.1.1
    asdf global elixir 1.7.4

Проверяем:

    
    asdf current
    
    # output:
    
    elixir         1.7.4    (set by \/home\/ubuntu\/.tool-versions)
    erlang         21.1.1   (set by \/home\/ubuntu\/.tool-versions)
    
    # ставим hex
    $ mix local.hex


<a id="orgf1d20cb"></a>

## Установка Node.js

    
    $ asdf plugin-add nodejs

Теперь немного жести.
Необходимо вручную установить несколько gpg ключей, без которых
нода не устанавливается через asdf. Текущая версия asdf - 0.6.2,
возможно в будущих версиях починят. (Вообще-то должны были смержить
изменения еще в июле 2018, но все-равно не работает).

вот я отписал в asdf issues - <https://github.com/asdf-vm/asdf-nodejs/issues/82#issuecomment-449992361>

делать:

    
    $ gpg --keyserver ipv4.pool.sks-keyservers.net --recv-keys C4F0DFFF4E8C1A8236409D08E73BC641CC11F4C8

Если это не помогло - таким же образом добавить все ключи отсюда - <https://github.com/asdf-vm/asdf-nodejs/commit/9237a7fa0fa70e3b7bfc64b1da49b15136ae2adf>

    
    $ asdf install nodejs 10.4.0
    $ asdf global nodejs 10.4.0


<a id="org43bbacc"></a>

# Конфигурация проекта


<a id="org43b3cc6"></a>

## config/prod.exs

    
    # config/prod.exs дефолтный
    ...
       config :myproject, MyprojectWeb.Endpoint,
         load_from_system_env: true,
         url: [host: "example.com", port: 80],
         cache_static_manifest: "priv/static/cache_manifest.json"
    ...
    
    # config/prod.exs изменить на это
    ...
      config :myproject, MyprojectWeb.Endpoint,
        http: [port: 4000],
        url: [host: "example.com", port: 80],
        cache_static_manifest: "priv/static/manifest.json",
        server: true,
        code_reloader: false
    ...


<a id="orgda796d8"></a>

## Хранение prod.secret.exs

Зайти на сервер под `deploy` и создать в корне
домашней папки место, куда будем копировать продовский конфиг

    
    # на сервере
    cd ~
    $ mkdir app_config
    
    # защищенно копируем c помощью scp (эту команду запустить локально, не на сервере)
    scp ~/myproject/config/prod.secret.exs example.com:/home/deploy/app_config/prod.secret.exs


<a id="org50597e4"></a>

## Distillery, Edeliver


<a id="org7737b1b"></a>

### Добавляем в зависимости в `mix.exs`

    
    def application, do: [
      applications: [
      ...
       # Add edeliver to the END of the list
       extra_applications: [:logger, :runtime_tools, :timex, :httpoison, :edeliver]
       ]
    ]
    
    defp deps do
    [
     ...
     {:edeliver, ">= 1.6.0"},
     {:distillery, ">= 2.0.3", warn_missing: false},
     ]
    end

`mix deps.get`


<a id="orgb2ad85f"></a>

### Создаем релиз(конфиг edeliver)

    
    mix release.init
    
    # Output
    An example config file has been placed in rel/config.exs, review it,
    make edits as needed/desired, and then run `mix release` to build the release

Создать папку `.deliver/` в корне проекта и создать в ней файл `config`
с содержанием:

    
    APP="rbk_payment"
    
    BUILD_HOST="rbk.pay.amarkets.net"
    BUILD_USER="deploy"
    BUILD_AT="/home/deploy/app_build"
    
    PRODUCTION_HOSTS="rbk.pay.amarkets.net"
    PRODUCTION_USER="deploy" 
    DELIVER_TO="/home/deploy/app_release" 

Для того, чтобы подтягивались секреты из prod.secret.exs, добавить в
`.deliver/config` следующее:

    
    pre_erlang_get_and_update_deps() {
      local _prod_secret_path="/home/deploy/app_config/prod.secret.exs"
      if [ "$TARGET_MIX_ENV" = "prod" ]; then
        __sync_remote "
          ln -sfn '$_prod_secret_path' '$BUILD_AT/config/prod.secret.exs'
        "
      fi
    }

-   Добавить в `.gitignore` `.deliver/releases`
-   Закоммитить все перед постройкой релиза (edeliver берет код из гита, поэтому все должно быть

закоммичено).

-   На сервере добавить в `/.profile` последней строчкой(в начале точка с пробелом):

    . /home/deploy/.asdf/asdf.sh

-   Проапдэйтить локали на сервере: `sudo update-locale LC_ALL=en_US.UTF-8`

ИИииии, если все было сделано правильно и сегодня хороший день, запускаем 
на локальной машине не дыша с обязательным указанием ветки, из которой деплоим:
(если не указать, edeliver возьмет тупо `master`)

    $ mix edeliver build release --branch=feature/deploy
    
    # output
    
    BUILDING RELEASE OF PS_RBK APP ON BUILD HOST
    
    -----> Authorizing hosts
    -----> Ensuring hosts are ready to accept git pushes
    -----> Pushing new commits with git to: deploy@rbk.pay.amarkets.net
    -----> Resetting remote hosts to f968a62cfd6a0aff14cae3a5a7de4b36d8e5a8ea
    -----> Cleaning generated files from last build
    -----> Fetching / Updating dependencies
    -----> Compiling sources
    -----> Generating release
    -----> Copying release 0.1.0 to local release store
    -----> Copying ps_rbk.tar.gz to release store
    
    RELEASE BUILD OF PS_RBK WAS SUCCESSFUL!

Если ошибки, во первых проверьте, что указанный хэш коммита
соответствует вашей ветке и в коммите есть все конфиги, которые тут
обсуждались.


<a id="org5eab61c"></a>

# Управление релизами


<a id="org2d20e2a"></a>

## Ngnix и reverse proxy

После того, как запустили приложение,
оно должно быть доступно по адресу
<http://rbk.pay.amarkets.net:4000>

У нас сделан тестовый эндпоинт, по которому 
<http://rbk.pay.amarkets.net:4000/ping> возвращает
`200, ok`

Редактируем серверный блок

    $ sudo vim /etc/nginx/sites-available/rbk.pay.amarkets.net

Добавляем в самом начале файла перед первым `server`:

    upstream phoenix {
         server 127.0.0.1:4000;
    }

Поменять `location`:

    location / {
                allow all;
    
                # Proxy Headers
                proxy_http_version 1.1;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_set_header X-Cluster-Client-Ip $remote_addr;
    
                # WebSockets
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
    
                proxy_pass http://phoenix;
        }

Дальше:

    $ sudo nginx -t
    $ sudo systemctl restart nginx
    $ sudo ufw delete allow 4000
    $ sudo ufw status
    
    # output
    
    Status: active
    
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere
    Nginx Full                 ALLOW       Anywhere
    OpenSSH (v6)               ALLOW       Anywhere (v6)
    Nginx Full (v6)            ALLOW       Anywhere (v6)

Теперь приложение доступно по `https`


<a id="org2613c6b"></a>

## Деплой, администрирование релизов

Что нужно установить и сделать, чтобы релизить на уже подготовленную
машину?

-   Нужно установить на локальную машину `asdf`, Erlang, Elixir, Nodejs как описано для сервера,
    на мак ставится все так же (без `brew`). Ноду можно через брю, главное, чтобы версии совпадали.
-   Использовать ветку, предназначенную для деплоя, в которой будут конфиги edeliver-a и distillery.
-   Настроить SSH-доступ к серверу, нужно, чтобы можно можно было заходить под пользователем `deploy`
    следующим образом - `ssh <домен.com>`, для этого:
    
    -   скопировать свой ключ на сервер в `~/.ssh/authorized_keys` (попросить того, у кого уже есть доступ)
    -   локально добавить в `.ssh/config`:
    
        Host <домен.com>
            HostName <домен.com>
            User deploy
            IdentityFile ~/.ssh/private_key_file


<a id="orga1fa803"></a>

### Команды деплоя

    # билд релиза
    $ mix edeliver build release --branch=feature/deploy
    (проверить хэш коммита, чтобы точно вы сбилдили то, что хотели)
    
    #output
    BUILDING RELEASE OF PS_RBK APP ON BUILD HOST
    
    -----> Authorizing hosts
    -----> Ensuring hosts are ready to accept git pushes
    -----> Pushing new commits with git to: deploy@rbk.pay.amarkets.net
    -----> Resetting remote hosts to 405f7ba77b5a4fb2bb3f5fd6b3f3f13c72caea34 # <----- вот он хэш коммита
    -----> Cleaning generated files from last build
    -----> Fetching / Updating dependencies
    -----> Running npm install
    -----> Compiling assets
    -----> Running phoenix.digest
    -----> Compiling sources
    -----> Generating release
    -----> Copying release 0.1.0+deploywebhook-405f7ba-20190109-123942 to local release store
    -----> Copying ps_rbk.tar.gz to release store
    
    RELEASE BUILD OF PS_RBK WAS SUCCESSFUL!
    
    # остановка сервера на проде
    $ mix edeliver stop production
    
    # деплой
    $ mix edeliver deploy release to production
    (выбрать копипастой нужный релиз из списка)
    
    # старт сервера
    $ mix edeliver start production
    (из-за багов edeliver у многих есть эти ошибки, но сервер запускается,
    главное, чтобы в конце было написано  START DONE!)
    
    #output
    
    EDELIVER PS_RBK WITH START COMMAND
    
    -----> starting production servers
    
    production node:
    
    user    : deploy
    host    : rbk.pay.amarkets.net
    path    : /home/deploy/app_release
    response: ▸  Received 'pang' from ps_rbk@127.0.0.1!
    ▸  Possible reasons for this include:
    ▸    - The cookie is mismatched between us and the target node
    ▸    - We cannot establish a remote connection to the node
    ▸  Received 'pang' from ps_rbk@127.0.0.1!
    ▸  Possible reasons for this include:
    ▸    - The cookie is mismatched between us and the target node
    ▸    - We cannot establish a remote connection to the node
    
    
    START DONE!
    
    
    
    $ mix edeliver ping production # shows which nodes are up and running
    $ mix edeliver version production # shows the release version running on the nodes
    $ mix edeliver show migrations on production # shows pending database migrations
    $ mix edeliver migrate production # run database migrations
    $ mix edeliver restart production # or start or stop

Новый релиз взамен старого c остановкой прода:

-   билдим `$ mix edeliver build release --branch=feature/deploy`
-   останавливаем на проде: `$ mix edeliver stop production`
-   деплоим `$ mix edeliver deploy release to production`
-   запускаем на проде `$ mix edeliver start production`
-   запускаем миграции (накатываются на работающее приложение без проблем). `$ mix edeliver migrate production`

По умолчанию запускается самый новый релиз.


<a id="orge955312"></a>

### Логи

Логи находятся в `app_release/<название_приложения>/var/logs`

    .
    ├── erlang.log.1
    ├── erlang.log.3
    ├── erlang.log.4
    ├── erlang.log.5
    └── run_erl.log

При деплое и новом запуске (перезапуске) приложения, если нет файлов в этой директории,
создается новый файл. Если есть путаница, куда пишутся логи (или не пишутся), 
лучше удалить все файлы отсюда и перезапустить приложение. Останется `erlang.log.1`, в 
который точно будут писаться логи. (рецепт не для прода).


<a id="orge576074"></a>

### Компилирование ассетов при деплое

Добавить в `.deliver/config`

    # for compiling assets
    
    pre_erlang_clean_compile() {
    status "Running npm install"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile
          set -e
          cd '$BUILD_AT'/assets
          npm install
        "
    
    status "Compiling assets"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile
          set -e
          cd '$BUILD_AT'/assets
          node_modules/.bin/webpack --mode production --silent
        "
    
    status "Running phoenix.digest"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile 
          set -e 
          cd '$BUILD_AT'
          mkdir -p priv/static
          APP='$APP' MIX_ENV='$TARGET_MIX_ENV' $MIX_CMD phx.digest $SILENCE
        "
     }


<a id="org2e39329"></a>

## .edeliver/config - финальный вид

    APP="ps_rbk"
    
    BUILD_HOST="rbk.pay.amarkets.net"
    BUILD_USER="deploy"
    BUILD_AT="/home/deploy/app_build"
    
    PRODUCTION_HOSTS="rbk.pay.amarkets.net"
    PRODUCTION_USER="deploy" 
    DELIVER_TO="/home/deploy/app_release" 
    
    AUTO_VERSION=git-branch+git-revision+build-date+build-time
    
    # for implementing prod.secret.exs in prod server
    
    pre_erlang_get_and_update_deps() {
      local _prod_secret_path="/home/deploy/app_config/prod.secret.exs"
      if [ "$TARGET_MIX_ENV" = "prod" ]; then
        __sync_remote "
          ln -sfn '$_prod_secret_path' '$BUILD_AT/config/prod.secret.exs'
        "
      fi
    }
    
    # for compiling assets
    
    pre_erlang_clean_compile() {
    status "Running npm install"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile
          set -e
          cd '$BUILD_AT'/assets
          npm install
        "
    
    status "Compiling assets"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile
          set -e
          cd '$BUILD_AT'/assets
          node_modules/.bin/webpack --mode production --silent
        "
    
    status "Running phoenix.digest"
        __sync_remote "
          [ -f ~/.profile ] && source ~/.profile 
          set -e 
          cd '$BUILD_AT'
          mkdir -p priv/static
          APP='$APP' MIX_ENV='$TARGET_MIX_ENV' $MIX_CMD phx.digest $SILENCE
        "
     }


<a id="org2da379c"></a>

# Возможные проблемы

При релизе возникает такая ошибка:

    'erlang-build-release' strategy does not exist
    
    edeliver v1.4.5 | https://github.com/boldpoker/edeliver
    
    Available strategies:

Это баг (очередной) edeliver-a, нужно проверить локальный путь к папке приложения
на отсутствие пробелов. Типа `../My projects/payment_systems`, так вот, убрать пробелы надо.


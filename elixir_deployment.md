
# Table of Contents

1.  [Настраиваем фаервол](#orga4c0c4b)
2.  [Добавляем пользователя для деплоя](#orgf4feb5c)
3.  [Устанавливаем NGNIX](#org39f9d22)
    1.  [Установка, запуск Ngnix](#orgb9f39f7)
    2.  [Конфигурация Ngnix](#org2deeb2d)
4.  [SSL-сертификат](#org2ff9486)
5.  [Postgresql](#org216a9fe)
6.  [Установка elixir, erlang и node.js](#orgd13ef7b)
    1.  [Mенеджер версий asdf](#org5db991b)
    2.  [Установка Эрланга](#org07e43e7)
    3.  [Установка Эликсира](#orge42eac4)
    4.  [Установка Node.js](#org222e197)

Англоязыйчый гайд - <https://www.digitalocean.com/community/tutorials/how-to-automate-elixir-phoenix-deployment-with-distillery-and-edeliver-on-ubuntu-16-04>


<a id="orga4c0c4b"></a>

# Настраиваем фаервол

Осторожно! Можно самому себе забокировать доступ на сервер!
Дальнейшее подразумевает, что вы зашли(уже зашли) по ssh ключу и он
добавлен(уже лежит) в `/home/ubuntu/.ssh/authorized_keys`

    
    sudo ufw app list
    
    # output:
    
    Available applications:
    OpenSSH
    
    sudo ufw allow OpenSSH
    
    sudo ufw enable
    
    sudo ufw status
    
    # output:
    
    Status: active
    
    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere
    OpenSSH (v6)               ALLOW       Anywhere (v6)
    
    # добавить порт для Феникса
    
    sudo ufw allow 4000


<a id="orgf4feb5c"></a>

# Добавляем пользователя для деплоя

И даем ему права.

    
    adduser deploy
    sudo usermod -a -G sudo deploy

Не забудьте пароль записать/запомнить.

Делаем возможность заходить под `deploy` через ssh
и копируем права.
(внимательно с пробелами!)

    
    sudo rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy

Проверить:

-   зайти в новом терминале `ssh deploy@<ip-адрес сервера>`


<a id="org39f9d22"></a>

# Устанавливаем NGNIX


<a id="orgb9f39f7"></a>

## Установка, запуск Ngnix

Заходим на сервер через ssh под юзером `deploy`

    
    sudo apt update
    sudo apt install nginx

Апдэйтим файервол:

    
    sudo ufw allow 'Nginx HTTP'
    sudo ufw status
    
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

    
    systemctl status nginx
    
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

    
    sudo systemctl stop nginx
    sudo systemctl start nginx
    sudo systemctl restart nginx
    
    # при изменеиии конфигов перезапускать не обязательно:
    
    sudo systemctl reload nginx


<a id="org2deeb2d"></a>

## Конфигурация Ngnix

Делаем отдельный новый серверный блок для приложения.
Дефолтный старый серверный блок будет отдавать 404 страницы,
если страница не найдена в приложении.

    
    sudo mkdir -p /var/www/rbk.pay.amarkets.net/html
    sudo chown -R $USER:$USER /var/www/rbk.pay.amarkets.net/html
    sudo chmod -R 755 /var/www/rbk.pay.amarkets.net
    
    # отредактировать home-страницу для нового блока
    
    vim /var/www/rbk.pay.amarkets.net/html/index.html
    
    # завести новый конфиг-файл для нового блока
    
    sudo vim /etc/nginx/sites-available/rbk.pay.amarkets.net
    
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
    
    sudo ln -s /etc/nginx/sites-available/rbk.pay.amarkets.net /etc/nginx/sites-enabled/
    
    # подправить кофиг
    
    sudo vim /etc/nginx/nginx.conf
    
    # раскомментить строчку:
    
    ...
    http {
    ...
          server_names_hash_bucket_size 64;
    ...
    }
    ...
    
    # проверить, что конфигурация без ошибок:
    
    sudo nginx -t
    
    # output:
    
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    
    # перезапустить nginx:
    
    sudo systemctl restart nginx

По адресу веб-адресу сервера должна быть надпись:
`Success! The rbk.pay.amarkets.net server block is working!`


<a id="org2ff9486"></a>

# SSL-сертификат

    
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install python-certbot-nginx

Апдэйт файервола:

    
    sudo ufw allow 'Nginx Full'
    sudo ufw delete allow 'Nginx HTTP'

Получение сертификата

    
    sudo certbot --nginx -d rbk.pay.amarkets.net


<a id="org216a9fe"></a>

# Postgresql

Подробней - <https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04>

    
    sudo apt update
    sudo apt install postgresql postgresql-contrib
    
    # создать юзера с таким же именем, как и юзер, под
    # которым зашли на сервер (deploy).
    sudo -u postgres createuser --interactive
    
    # сделать одноименную базу
    sudo -u postgres createdb deploy
    
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


<a id="orgd13ef7b"></a>

# Установка elixir, erlang и node.js

Устанавливаем под юзером `deploy`, которого создали ранее. 
`ASDF` будет работать только в папке этого юзера, т.е. если зайти
под другим юзером, эрланга и эликсира не будет.


<a id="org5db991b"></a>

## Mенеджер версий asdf

Аналог `rbenv`  для эликсира/эрланга
<https://github.com/asdf-vm/asdf>

    
    git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.6.2
    
    echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
    echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc

Не забыть выйти и заново зайти на сервер (перезагрузать терминал).

    
    asdf plugin-add erlang
    asdf plugin-add elixir


<a id="org07e43e7"></a>

## Установка Эрланга

    
    asdf install erlang 21.1.1

Если в процессе установки есть ошибки такого вида:
`WARNING: It appears that a required development package 'libssl-dev' is not installed.`

Установить недостающие библиотеки:

    
    sudo apt-get update && sudo apt-get install libssl-dev

И перезапустить `asdf install erlang 21.1.1`
Эрланг компилируется довольно долго, ± 10 минут.
По завершению должно быть сообщение:

    
    Erlang 21.1.1 has been installed. Activate globally with:
    
    asdf global erlang 21.1.1
    
    Activate locally in the current folder with:
    
    asdf local erlang 21.1.1


<a id="orge42eac4"></a>

## Установка Эликсира

Уставливаем эликсир:

    
    asdf install elixir 1.7.4

Проверяем, что все установилось:

    
    asdf list
    
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


<a id="org222e197"></a>

## Установка Node.js

    
    asdf plugin-add nodejs

Теперь немного жести.
Необходимо вручную установить несколько gpg ключей, без которых
нода не устанавливается через asdf. Текущая версия asdf - 0.6.2,
возможно в будущих версиях починят. (Вообще-то должны были смержить
изменения еще в июле 2018, но все-равно не работает).

вот я отписал в asdf issues - <https://github.com/asdf-vm/asdf-nodejs/issues/82#issuecomment-449992361>

делать:

    
    gpg --keyserver ipv4.pool.sks-keyservers.net --recv-keys C4F0DFFF4E8C1A8236409D08E73BC641CC11F4C8

Если это не помогло - таким же образом добавить все ключи отсюда - <https://github.com/asdf-vm/asdf-nodejs/commit/9237a7fa0fa70e3b7bfc64b1da49b15136ae2adf>

    
    asdf install nodejs 10.4.0


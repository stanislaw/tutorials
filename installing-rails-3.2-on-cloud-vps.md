<meta charset='utf-8'>

# Установка Ruby 1.9.3, Unicorn, Nginx, Mysql на CloudVPS-сервер компании Hostpro (дистрибутив CentOS 5.5 x86)

Это краткий конспект шагов, необходимых для установки и запуска нескольких одновременно работающих Ruby on Rails приложения на CloudVPS сервере Hostpro с чистым дистрибутивом CentOS 5.5.

## Устанавливаемое ПО

* CentOS 5.5
* Mysql 5
* Nginx
* RVM
* Ruby 1.9.3
* Unicorn web server
* Rails 3.2

CentOS - операционная система, MySQL - базы данных, с ними вроде
всё ясно.

Nginx + Unicorn - одна из самых популярных связок в Ruby-мире.  Nginx встречает запросы и отдаёт статику (картинки, скрипты, css и прочее), а остальное передаёт Rails-приложению т.е. Unicorn'у. Это позволяет  сильно увеличить скорость обработки запросов и существенно снизить нагрузку с приложения. https://github.com/blog/517-unicorn.

RVM - Ruby Version Manager. Позволяет использовать разные версии Ruby и наборы gem-пакетов в Rails-приложениях. Например, одно из приложений всё ещё работает на Ruby 1.8.7 и использует старые версии gem'ов - типичный пример Redmine, который не хочется апгрейдить, - мы создаём под него свою конфигурацию Ruby 1.8.7 + пакеты и т.д. https://rvm.io

## Начинаем

### Создаём пользователя и настраиваем SSH-доступ

Уже в самом начале мне было удобно задать для сервера алиас и прописать его в настройках *~/.ssh/config*

```bash
Host hostpro
  Hostname 194.28.86.211 # или Hostname onapp.my_domain.net
```

Такая запись позволяет впоследствии писать просто

```bash
ssh hostpro # вместо root@onapp.my_domain.net
```

Итак, перед нами чистый дистрибутив CentOS с root-доступом:

```bash
ssh root@hostpro
Last login: Fri Jun 15 01:20:47 2012 from 91.218.89.42

[root@onapp ~]# cat /etc/redhat-release
CentOS release 5.5 (Final)

[root@onapp ~]# pwd
/root
```

Добавляем пользователя.

```bash
useradd -m -G users wheel stanislaw
```

Задаём пароль:

```bash
passwd stanislaw
cd /home/stanislaw
```

Со своего компьютера копируем публичный ключ ssh, на своём компьютере:

```bash
scp ~/.ssh/id_rsa.pub hostpro:/home/stanislaw
```

На сервере содержимое публичного ключа добавляем в перечень допустимых ключей (сервер чистый, поэтому, конечно, файл *~/.ssh/authorized_keys* пока пуст):

```bash
[stanislaw@onapp ~]$ mkdir ~/.ssh
[stanislaw@onapp ~]$ cat ~/id_rsa >> ~/.ssh/authorized_keys
```

Результат - теперь мы можем просто писать

```bash
ssh hostpro
```
затем вводить свой любимый пароль и попадать в домашнюю директорию
созданного пользователя.

### Устанавливаем MySQL

Устанавливаем пакеты:

```bash
su
yum install mysql mysql-devel mysql-server
```

Запускаем MySQL:

```bash
/etc/init.d/mysqld start
```

Задаём root-пароль:

```bash
/usr/bin/mysqladmin -u root password 'root-cool-pass'
```

Заходим в mysql-консоль, чтобы создать пользователя stanislaw:

```bash
mysql -u root -p
mysql> GRANT ALL PRIVILEGES ON *.* TO 'stanislaw'@'localhost'
IDENTIFIED BY 'mysql-cool-pass' WITH GRANT OPTION;
```

Впоследствии, конечно, желательно ограничить права пользователя stanislaw - разрешить только те базы, для которых ему действительно нужен доступ.

Добавляем MySQL в автозапуск. Сервер по умолчанию работает на runlevel 3.

```bash
/sbin/chkconfig --add mysqld
/sbin/chkconfig --level 3 mysqld on
```

### Nginx

```bash
yum install nginx
```

После установки нужно сделать

```bash
chmod 0751 /home/stanislaw # Если интересно, причина: http://stackoverflow.com/questions/6795350/nginx-403-forbidden-for-all-files
```

Добавляем в авто-запуск

```bash
sudo /sbin/chkconfig nginx on

sudo /sbin/chkconfig --list nginx
nginx           0:off   1:off   2:on    3:on    4:on    5:on    6:off
```


### Git
в пакетах CentOS 5 git отсутствует.

Наиболее часто используемое решение - использовать дополнительный репозиторий (http://www.webtatic.com/packages/git17/):

```bash
rpm -Uvh http://repo.webtatic.com/yum/centos/5/latest.rpm
yum install --enablerepo=webtatic git-all
```

## ДОПОЛНИТЕЛЬНЫЕ ЗАВИСИМОСТИ

### ImageMagick

Его обязательно потребует при установке gem *rmagick*, который используется практически в любом Rails-проекте. Понадобится свежая версия прямо с родного сайта, т.к. версия в репозитории CentOS 5 очень давняя:

```bash
wget http://www.imagemagick.org/download/linux/CentOS/i386/ImageMagick-6.7.7-7.i386.rpm
wget http://www.imagemagick.org/download/linux/CentOS/i386/ImageMagick-devel-6.7.7-7.i386.rpm
```

Библиотеки, которые потребуются при сборке:

```bash
yum install libtool-ltdl lzma freetype-devel jasper-devel
```

```bash
rpm -Uvh ImageMagick-6.7.7-7.i386.rpm
rpm -Uvh ImageMagick-devel-6.7.7-7.i386.rpm
```

## Ruby-часть

### Ruby + RVM

RVM я устанавливаю в домашнюю директорию своего пользователя, т.е.  все действия (кроме установки пакетов с использованием *yum*) в этом разделе ведутся от имени пользователя, а не администратора!

Чтобы не возиться с SSL-сертификатами (после установки RVM это, конечно, лучше убрать):

```bash
echo insecure >> ~/.curlrc
```

Устанавливаем RVM:

```bash
curl -L https://get.rvm.io | bash -s stable
```

Смотрим, что нужно для установки Ruby, команда

```bash
rvm requirements
```

отобразит все необходимые зависимости, которые выглядят так:

```bash
yum install -y gcc-c++ patch readline readline-devel zlib zlib-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison
```

Дополнительные зависимости для разных гемов:

```bash
yum install libxml2 libxml2-devel libxslt libxslt-devel
```

Наконец, можно установить Ruby:

```bash
rvm install 1.9.3
```

И Bundler

```bash
gem install bundler
```

### Unicorn

```ruby
gem install unicorn
```

Наверное, единственное, что стоит отметить про связку Nginx + Unicorn - это то, что между собой Unicorn и Nginx могут взаимодействовать как через порты, так и через сокеты - второй способ предпочтителен и используется в конфигурации ниже. Т.е. Unicorn слушает socket-файл, а не, допустим, 80 порт. Это означает, что со стороны портов Unicorn можно закрыть вообще, см. соответствующий комментарий в конфигурационном файле *config/unicorn.rb* ниже.

Предположим, что у нас есть два Rails-приложения: **first_app** и **second_app**.

Каждое приложение должно содержать конфигурационный файл *config/unicorn.rb* со следующим содержанием (пример для **first_app**):

#### Конфигурация Unicorn (config/unicorn.rb)

```ruby
#worker_processes 4
deploy_to = "/home/stanislaw/apps/first_app"
rails_root = "#{deploy_to}/current"
pid_file = "#{deploy_to}/shared/pids/unicorn.pid"
socket_file = "#{deploy_to}/shared/unicorn.sock"
log_file = "#{rails_root}/log/unicorn.log"
error_log_file = "#{rails_root}/log/unicorn_error.log"
old_pid = pid_file + ".oldbin"

working_directory rails_root

listen socket_file, :backlog => 64

# listen 8080, :tcp_nopush => true # Слушать порт нет необходимости,
потому что nginx будет передавать запросы через socket-файл.

pid pid_file

stderr_path error_log_file
stdout_path log_file

timeout 30

preload_app true
GC.respond_to?(:copy_on_write_friendly=) and
  GC.copy_on_write_friendly = true

before_exec do |server|
  ENV["BUNDLE_GEMFILE"] = "#{rails_root}/Gemfile"
end

before_fork do |server, worker|
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!

  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      # someone else did our job for us
    end
  end
end

after_fork do |server, worker|
  # per-process listener ports for debugging/admin/migrations
  # addr = "127.0.0.1:#{9293 + worker.nr}"
  # server.listen(addr, :tries => -1, :delay => 5, :tcp_nopush => true)

  # the following is *required* for Rails + "preload_app true",
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection

  # if preload_app is true, then you may also want to check and
  # restart any other shared sockets/descriptors such as Memcached,
  # and Redis.  TokyoCabinet file handles are safe to reuse
  # between any number of forked children (assuming your kernel
  # correctly implements pread()/pwrite() system calls)
end
```

Обратите особое внимание на путь, указанный в переменной *pid_file* - он понадобится нам для работы скрипта автозапуска */etc/init.d/unicorn*, а путь из *socket_file* мы укажем в настройках Nginx.

#### Конфигурация Nginx (/etc/nginx/nginx.conf)

```text
user              nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log;
#error_log  /var/log/nginx/error.log  notice;
#error_log  /var/log/nginx/error.log  info;

pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip on;
    gzip_http_version 1.0;
    gzip_proxied any;
    gzip_min_length 500;
    gzip_disable "MSIE [1-6]\.";
    gzip_types text/plain text/html text/xml text/css
             text/comma-separated-values
             text/javascript application/x-javascript
             application/atom+xml;

    include /etc/nginx/conf.d/*.conf;

    upstream first_app_server {
        server unix:/home/stanislaw/apps/first_app/shared/unicorn.sock
fail_timeout=0;
    }

    upstream second_app_server {
        server unix:/home/stanislaw/apps/second_app/shared/unicorn.sock
fail_timeout=0;
    }

    server {
        listen 80 default_server;

        client_max_body_size 4G;

        server_name first_app.com www.first_app.com;

        location /robots.txt { alias /usr/local/etc/nginx/robots.txt; }

        location ~ ^/assets/ {
            expires 1y;
            add_header Cache-Control public;

            add_header Last-Modified "";
            add_header ETag "";
            break;
        }

        keepalive_timeout 5;

        root /home/stanislaw/apps/first_app/current/public;

        try_files $uri/index.html $uri.html $uri @app;

        location @app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_set_header Host $http_host;

            proxy_redirect off;

            proxy_pass http://first_app_server;
        }

        error_page 500 502 503 504 /500.html;
        location = /500.html {
            root /home/stanislaw/apps/first_app/current/public;
        }
    }

    server {
        listen 80 default_server;

        client_max_body_size 4G;

        server_name second_app.com www.second_app.com;

        location /robots.txt { alias /usr/local/etc/nginx/robots.txt; }

        location ~ ^/assets/ {
            expires 1y;
            add_header Cache-Control public;

            add_header Last-Modified "";
            add_header ETag "";
            break;
        }

        keepalive_timeout 5;

        root /home/stanislaw/apps/second_app/current/public;

        try_files $uri/index.html $uri.html $uri @app;

        location @app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_set_header Host $http_host;

            proxy_redirect off;

            proxy_pass http://second_app_server;
        }

        error_page 500 502 503 504 /500.html;
        location = /500.html {
            root /home/stanislaw/apps/second_app/current/public;
        }
    }
}
```

#### Конфигурация Capistrano (config/deploy.rb)

```ruby
load 'deploy/assets'
# require 'bundler/capistrano'

# Важные строки для правильного включения RVM
https://rvm.io//integration/capistrano/
set :rvm_ruby_string, '1.9.3'
require 'rvm/capistrano'

set :user, "stanislaw"
set :application, "first_app"
set :repository,  "."

set :hostpro_ip, "194.28.86.211"

set :deploy_to, "/home/stanislaw/apps/first_app"

set :scm, :none
set :scm_verbose, :true
set :deploy_via, :copy
set :use_sudo, :false
set :ssh_options, { :forward_agent => true, :keys =>
%w(/home/stanislaw/.ssh/id_rsa) }
set :unicorn_conf, "#{deploy_to}/current/config/unicorn.rb"
set :unicorn_pid, "#{deploy_to}/shared/pids/unicorn.pid"

role :web, hostpro_ip # Your HTTP server, Apache/etc
role :app, hostpro_ip # This may be the same as your `Web` server
role :db, hostpro_ip, :primary => true # This is where Rails migrations will run
# role :db,  "your slave db-server here"

before "deploy:assets:precompile" do
  run "cd #{latest_release}; bundle install --without=development test"
end

after "deploy:assets:precompile" do
  # run "cd #{latest_release}; bundle install --without=development test"
end

after 'deploy:update_code' do
  # run "cd #{latest_release}; RAILS_ENV=production rake assets:precompile"
end

namespace :deploy do
  task :restart do
    run "kill -USR2 `cat #{unicorn_pid}`"
  end
  task :start do
    run "cd #{deploy_to}/current; bundle exec unicorn_rails -c
#{unicorn_conf} -E #{rails_env} -D"
  end
  task :stop do
    run "kill -QUIT `cat #{unicorn_pid}`"
  end
end
```

#### Скрипт для авто-запуска /etc/init.d/unicorn (не забудьте сделать его исполняемым)

```bash
#!/bin/bash

#chkconfig: 345 20 80
#description: Multiple unicorns startup script

# init.d script for single or multiple unicorn installations. Expects
at least one .conf
# file in /etc/unicorn
#
# (2012.06.18) Modified by s.pankevich@gmail.com
http://github.com/stanislaw, as taken from:
http://jay.gooby.org/post/an-etcinitdunicorn-script-for-multiple-unicorn-installations:
# * RVM-based environments (.rvmrc should go in RAILS_ROOT if it
differs from default gemset)
# * Replaced 'exit 0' with 'return'. It was breaking entire script
after executing command (fx. stop) for the first file in /etc/unicorn/

# Modified by jay@gooby.org http://github.com/jaygooby
# based on http://gist.github.com/308216 by http://github.com/mguterl
#
## A sample /etc/unicorn/my_app.conf
##
## RAILS_ENV=production
## RAILS_ROOT=/home/stanislaw/apps/my_app/current
## UNICORN_PID=/home/stanislaw/apps/my_app/shared/pids/unicorn.pid

# This configures a unicorn master for your app at
/home/stanislaw/apps/my_app/current running in
# production mode. It will read config/unicorn.rb for further set up.
#
# You should ensure different ports or sockets are set in each
config/unicorn.rb if
# you are running more than one master concurrently.
#
# If you call this script without any config parameters, it will
attempt to run the
# init command for all your unicorn configurations listed in /etc/unicorn/*.conf
#
# /etc/init.d/unicorn start # starts all unicorns
#
# If you specify a particular config, it will only operate on that one
#
# /etc/init.d/unicorn start /etc/unicorn/my_app.conf

set -e

sig () {
  test -s "$PID" && kill -$1 `cat "$PID"`
}

oldsig () {
  test -s "$OLD_PID" && kill -$1 `cat "$OLD_PID"`
}

cmd () {
  case $1 in
    start)
      sig 0 && echo >&2 "Already running" && return;
      echo "Starting";
      eval $CMD;
      ;;
    stop)
      sig QUIT && echo "Stopping" && return;
      echo >&2 "Not running";
      ;;
    force-stop)
      sig TERM && echo "Forcing a stop" && return;
      echo >&2 "Not running"
      ;;
    restart|reload)
      sig USR2 && sleep 5 && oldsig QUIT && echo "Killing old master"
`cat $OLD_PID` && return;
      echo >&2 "Couldn't reload, starting '$CMD' instead"
      eval $CMD
      ;;
    upgrade)
      sig USR2 && echo Upgraded && return;
      echo >&2 "Couldn't upgrade, starting '$CMD' instead"
      eval $CMD
      ;;
    rotate)
      sig USR1 && echo rotated logs OK && return;
      echo >&2 "Couldn't rotate logs" && exit 1
      ;;
    *)
      echo >&2 "Usage: $0 <start|stop|restart|upgrade|rotate|force-stop>"
      exit 1
      ;;
  esac
}

setup () {
  echo -n "$RAILS_ROOT: "
  cd $RAILS_ROOT || exit 1

  export PID=$UNICORN_PID
  export OLD_PID="$PID.oldbin"

  # We need to source .bash_profile to enable RVM!
  UNICORN_CMD="source ~/.bash_profile; unicorn_rails -c
config/unicorn.rb -E $RAILS_ENV -D"

  CMD="su -s /bin/bash stanislaw -c \"$UNICORN_CMD\""
}

start_stop () {

  # either run the start/stop/reload/etc command for every config
under /etc/unicorn
  # or just do it for a specific one

  # $1 contains the start/stop/etc command
  # $2 if it exists, should be the specific config we want to act on
  if [ $2 ]; then
    . $2
    setup
    cmd $1
  else
    for CONFIG in /etc/unicorn/*.conf; do
      # import the variables
      . $CONFIG
      setup

      # run the start/stop/etc command
      cmd $1
    done
   fi
}

ARGS="$1 $2"
start_stop $ARGS
```

Этот скрипт предполагает, что для каждого приложения в папке */etc/unicorn/* создаётся отдельный файл, например

#### /etc/unicorn/first_app.conf

```bash
#!/bin/sh

RAILS_ENV=production
UNICORN_PID=/home/stanislaw/apps/first_app/shared/pids/unicorn.pid
RAILS_ROOT=/home/stanislaw/apps/first_app/current
```

Чтобы при перезагрузке VPS сервера все Unicorn'ы запускались, нужно обязательно изменить порядок запуска startup-скриптов в /etc/rc3.d (сервер работает на runlevel 3): unicorn должен идти после mysql, т.е. файл *SЦИФРАunicorn* в директории */etc/rc3.d/* должен иметь цифру большую, чем файл *SЦИФРАmysqld*

#### Важный момент

По умолчанию каждое приложение из */home/stanislaw/apps/* будет использовать версию Ruby 1.9.3, которая была установлена в начале. Предположим, что для приложения **second_app** нам нужна версия 1.8.7. Устанавливаем её дополнительно:

```bash
rvm install 1.8.7
```

<p>и создаём в корне приложения **second_app** файл *.rvmrc*:

```bash
# http://stackoverflow.com/a/5143967/598057
rvm use 1.8.7
```

## After all

### MySQL

В целях большей безопасности рекомендуется отключить сетевой порт 3306, на который по умолчанию настроен MySQL. В файл */etc/my.cnf* добавляем

```text
skip-networking
```

И в обоих приложениях исправляем в *config/database.yml* секцию *production*:

```yaml
production:
  adapter:  mysql2
  database: albumer_production
  username: stanislaw
  password: stanislaw-cool-pass
  encoding: utf8
  socket:   /var/lib/mysql/mysql.sock
```

## РАЗНОЕ

С самого начала, заходя на ssh@hostpro или производя на нём какие-либо операции, наблюдал такие строки:

```bash
tput: unknown terminal "rxvt-unicode"
```

Подошло такое решение:

```bash
Solution 1:
The terminfo file /usr/share/terminfo/r/rxvt-unicode is missing from
the remote box. scp it from your laptop.
```

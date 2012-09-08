<meta charset='utf-8'>

# Запуск нескольких ruby микро-приложений на одном веб-сервере

Эта статья является продолжением статьи ["Установка Ruby 1.9.3, Unicorn, Nginx, Mysql на CloudVPS-сервер компании Hostpro (дистрибутив CentOS 5.5 x86)"](http://blog.hostpro.ua/nastroyka-ruby-1-9-3-v-oblake-hostpro), в которой детально описан процесс установки Ruby окружения на чистый CloudVPS-сервер компании Hostpro с возможностью запуска на нём нескольких работающих одновременно Rails-приложений.

В этой статье предполагается, что у читателя Ruby-окружение настроено аналогично описанному в предыдущей статье, главным образом:

* Дистрибутив CentOS 5 или 6
* RVM-based локальное Ruby-окружение - Ruby и gem-пакеты находятся ```~/.rvm```, где ```~/``` директория пользователя
* Используется Nginx как прокси-сервер.

## Постановка задачи

### Микроприложения

В пределах данной статьи мы будем считать, что микроприложения - это в первую очередь мини-сайты: статические сайты, блог-сайты (mywebdev blog и т.п.) и даже Rails-приложения (малое число зависимостей, облегчённые варианты) и во вторую очередь - простейшие API-сервисы. 

### Зачем?

Предположим, на вашем личном VPS сервере уже установлено одно "средней тяжести" Rails-приложение, которое занимает большую часть ресурсов вашего сервера. Кроме того, на этом сервере вяло работает Redmine, который используете вы и ещё несколько ваших коллег. Данная конфигурация сервера с двумя и более приложениями является типичной и встречается очень часто в мире web-разработки и VPS-серверов.

Предположим далее, что вдобавок у вас есть несколько маленьких личных проектов: например, пара блогов и один полу-статический сайт на Sinatra, которые вы хотели бы также разместить на вашем сервере в виду малого размера этих проектов.

Вот тут и возникает проблема экономии ресурсов, так как, как правило, сервер и соответствующий хостинг-план первоначально покупается под какой-то один ("полтора", если используется Redmine) основной несущий проект, откуда следует, что желание "подселить" на сервер пару-тройку проектов, пусть даже и небольших, оборачивается нехваткой ресурсов, в первую очередь RAM. Так, например, если для каждого мини-приложения вы будете создавать свой экземпляр Unicorn (как это описано в предыдущей статье), вы с большой вероятностью стокнётесь с тем, что вам не хватает объёма RAM, определённого вашим хостинг-пакетом.

Суть решения очень проста: так как приложения небольшие и вероятнее всего не предполагают большой посещаемости и как следствие высокой нагрузки, мы запустим один специальный веб-сервер, который будет обслуживать запросы ко всем нашим мини-проектам. Математика проста: если у вас есть 4 мини-приложения, то вместо 4 экземляров веб-сервера (если следовать подходу, описанному в предыдущей статье "одно приложение - один веб-сервер") вы получаете всего один.

Для того, чтобы решить данную задачу, достаточно вспомнить, что все Ruby-приложения в 95% случаев являются также и Rack-приложениями, для этого понадобится опуститься на один уровень глубже и разобраться в том, что общего между, например, Sinatra- и Rails-приложениями.

## Что понадобится

* RackStack
* Puma
* Знание Git submodules
* Несколько приложений на Ruby.

### RackStack

Задача - организовать из наших приложений своеобразную "гирлянду", которую мы хотим повесить на один единственный веб-сервер, ответственный за микро-приложения. На языке Ruby - это, вероятно, можно назвать "стеком, составленным из Rack-приложений" или "Rack apps stack".

**Примечание**. Если вы не знаете, что Rails или Sinatra в своей основе являются Rack-приложениями - прочитайте, например, эту [статью](http://m.onkey.org/ruby-on-rack-1-hello-rack) и [эту](http://habrahabr.ru/post/131429/).

На самом низком уровне Ruby, эту задачу может решить ```Rack::Builder``` (любители низких уровней могут в деталях рассмотреть [Rack::Builder](http://rack.rubyforge.org/doc/classes/Rack/Builder.html)). Мы же воспользуемся [RackStack](https://github.com/remi/rack-stack) - он позволит нам сделать то же самое, что делает RackBuilder, но проще и изящнее.

`RackStack` прямо основан на `Rack::Builder` и, так же как и свой родитель, является своеобразным маршрутизатором для Rack-приложений. `RackStack` имеет очень удобную для наших целей функцию выбора определённого Rack-приложения в зависимости от имени хоста, с которого поступил запрос (см. далее в примерах).

### Puma

На роль вебсервера автор статьи чисто из любопытства выбрал [Puma](http://puma.io/), хотя, конечно, решение, предлагаемое в данной статье, может запросто использовать Thin, Unicorn и другие. На сайте Puma очень хорошие сравнительные показатели - Memory Usage Comparison, знак качества "Project by EngineYard" и красивые картинки ;) И конечно "It is designed for running Rack apps only."

В статье используются модифицированные для CentOS 5 скрипты авто-запуска (см. ниже), расположенные в репозитории Puma: https://github.com/puma/puma/tree/master/tools/jungle

### Знание Git submodules

Автору показалось, что использование главного приложения, основанного на RackStack с мини-приложениями в качестве подмодулей, является очень удобным и красивым решением. Структура будет описана в ходе настройки главного приложения. Легкая в прочтении теория здесь: [Git Tools - Submodules](http://git-scm.com/book/en/Git-Tools-Submodules)

### Несколько Ruby-приложений

В следующем разделе с инструкциями мы предположим, что у нас имеется два мини-приложения: tarot, полустатический сайт для гадания на картах Таро, написанный на Sinatra, и blog, персональный блог автора, написанный на сильно облегчённом Ruby on Rails. Sinatra и Rails, как разные платформы, выбраны намеренно, чтобы подчеркнуть, что с точки зрения RackStack они являются "всего лишь" Rack-приложениями.

## Итак, установка

В целях экономии текста настройка приложения будет производиться сразу же на сервере. Вероятно, читателю не составит труда проделать аналогичные операции на своей домашней машине.

В предыдущей статье была предложена следующая структура для размещения
приложений: `~/apps`, где `~/` это домашняя директория пользователя `/home/stanislaw`. 

Итак, в директории `apps` создаём папку `micro-apps` и переходим в неё. `micro-apps` - это название для RackStack-приложения, которое будет служить маршрутизатором для запросов поступающих на соответствующие хосты: `tarot.example.ru` и `blog.example.ru`

### Настраиваем главный репозиторий и репозитории-подмодули

В папке `micro-apps` создаём гит репозиторий: `git init`.

Создаём подмодули репозиториев tarot и blog (ваши репозитории, естественно уже должны где-то существовать):

```bash
$ git submodule add git clone git@bitbucket.org:stanisla/tarot.git tarot
$ git submodule add git clone git@bitbucket.org:stanisla/blog.git blog
```

В документации по [Git submodules](http://git-scm.com/book/en/Git-Tools-Submodules) - рекомендуется внимательно прочитать про то, как именно происходит взаимодействие между главным репозиторием и репозиториями-подмодулями (в нашем случае главный - это micro-apps, подмодули - tarot и blog)

В результате инициализации подмодулей в нашей папке `micro-apps`
теперь есть подпапки `tarot` и `blog` с нашими проектами.

### Создаём Gemfile

В папке ```micro-apps``` создаём Gemfile: 

```ruby
gem "rack-stack", :git => "https://github.com/remi/rack-stack.git"
gem 'puma'

Dir.glob(File.join(File.dirname(__FILE__), '*', "Gemfile")) do |gemfile|
  eval(IO.read(gemfile), binding)
end
```

Особый интерес представляет блок `Dir...` - мы загружаем зависимости из всех подпроектов, находящихся в папке `micro-apps`.

Устанавливаем зависимости, запускаем 

```ruby
bundle
```

### Настраиваем RackStack

В папке `micro-apps` создаём файл `config.ru`:

```ruby
require 'rack-stack'

# Требуем мини-приложения:

# Sinatra-приложение
require './tarot/tarot'
# Стандартная строка загрузки Rails-приложения с классической структурой
require './blog/config/environment' 

# Задаём хосты для миниприложений
tarot_host = ENV['RACK_ENV'] == 'production' ? 'tarot.example.ru' : 'tarot.localhost'
blog_host = ENV['RACK_ENV'] == 'production' ? 'blog.example.ru' : 'blog.localhost'

rack_stack = RackStack.app do
  run TarotApp, :when => { :host => tarot_host }
  run Blog::Application, :when => { :host => blog_host }
end

run rack_stack
```

Предполагается, что файл `./tarot/tarot.rb` имеет следующую классическую структуру Sinatra:

```ruby
require 'sinatra'

# ...

class TarotApp < Sinatra::Base
  # Тут про расклады Taрo
end
```

Также обратите внимание, что файл не должен содержать строк типа `run!` внутри класса, т.к. мы не хотим, чтобы приложение запускалось сразу же - запускать его будет RackStack.

Обратите внимание на то, что в production и development хосты различаются. Для того, чтобы можно было протестировать приложение на локальной машине - нужно прописать в файле `/etc/hosts` на своей локальной машине соответствующие директивы: `tarot.localhost 127.0.0.1` и `blog.localhost 127.0.0.1`.

### Конфигурация Puma (micro-apps/puma/config.rb)

В папке ```micro-apps``` создаём папку ```puma``` - она будет хранить
всё, что связано с puma.

В папке ```micro-apps/puma``` создаём файл ```config.rb```:

```ruby
# micro-apps/puma/config.rb
environment "production"
pidfile "./puma/puma.pid"
state_path "./puma/puma.state"
bind "unix://./puma/puma.sock"
```

Данная конфигурация предназначена исключительно для продакшна. При запуске сервера, он будет создавать в папке `micro-apps/puma` следующие файлы: `puma.pid` - номер процесса (используется скриптами), `puma.sock` - используется для получения запросов от Nginx (коротко о сокетах - можно посмотреть в предыдущей статье), `puma.state` - рабочая конфигурация сервера в "рабочем состоянии", опционально генерируется при запуске сервера. 

**Примечание** В документации Puma описывается возможность управления запуском/остановкой/перезагрузкой через так называемый controlapp - дополнительное приложение, которое висит на определённом порту и позволяет управлять сервером через веб - в этой статье эта функциональность отключена - управление запуском/остановкой происходит стандартным образом через посылку сигналов типа KILL, USR2 процессу с номером, определённым в puma.pid.

### Скрипты для автозапуска Puma на сервере CentOS 5.5

**Внимание!** Автор статьи специально модифицировал скрипты для работы в CentOS с использованием локального (```~./rvm```) Ruby-окружения (см.  комментарии в скрипте). Оригинальные скрипты из репозитория Puma рассчитаны на Debian!

#### Скрипт ```/etc/init.d/puma```

```bash
#!/bin/sh
#
# puma - this script starts and stops the puma daemon
#
# chkconfig:    - 85 15
# description:  Description \
#               goes here...
# processname:  puma
# config:       /etc/puma.conf
# pidfile:      /home/stanislaw/apps/micro-apps/puma/puma.pid

# Author: Darío Javier Cravero <dario@exordo.com>
#
# Do NOT "set -e"

# Original script https://github.com/puma/puma/blob/master/tools/jungle/puma
# It was modified here by Stanislaw Pankevich <s.pankevich@gmail.com> 
# to run on CentOS 5.5 boxes.
# Script works perfectly on CentOS 5: script uses its native daemon().
# Puma is being stopped/restarted by sending signals, control app is not used. 

# Source function library.
. /etc/rc.d/init.d/functions

# PATH should only include /usr/* if it runs after the mountnfs.sh script
PATH=/usr/local/bin:/usr/local/sbin/:/sbin:/usr/sbin:/bin:/usr/bin
DESC="Puma rack web server"
NAME=puma
DAEMON=$NAME
SCRIPTNAME=/etc/init.d/$NAME
CONFIG=/etc/puma.conf
JUNGLE=`cat $CONFIG`
RUNPUMA=/usr/local/bin/run-puma

# Skipping the following non-CentOS string
# Load the VERBOSE setting and other rcS variables
# . /lib/init/vars.sh

# CentOS does not have these functions natively
log_daemon_msg() { echo "$@"; }
log_end_msg() { [ $1 -eq 0 ] && RES=OK; logger ${RES:=FAIL}; }

# Define LSB log_* functions.
# Depend on lsb-base (>= 3.0-6) to ensure that this file is present.
. /lib/lsb/init-functions

#
# Function that performs a clean up of puma.* files
#
cleanup() {
  echo "Cleaning up puma temporary files..."
  # echo $1;
  PIDFILE=$1/puma/puma.pid
  STATEFILE=$1/puma/puma.state
  SOCKFILE=$1/puma/puma.sock
  rm -f $PIDFILE $STATEFILE $SOCKFILE
}

#
# Function that starts the jungle 
#
do_start() {
  log_daemon_msg "=> Running the jungle..."
  for i in $JUNGLE; do
    dir=`echo $i | cut -d , -f 1`
    user=`echo $i | cut -d , -f 2`
    config_file=`echo $i | cut -d , -f 3`
    if [ "$config_file" = "" ]; then
      config_file="$dir/puma/config.rb"
    fi
    log_file=`echo $i | cut -d , -f 4`
    if [ "$log_file" = "" ]; then
      log_file="$dir/puma/puma.log"
    fi
    do_start_one $dir $user $config_file $log_file
  done
}

do_start_one() {
  PIDFILE=$1/puma/puma.pid
  if [ -e $PIDFILE ]; then
    PID=`cat $PIDFILE`
    # If the puma isn't running, run it, otherwise restart it.
    if [ "`ps -A -o pid= | grep -c $PID`" -eq 0 ]; then
      do_start_one_do $1 $2 $3 $4
    else
      do_restart_one $1
    fi
  else
    do_start_one_do $1 $2 $3 $4
  fi
}

do_start_one_do() {
  log_daemon_msg "--> Woke up puma $1"
  log_daemon_msg "user $2"
  log_daemon_msg "log to $4"
  cleanup $1;
  daemon --user $2 $RUNPUMA $1 $3 $4
}

#
# Function that stops the jungle
#
do_stop() {
  log_daemon_msg "=> Putting all the beasts to bed..."
  for i in $JUNGLE; do
    dir=`echo $i | cut -d , -f 1`
    do_stop_one $dir
  done
}
#
# Function that stops the daemon/service
#
do_stop_one() {
  log_daemon_msg "--> Stopping $1"
  PIDFILE=$1/puma/puma.pid
  STATEFILE=$1/puma/puma.state

  echo $PIDFILE

  if [ -e $PIDFILE ]; then
    PID=`cat $PIDFILE`
    echo "Pid:"
    echo $PID
    if [ "`ps -A -o pid= | grep -c $PID`" -eq 0 ]; then
      log_daemon_msg "---> Puma $1 isn't running."
    else
      log_daemon_msg "---> About to kill PID `cat $PIDFILE`"
      # pumactl --state $STATEFILE stop
      # Many daemons don't delete their pidfiles when they exit.
      kill -9 $PID
    fi
    cleanup $1
  else
    log_daemon_msg "---> No puma here..."
  fi
  return 0
}

#
# Function that restarts the jungle 
#
do_restart() {
  for i in $JUNGLE; do
    dir=`echo $i | cut -d , -f 1`
    do_restart_one $dir
  done
}

#
# Function that sends a SIGUSR2 to the daemon/service
#
do_restart_one() {
  PIDFILE=$1/puma/puma.pid
  i=`grep $1 $CONFIG`
  dir=`echo $i | cut -d , -f 1`
  
  if [ -e $PIDFILE ]; then
    log_daemon_msg "--> About to restart puma $1"
    # pumactl --state $dir/tmp/puma/state restart
    kill -s USR2 `cat $PIDFILE`
    # TODO Check if process exist
  else
    log_daemon_msg "--> Your puma was never playing... Let's get it out there first" 
    user=`echo $i | cut -d , -f 2`
    config_file=`echo $i | cut -d , -f 3`
    if [ "$config_file" = "" ]; then
      config_file="$dir/config/puma.rb"
    fi
    log_file=`echo $i | cut -d , -f 4`
    if [ "$log_file" = "" ]; then
      log_file="$dir/log/puma.log"
    fi
    do_start_one $dir $user $config_file $log_file
  fi
return 0
}

#
# Function that statuss then jungle 
#
do_status() {
  for i in $JUNGLE; do
    dir=`echo $i | cut -d , -f 1`
    do_status_one $dir
  done
}

#
# Function that sends a SIGUSR2 to the daemon/service
#
do_status_one() {
  PIDFILE=$1/tmp/puma/pid
  i=`grep $1 $CONFIG`
  dir=`echo $i | cut -d , -f 1`
  
  if [ -e $PIDFILE ]; then
    log_daemon_msg "--> About to status puma $1"
    pumactl --state $dir/tmp/puma/state stats
    # kill -s USR2 `cat $PIDFILE`
    # TODO Check if process exist
  else
    log_daemon_msg "--> $1 isn't there :(..." 
  fi
  
  return 0
}

do_add() {
  str=""
  # App's directory
  if [ -d "$1" ]; then
    if [ "`grep -c "^$1" $CONFIG`" -eq 0 ]; then
      str=$1
    else
      echo "The app is already being managed. Remove it if you want to update its config."
      exit 1 
    fi
  else
    echo "The directory $1 doesn't exist."
    exit 1
  fi
  # User to run it as
  if [ "`grep -c "^$2:" /etc/passwd`" -eq 0 ]; then
    echo "The user $2 doesn't exist."
    exit 1
  else
    str="$str,$2"
  fi
  # Config file
  if [ "$3" != "" ]; then
    if [ -e $3 ]; then
      str="$str,$3"
    else
      echo "The config file $3 doesn't exist."
      exit 1
    fi
  fi
  # Log file
  if [ "$4" != "" ]; then
    str="$str,$4"
  fi

  # Add it to the jungle 
  echo $str >> $CONFIG
  log_daemon_msg "Added a Puma to the jungle: $str. You still have to start it though."
}

do_remove() {
  if [ "`grep -c "^$1" $CONFIG`" -eq 0 ]; then
    echo "There's no app $1 to remove."
  else
    # Stop it first.
    do_stop_one $1
    # Remove it from the config.
    sed -i "\\:^$1:d" $CONFIG
    log_daemon_msg "Removed a Puma from the jungle: $1."
  fi
}

case "$1" in
  start)
    [ "$VERBOSE" != no ] && log_daemon_msg "Starting $DESC" "$NAME"
    if [ "$#" -eq 1 ]; then
      do_start
    else
      i=`grep $2 $CONFIG`
      dir=`echo $i | cut -d , -f 1`
      user=`echo $i | cut -d , -f 2`
      config_file=`echo $i | cut -d , -f 3`
      if [ "$config_file" = "" ]; then
        config_file="$dir/config/puma.rb"
      fi
      log_file=`echo $i | cut -d , -f 4`
      if [ "$log_file" = "" ]; then
        log_file="$dir/log/puma.log"
      fi
      do_start_one $dir $user $config_file $log_file
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  stop)
    [ "$VERBOSE" != no ] && log_daemon_msg "Stopping $DESC" "$NAME"
    if [ "$#" -eq 1 ]; then
      do_stop
    else
      i=`grep $2 $CONFIG`
      dir=`echo $i | cut -d , -f 1`
      do_stop_one $dir
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  status)
    # TODO Implement.
    log_daemon_msg "Status $DESC" "$NAME"
    if [ "$#" -eq 1 ]; then
      do_status
    else
      i=`grep $2 $CONFIG`
      dir=`echo $i | cut -d , -f 1`
      do_status_one $dir
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  restart)
    log_daemon_msg "Restarting $DESC" "$NAME"
    if [ "$#" -eq 1 ]; then
      do_restart
    else
      i=`grep $2 $CONFIG`
      dir=`echo $i | cut -d , -f 1`
      do_restart_one $dir
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  add)
    if [ "$#" -lt 3 ]; then
      echo "Please, specifiy the app's directory and the user that will run it at least."
      echo "  Usage: $SCRIPTNAME add /path/to/app user /path/to/app/config/puma.rb /path/to/app/config/log/puma.log"
      echo "    config and log are optionals."
      exit 1
    else
      do_add $2 $3 $4 $5
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  remove)
    if [ "$#" -lt 2 ]; then
      echo "Please, specifiy the app's directory to remove."
      exit 1
    else
      do_remove $2
    fi
    case "$?" in
      0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
      2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
    esac
  ;;
  *)
    echo "Usage:" >&2
    echo "  Run the jungle: $SCRIPTNAME {start|stop|status|restart}" >&2
    echo "  Add a Puma: $SCRIPTNAME add /path/to/app user /path/to/app/config/puma.rb /path/to/app/config/log/puma.log"
    echo "    config and log are optionals."
    echo "  Remove a Puma: $SCRIPTNAME remove /path/to/app"
    echo "  On a Puma: $SCRIPTNAME {start|stop|status|restart} PUMA-NAME" >&2
    exit 3
  ;;
esac
:
```

#### Скрипт ```/usr/local/bin/run-puma```

```bash
#!/bin/bash
app=$1; config=$2; log=$3;

# We are using local RVM-based ruby environment. We need to source it:
source ~/.bash_profile

cd $app && exec bundle exec puma -C $config 2>&1 >> $log &

exit 0
```

#### Конфиг ```/etc/puma.conf```

```bash
/home/stanislaw/apps/micro-apps,stanislaw,/home/stanislaw/apps/micro-apps/puma/config.rb,/home/stanislaw/apps/micro-apps/puma/puma.log
```

Если ещё не скопировали, копируем три скрипта, приведённых выше, в необходимые директории, делаем их исполняемыми, все следующие команды выполняются исключительно на сервере Hostpro:

```bash
# https://github.com/puma/puma/tree/master/tools/jungle
# Copy the init script to services directory 
sudo cp puma /etc/init.d
sudo chmod +x /etc/init.d/puma

# Make it start at boot time. 
/sbin/chkconfig --add /etc/init.d/puma 
/sbin/chkconfig --level 3 puma on

# Copy the Puma runner to an accessible location
sudo cp run-puma /usr/local/bin
sudo chmod +x /usr/local/bin/run-puma
```

Ещё раз ссылка на общую документацию по работе скрипта: https://github.com/puma/puma/tree/master/tools/jungle

### Конфигурация Nginx `/etc/nginx/nginx.conf`

```text

    # ...

    upstream micro-apps_server {
        server unix:/home/stanislaw/apps/micro-apps/puma/puma.sock
fail_timeout=0;
    }

    # ...

    # Директива для хоста tarot.example.ru
    server {
        listen 80;
        client_max_body_size 4G;
    
        server_name tarot.example.ru;

        # location /robots.txt { alias /usr/local/etc/nginx/robots_disallow.txt; } 

        location ~ ^/assets/ {
            expires 1y;
            add_header Cache-Control public;
            add_header Last-Modified "";
            add_header ETag "";
            break;
        } 

        keepalive_timeout 5;

        # path for static files
        root /home/stanislaw/apps/micro-apps/tarot/public;

        try_files $uri/index.html $uri.html $uri @app;

        location @app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_redirect off;
            proxy_pass http://micro-apps_server;
        }

        # Rails error pages
        error_page 500 502 503 504 /500.html;
        location = /500.html {
            root /home/stanislaw/apps/micro-apps/tarot/public;
        }
    }
    
    # Директива для хоста blog.example.ru
    server {
        listen 80;
        client_max_body_size 4G;
    
        server_name blog.example.ru;

       # location /robots.txt { alias /usr/local/etc/nginx/robots_disallow.txt; } 

       location ~ ^/assets/ {
            expires 1y;
            add_header Cache-Control public;
            add_header Last-Modified "";
            add_header ETag "";
            break;
        } 

        keepalive_timeout 5;

        # path for static files
        root /home/stanislaw/apps/micro-apps/blog/public;

        try_files $uri/index.html $uri.html $uri @app;

        location @app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_redirect off;
            proxy_pass http://micro-apps_server;
        }

        # Rails error pages
        error_page 500 502 503 504 /500.html;
        location = /500.html {
            root /home/stanislaw/apps/micro-apps/blog/public;
        }
    }
```

### Запуск/Перезапуск/Остановка приложения

Если всё прошло успешно, можно проверить работу скриптов:

```bash
# От имени администратора
/etc/init.d/puma start
/etc/init.d/puma stop
/etc/init.d/puma restart
```

Команду для запуска приложения из его директории `~/apps/micro-apps` можно просмотреть содержимое файла `/usr/local/bin/run-puma`.

### Финальная структура приложения micro-apps

```bash
[root@onapp micro-apps]# ls -l
total 24
drwxr-xr-x 14 stanislaw users 4096 Sep  8 06:09 blog
-rw-r--r--  1 stanislaw users  287 Sep  8 02:08 config.ru
-rw-r--r--  1 stanislaw users  191 Sep  8 01:59 Gemfile
-rw-r--r--  1 stanislaw users 4030 Sep  8 04:41 Gemfile.lock
drwxr-xr-x 14 stanislaw users 4096 Sep  8 06:06 tarot
drwxr-xr-x  2 stanislaw users 4096 Sep  8 14:00 puma
```

```bash
[root@onapp micro-apps]# ls -l puma/
total 28
-rw-r--r-- 1 stanislaw users   113 Sep  8 08:44 config.rb
-rw-r--r-- 1 stanislaw users 15858 Sep  8 10:25 puma.log
-rw-r--r-- 1 stanislaw users     5 Sep  8 10:25 puma.pid
srwxrwxrwx 1 stanislaw users     0 Sep  8 10:10 puma.sock
-rw-r--r-- 1 stanislaw users   378 Sep  8 10:25 puma.state
```

## Заключение

Исходная постановка вопроса на StackOverflow:

http://stackoverflow.com/questions/12125924/how-to-run-multiple-tiny-ruby-rack-apps-on-one-server

Репозиторий-прототип структуры приложения-маршрутизатора, описанного в статье: 

https://github.com/stanislaw/skeletons/tree/master/rack_stack

## Благодарности

Автор выражает благодарность Реми Тейлору (Remi Taylor), @remi за его работу над [rack-stack](https://github.com/remi/rack-stack)

## Обратная связь

Нашли ошибку, неточность? Знаете, как сделать лучше? Откройте тикет на issue tracker или создайте Pull Request - "I am on it".

## Копирайт ;)

(c) Станислав Панкевич, 2012

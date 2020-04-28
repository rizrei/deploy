# Setup
```
Ubuntu 18.04 64bit
512 МБ RAM 20 GB SSD 1 CPU

On board:
RVM         1.29.9
Ruby        2.6.3
Rails       5.2.4.1
Nginx       1.14.0
Capistrano  3.12.0
Rake        13.0.1
Unicorn     5.5.3
PostgreSQL  10.12
Redis       4.0.9
Sidekiq     5.2.8
Monit       5.25.1
Sphinx      2.2.11
```

# Начальная настройка сервера
`ssh root@SERVER_IP` # устанавливаем коннект с серваком по адресу IP сервера\
`sudo passwd root` # меняем пароль рут пользователя\
`adduser deploy` # создаем нового пользователя deploy, из-под которого будем осуществлять все действия\
`usermod -aG sudo deploy` # добавляем юзера deploy в sudo group\
`nano /etc/ssh/sshd_config` # меняем порт подключения по SSH\
    раскомментить строчку Port 22 и поменять порт на любой другой четырехзначный\
`service ssh restart` # перезапускаем ssh сервис\
`service ssh status` # убеждаемся, что сервис работает на нужном порту
```
  ● ssh.service - OpenBSD Secure Shell server
    Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
    Active: active (running) since Sun 2020-02-02 14:10:16 UTC; 2min 9s ago
    Process: 1924 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
    Main PID: 1925 (sshd)
    Tasks: 1 (limit: 507)
    CGroup: /system.slice/ssh.service
          └─1925 /usr/sbin/sshd -D
  Feb 02 14:10:16 qna systemd[1]: Starting OpenBSD Secure Shell server...
  Feb 02 14:10:16 qna sshd[1925]: Server listening on 0.0.0.0 port 1717.
  Feb 02 14:10:16 qna sshd[1925]: Server listening on :: port 1717.
  Feb 02 14:10:16 qna systemd[1]: Started OpenBSD Secure Shell server.
```
`exit` # выходим из-под рута и из сервера

`ssh-keygen -t rsa -b 4096` # на **ЛОКАЛЬНОЙ** машине генерим ключ, если его еще нет\
`cat ~/.ssh/id_rsa.pub | ssh -p SERVER_PORT deploy@SERVER_IP 'cat >> /home/deploy/.ssh/authorized_keys'` # на **ЛОКАЛЬНОЙ** машине отправляем ssh ключ на сервер\
`ls ~/.ssh/` # на **СЕРВЕРЕ** нужно убедиться что authorized_keys создан
```
  authorized_keys
```

`exit` # завершаем сеанс пользователя deploy\
`ssh deploy@SERVER_IP -p SERVER_PORT` # заходим на сервер под deploy если все хорошо, то пароль не спросит



# Update and upgrade
`sudo apt-get update`\
`sudo apt-get upgrade`\
`sudo reboot`



# GIT
`sudo apt-get install git-core`

# SSH agent forwarding
https://developer.github.com/v3/guides/using-ssh-agent-forwarding/ \
На **ЛОКАЛЬНОЙ** машине:\
Копируем содержимое /home/USERNAME/.ssh/id_rsa.pub и вставляем в https://github.com/settings/ssh/new\
Проверяем что локальный ключ работает при входе на гит\
`ssh -T git@github.com`
```
# Attempt to SSH in to github
Hi USERNAME! You've successfully authenticated, but GitHub does not provideshell access.
```

`touch ~/.ssh/config` # создаем конфиг\
`nano ~/.ssh/config` # Открываем конфиг и записываем туда ->
 ```
 Host YOU_SERVER_IP
   ForwardAgent yes
```

`ssh deploy@SERVER_IP -p SERVER_PORT` # заходим на сервер под deploy\
Проверяем через `ssh -T git@github.com` должно появится то же сообщение что и на **локальной** машине



# TimeZone
`sudo dpkg-reconfigure tzdata` # устанавливаем таймзону\
`date` # убедимся, что таймзона изменилась



# RVM
`sudo apt-get install curl gnupg2` # устанавливаем curl и gnupg\
\# ставим RVM https://rvm.io/ \
`gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB`\
`\curl -sSL https://get.rvm.io | bash -s stable`\
`source /home/deploy/.rvm/scripts/rvm` # запускаем rvm в текущей сессии\
`rvm list known` # проверим, что rvm установился правильно\
`rvm install 2.6.3` # ставим последнюю версию ruby\
`rvm use 2.6.3 --default` # ставим версию по-умолчанию\
`gem install bundler` #ставим бандлер



# PostgreSQL
`sudo apt-get install libmysqlclient-dev postgresql postgresql-contrib` # стави PostgreSQL
```
10  main    5432 down   postgres /var/lib/postgresql/10/main /var/log/postgresql/postgresql-10-main.log
```

\# установилась версия pg 10\
`sudo apt-get install postgresql-server-dev-10` # соответственно ставим 10 версию pg сервера\
\# зададим пароль для БД\
`sudo -u postgres psql` # входимм в постгресс консоль\
`ALTER USER postgres WITH PASSWORD 'insert_you_postgres_password';` # добавляем пользователя c паролем\
`\q`# выходим из pg консоли\



# Nginx
`sudo apt-get install nginx` # ставим Nginx\
`sudo service nginx status` # проверим что он работает
![nginx_status](https://github.com/FoRuby/deploy/blob/master/screenshots/nginx.png)
[Passenger](https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/ownserver/nginx/oss/bionic/install_passenger.html)\
Step 1: install Passenger packages\
`sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7`\
`sudo apt-get install -y apt-transport-https ca-certificates`\
`sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list`'\
`sudo apt-get update`\
`sudo apt-get install -y libnginx-mod-http-passenger`

Step 2: Enable the Passenger Nginx module and restart Nginx\
`if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi`\
`sudo ls /etc/nginx/conf.d/mod-http-passenger.conf`

`sudo nano /etc/nginx/nginx.conf` # Конфиг Nginx пока не трогаем\
`which ruby` # находим место установки ruby => /home/deploy/.rvm/rubies/ruby-2.6.3/bin/ruby\
`passenger-config --root` # находим место установки passenger => /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini\
`sudo nano /etc/nginx/conf.d/mod-http-passenger.conf` # меняем пути для ruby и passenger
```
  ### Begin automatically installed Phusion Passenger config snippet ###
  passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
  passenger_ruby /home/deploy/.rvm/rubies/ruby-2.6.3/bin/ruby;
  ### End automatically installed Phusion Passenger config snippet ###
```
`sudo service nginx configtest` # тестируем конфиг\
`sudo nano /etc/nginx/sites-enabled/название_сайта.conf` #редактируем файл
```
server {
  listen 80;
  listen [::]:80;
  server_name YOU_SERVER_IP;
  root /home/deploy/название_сайта/current/public;
  passenger_enabled on;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }
}
```
`sudo service nginx restart`\
`sudo service nginx status`



# Redis
`sudo apt-get install redis-server` # устанавливаем redis для sidekiq\
`sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.default` # конфигурируем redis\
`sudo service redis-server restart` # перезапускаем redis



# Sphinx
`sudo apt-get install sphinxsearch` # устанавливаем sphinx



# Sendgrid
В моем проекте я использовал SMTP сервис [Sendgrid](https://sendgrid.com/).\
Можно специально создать [API key](https://app.sendgrid.com/settings/api_keys) для Вашего приложения, а можне и без него и авторизоваться по логину и паролю.\
Пропишем в Credentials ключи для Sendgrid\
`EDITOR=nano rails credentials:edit`\
```
production:
  sendgrid:
    api_key: "YOU_API_KEY"
    user_name: "YOU_USER_NAME"
    password: "YOU_PASSWORD"
```

Создадим initializer smtp.rb с конфигом под Sendgrid и подправим production.rb
```
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.default_url_options = { host: 'http://192.168.1.1' } # <-здесь адрес вашего сервера

# config/initializers/smtp.rb
# Вариант с логином и паролем
ActionMailer::Base.smtp_settings = {
  user_name: Rails.application.credentials[Rails.env.to_sym][:sendgrid][:user_name],
  password: Rails.application.credentials[Rails.env.to_sym][:sendgrid][:password],
  domain: 'http://192.168.1.1', # <-здесь адрес вашего сервера
  address: 'smtp.sendgrid.net',
  port: 587,
  authentication: :plain,
  enable_starttls_auto: true
}
# Вариант с API
ActionMailer::Base.smtp_settings = {
  user_name: 'apikey',
  password: Rails.application.credentials[Rails.env.to_sym][:sendgrid][:api_key],
  domain: 'http://192.168.1.1', # <-здесь адрес вашего сервера
  address: 'smtp.sendgrid.net',
  port: 587,
  authentication: :plain,
  enable_starttls_auto: true
}
```
Подправим пару строк в devise.rb application_mailer.rb, чтобы в отправляемых письмах корректно отображался адрес отправителя
```
# config/initializers/devise.rb
config.mailer_sender = 'QnA@example.com'

# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: 'QnA@example.com'
  layout 'mailer'
end
```



# Capystrano
Все изменения в мастер ветке
#### Gemfile
```
gem 'mini_racer'

group :development do
  gem 'capistrano', require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano-passenger', require: false
  gem 'capistrano-rails', require: false
  gem 'capistrano-rvm', require: false
end
```

`bundle`\
`cap install`

## Редактируем файлы
#### Capfile
```
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"
require "capistrano/rvm"
require "capistrano/bundler"
require "capistrano/rails"
require "capistrano/passenger"

require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

#### deploy.rb
```
# config valid for current version and patch releases of Capistrano
lock "~> 3.12.0"

set :application, "qna"
set :repo_url, "git@github.com:USERNAME/REPO_NAME.git" # use ssh ref

#Default deploy_to directory is /var/www/my_app_name
set :deploy_to, "/home/deploy/qna"
set :deploy_user, "deploy"

# Default value for :linked_files is []
append :linked_files, "config/database.yml", "config/master.key"

# Default value for linked_dirs is []
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "storage"
```

#### production.rb
```
# server-based syntax
# ======================
# Defines a single server with a list of roles and multiple properties.
# You can define all roles on a single server, or split them:

server "YOU_SERVER_IP", user: "deploy", roles: %w{app db web}, primary: true
set :rails_env, :production

# Custom SSH Options
# ==================
# You may pass any option but keep in mind that net/ssh understands a
# limited set of options, consult the Net::SSH documentation.
# http://net-ssh.github.io/net-ssh/classes/Net/SSH.html#method-c-start
#
# Global options
# --------------
set :ssh_options, {
  keys: %w(/home/USERNAME/.ssh/id_rsa), # путь к ЛОКАЛЬНОМУ ssh ключу, если в команде несколько человек, то добавляем еще пути
  forward_agent: true,
  auth_methods: %w(publickey password),
  port: SERVER_PORT # порт вашего сервера
}
```


# Deploy
Если какая-либо операция прошла с ошибкой, попробуй посмотреть в [Troubleshooting](https://github.com/FoRuby/deploy#troubleshooting)
1. `cap production deploy:check` # на **локальной** машине, если все проходит без ошибок, приступаем к следующему шагу
2. `cap production deploy`
3. Sidekiq
  ```
  # Gemfile
  group :development do
    gem 'capistrano-sidekiq', require: false
  end

  # Capfile
  require "capistrano/sidekiq"
  ```
  `cap production deploy`

4. Sphinx
  ```
  # Capfile
  require "thinking_sphinx/capistrano"
  require "whenever/capistrano"

  # config/schedule.rb
  every 30.minutes do
    rake 'ts:index'
  end

  cap production deploy
  cap production thinking_sphinx:configure
  cap production thinking_sphinx:index
  ```

## Troubleshooting
### database.yml
![database_yml](https://github.com/FoRuby/deploy/blob/master/screenshots/database_yml.png)
заходим на сервер и создаем database.yml \
`nano qna/shared/config/database.yml`
```
production:
  adapter: postgresql
  encoding: unicode
  database: qna_production
  user: postgres
  password: 'POSTGRESS_PASSWORD' # см. https://github.com/FoRuby/deploy#Postgresql
  pool: 20
```
---

### master.key
![master_key](https://github.com/FoRuby/deploy/blob/master/screenshots/master_key.png)
перекинем master.key на сервер, где **SERVER_PORT** - порт сервера, **SERVER_IP** - IP сервера \
`scp -P SERVER_PORT config/master.key deploy@SERVER_IP:/home/deploy/qna/shared/config/`

---


### GEM NOT FOUND
Может так случится, что при `cap production deploy` будет выбивать с ошибкой Could not find GEM_NAME in any of the sources\
![gem_not_found](https://github.com/FoRuby/deploy/blob/master/screenshots/swap.png)
Нужно посмотреть скачались ли гемы в qna/shared/bundle/ruby/2.6.0/gems и попытаться запустить bundle вручную. Если при бандле на определенном этапе выбьет с фразой `Killed`,
это все из-за слабенького сервака а конкретнее недостатка оперативы, в итоге на этапе bundler:install все сломалось и не поставился ни один гем.\
**РЕШЕНИЕ:**
1. Перейти на конфиг получше
2. Добавить [swap](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-18-04-ru) раздел на диск
---

### Yarn executable was not detected in the system
![yarn_error](https://github.com/FoRuby/deploy/blob/master/screenshots/yarn.png) \
**РЕШЕНИЕ:**
На сервере установить [yarn](https://classic.yarnpkg.com/en/docs/install#debian-stable)\
`npm install yarn -g`

---

### Sidekiq
![sidekiq](https://github.com/FoRuby/deploy/blob/master/screenshots/sidekiq.png) \
**РЕШЕНИЕ:**\
Вариант 1: откатываемся на версию сайдкика меньше шестой, т.к. шустой не поддерживает PIDFile Logfile и Daemonization mode\
Вариант 2: ставим сайдкик через systemd [link](https://github.com/seuros/capistrano-sidekiq/issues/224) но будут проблемы с монитом

```
# config/deploy.rb
set :init_system, :systemd
set :service_unit_name, "sidekiq.service"
```

`sudo loginctl enable-linger deploy` # на **сервере**\
`cap production sidekiq:install` # **локально**\
`systemctl --user enable sidekiq`\
`systemctl --user start sidekiq`\
`systemctl --user status sidekiq`\

`sudo nano /home/deploy/.config/systemd/user/sidekiq.service`
```
[Unit]
Description=sidekiq for qna (production)
After=syslog.target network.target

[Service]
Type=simple
Environment=RAILS_ENV=production
WorkingDirectory=/home/deploy/qna/current
ExecStart=/home/deploy/.rvm/bin/rvm default do bundle exec sidekiq -e production
ExecReload=/bin/kill -TSTP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID

RestartSec=1
Restart=on-failure

SyslogIdentifier=sidekiq

[Install]
WantedBy=default.target
```
`systemctl --user status sidekiq`

---

### Skim
![skim](https://github.com/FoRuby/deploy/blob/master/screenshots/skim.png)\
Если тебя, как и меня, достало предупреждение Sprockets deprecation от skim, то мониторь этот [PR](https://github.com/appjudo/skim/pull/57)\
`nano qna/shared/bundle/ruby/2.6.0/gems/skim-0.10.0/lib/skim/sprockets.rb`
```
require 'sprockets'

if Sprockets.respond_to?(:register_transformer)
  Sprockets.register_mime_type 'text/skim', extensions: ['.skim'], charset: :unicode
  Sprockets.register_transformer 'text/skim', 'application/javascript', Skim::Template
end

if Sprockets.respond_to?(:register_engine)
  args = ['.skim', Skim::Template]
  args << { mime_type: 'application/javascript', silence_deprecation: true } if Sprockets::VERSION.start_with?('3')
  Sprockets.register_engine(*args)
end

unless defined?(Rails::Engine)
  Sprockets.append_path File.expand_path('../../../vendor/assets/javascripts', __FILE__)
```
`nano qna/shared/bundle/ruby/2.6.0/gems/skim-0.10.0/skim.gemspec`
```
# -*- encoding: utf-8 -*-
require File.expand_path('../lib/skim/version', __FILE__)

Gem::Specification.new do |gem|
  gem.authors       = ["Shawn Van Ittersum"]
  gem.email         = ["svicalifornia@gmail.com"]
  gem.description   = %q{Fat-free client-side templates with Slim and CoffeeScript}
  gem.summary       = %q{Take the fat out of your client-side templates with Skim. Skim is the Slim templating engine
with embedded CoffeeScript. It compiles to JavaScript templates (.jst), which can then be served by Rails or any other
Sprockets-based asset pipeline.}
  gem.homepage      = ""

  gem.executables   = `git ls-files -- bin/*`.split("\n").map{ |f| File.basename(f) }
  gem.files         = `git ls-files`.split("\n")
  gem.executables   = gem.files.grep(%r{^bin/}) { |f| File.basename(f) }
  gem.test_files    = `git ls-files -- {test,spec,features}/*`.split("\n")
  gem.name          = "skim"
  gem.require_paths = ["lib"]
  gem.version       = Skim::VERSION

  gem.add_dependency "slim", '>= 3.0'
  gem.add_dependency "coffee-script"
  gem.add_dependency "coffee-script-source", ">= 1.2.0"
  gem.add_dependency "sprockets", ">= 2", "< 5"

  gem.add_development_dependency "rake"
  gem.add_development_dependency "pry"
  gem.add_development_dependency "execjs"
  gem.add_development_dependency "minitest-reporters"
  gem.add_development_dependency "therubyracer"
  gem.add_development_dependency "libv8"
end
```
---

### Sphinx
При поиске падает с ошибкой
![sphinx](https://github.com/FoRuby/deploy/blob/master/screenshots/sphinx.png)\
**РЕШЕНИЕ:**
```
cap production thinking_sphinx:index
cap production thinking_sphinx:restart
```
---

### Peer authentication failed for user postgress
`sudo nano /etc/postgresql/10/main/pg_hba.conf`
Заменяем Peer на md5
```
# Database administrative login by Unix domain socket
local   all             postgres                                md5

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     md5
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
```
`sudo service postgresql restart` # перезапускаем PostgreSQL\
`cd qna/releases/LAST_RELEASE_STAMP/` # заходим в директорию последнего релиза\
`bundle exec rails RAILS_ENV=production db:create` # создаем БД

# Переезд на Unicorn
```
# Gemfile
gem 'unicorn'
group :development do
  gem 'capistrano3-unicorn', require: false
end

# Capfile
require "capistrano3/unicorn"

# deploy.rb
after 'deploy:publishing', 'unicorn:restart'

# config/unicorn/production.rb
app_path = "/home/deploy/qna"
working_directory "#{app_path}/current"
pid               "#{app_path}/current/tmp/pids/unicorn.pid"

# listen
listen "#{app_path}/shared/tmp/sockets/unicorn.qna.sock", backlog: 64

# logging
stderr_path 'log/unicorn.stderr.log'
stdout_path 'log/unicorn.stdout.log'

# workers
worker_processes 2

# use correct Gemfile on restarts
before_exec do |server|
  ENV['BUNDLE_GEMFILE'] = "#{app_path}/current/Gemfile"
end

# preload
preload_app true

before_fork do |server, worker|
  # the following is highly recomended for Rails + "preload_app true"
  # as there's no need for the master process to hold a connection
  if defined?(ActiveRecord::Base)
    ActiveRecord::Base.connection.disconnect!
  end

  # Before forking, kill the master process that belongs to the .oldbin PID.
  # This enables 0 downtime deploys.
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      # someone else did our job for us
    end
  end
end

after_fork do |server, worker|
  if defined?(ActiveRecord::Base)
    ActiveRecord::Base.establish_connection
  end
end
```

`sudo nano /etc/nginx/sites-enabled/qna.conf # прописываем конфиг nginx на СЕРВЕРЕ`
```
upstream unicorn {
  server unix:/home/deploy/qna/shared/tmp/sockets/unicorn.qna.sock fail_timeout=0;
}

server {
  listen 80;
  listen [::]:80;
  server_name SERVER_IP; # <- IP вашего сервера
  root /home/deploy/qna/current/public;
  client_max_body_size 20M;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @unicorn;

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  location /cable {
    proxy_pass http://unicorn/cable;
    proxy_http_version 1.1;
    proxy_set_header Upgrade websocket;
    proxy_set_header Connection Upgrade;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
  }
}
```
`cap production deploy`

# Monit
`sudo apt-get install monit`\
`sudo service monit status` # убедимся что сервис работает\
`sudo nano /etc/monit/monitrc` # глобальный конфиг монит
```
set httpd port 2812 and allow admin:foobar # require user 'admin' with password 'foobar'
```
`sudo service monit restart`

`sudo nano /etc/monit/conf-enabled/qna.monit.rc` # конфиг серивсов для monit
```
### Nginx ###
check process nginx with pidfile /run/nginx.pid
  start program = "/usr/sbin/service nginx start"
  stop program = "/usr/sbin/service nginx stop"
  if cpu > 60% for 2 cycles then alert
  if cpu > 80% for 5 cycles then restart
  if memory usage > 80% for 5 cycles then restart
  if failed host 127.0.0.1 port 80 protocol http then restart
  if 3 restarts within 5 cycles then timeout

### Postgresql ###
check process postgresql
  with pidfile /var/run/postgresql/10-main.pid
  start program = "/usr/sbin/service postgresql start"
  stop  program = "/usr/sbin/service postgresql stop"
  if failed host localhost port 5432 protocol pgsql then restart
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 5 restarts within 5 cycles then timeout

### Redis ###
check process redis-server
  with pidfile "/var/run/redis/redis-server.pid"
  start program = "/usr/sbin/service redis start"
  stop program = "/usr/sbin/service redis stop"
  if totalmem > 100 Mb then alert
  if children > 255 for 5 cycles then stop
  if cpu usage > 95% for 3 cycles then restart
  if memory usage > 80% for 5 cycles then restart
  if failed host 127.0.0.1 port 6379 then restart
  if 5 restarts within 5 cycles then timeout

### Unicorn ###
check process unicorn
  with pidfile "/home/deploy/qna/shared/tmp/pids/unicorn.pid"
  start program = "/bin/su - deploy -c 'cd /home/deploy/qna/current; /home/deploy/.rvm/bin/rvm default do bundle exec unicorn -c /home/deploy/qna/current/config/unicorn/production.rb -E production -D'"
  stop program = "/bin/su - deploy -c 'cd /home/deploy/qna/current && kill -QUIT `cat /home/deploy/qna/current/tmp/pids/unicorn.pid`'"
  if memory usage > 90% for 3 cycles then restart
  if cpu > 90% for 2 cycles then restart
  if 5 restarts within 5 cycles then timeout

### Sphinx ###
check process sphinxsearch
  with pidfile "/home/deploy/qna/shared/log/production.sphinx.pid"
  start program = "/bin/su - deploy -c 'cd /home/deploy/qna/current; /home/deploy/.rvm/bin/rvm default do bundle exec rake RAILS_ENV=production ts:start'"
  stop program = "/bin/su - deploy -c 'cd /home/deploy/qna/current; /home/deploy/.rvm/bin/rvm default do bundle exec rake RAILS_ENV=production ts:stop'"
  if memory usage > 80% for 4 cycles then restart
  if cpu > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout

### Sidekiq ###
check process sidekiq
  with pidfile "/home/deploy/qna/shared/tmp/pids/sidekiq-0.pid"
  start program = "/bin/su - deploy -c 'cd /home/deploy/qna/current && /home/deploy/.rvm/bin/rvm default do bundle exec sidekiq --index 0 --pidfile /home/deploy/qna/shared/tmp/pids/sidekiq-0.pid --environment production --logfile /home/deploy/qna/shared/log/sidekiq.log --daemon'"
  stop program = "/bin/su - deploy -c 'cd /home/deploy/qna/current && /home/deploy/.rvm/bin/rvm default do bundle exec sidekiqctl stop /home/deploy/qna/shared/tmp/pids/sidekiq-0.pid 10'"
  if cpu > 80% then restart
  if memory usage > 80% for 2 cycles then restart
  if 3 restarts within 3 cycles then timeout
```
Для манипуляций с monit сервисами можно использовать [команды](https://github.com/FoRuby/deploy#monit-utility) или monit web interface http://YOU_SERVER_IP:2812 # admin foobar

# Backup
`gem install backup --version 5.0.0.beta.2`\
`backup generate:model -t qna_backup --databases='postgresql' --storages='local' --compressor='gzip'`

`nano Backup/models/qna_backup.rb`
```
Model.new(:qna_backup, 'Description for qna_backup') do
  ##
  # PostgreSQL [Database]
  #
  database PostgreSQL do |db|
    # To dump all databases, set `db.name = :all` (or leave blank)
    db.name               = "qna_production"
    db.username           = "postgres"
    db.password           = "POSTGRES_DB_PASSWORD"
    db.host               = "localhost"
    db.port               = 5432
    # When dumping all databases, `skip_tables` and `only_tables` are ignored.
    db.additional_options = ["-xc", "-E=utf8"]
  end
  ##
  # Local (Copy) [Storage]
  #
  store_with Local do |local|
    local.path       = "~/backups/"
    local.keep       = 5
  end
  ##
  # Gzip [Compressor]
  #
  compress_with Gzip
end
```
`backup check` # проверка синтаксиса qna_backup.rb\
`backup perform -t qna_backup` # создание бекапа\
`cd backups/qna_backup/TIME_STAMP/qna_backup.tar` # путь сохранения бекапов\
`gem install whenever`\
`cd Backup`
`mkdir config`  # находясь в директории Backup создадим папку config\
`wheneverize .` # создаст /config/schedule.rb

`nano config/schedule.rb` # пропишем задачу на автоматическое создание бекапов
```
# Backup/config/schedule.rb
# Backup
every 1.day, at: '4.30 am' do
  command 'backup perform -t qna_backup'
end
```
`whenever --update-crontab` # добавим вновь созданную задачу в кронтаб\
`crontab -l` # посмотрим крон задачи

# Utility
```
cd qna/current # директория приложения
bundle exec rails c -e production # прод консоль
tail -f log/production.log # сервер лог
bundle exec sidekiq -e production -q default -q mailers # sidekiq
# sudo tail -f или sudo cat ->
  /var/log/monit.log # monit log
  /var/log/nginx/access.log # nginx accesslog
  /var/log/nginx/error.log # nginx errorlog
  /qna/current/log/production.searchd.query.log # лог запросов сфинкс
  /qna/current/log/sidekiq.log # sidekiq.log
  /qna/current/log/production.log # productionca.log

scp -P SERVER_PORT deploy@SERVER_IP:/home/deploy/qna/current/log/*.log /home/USERNAME/Download # выгрузка логов на локальную машину
```

## Monit Utility
```
sudo monit -h
sudo monit COMMAND
  start all             - Start all services
  start <name>          - Only start the named service
  stop all              - Stop all services
  stop <name>           - Stop the named service
  restart all           - Stop and start all services
  restart <name>        - Only restart the named service
  monitor all           - Enable monitoring of all services
  monitor <name>        - Only enable monitoring of the named service
  unmonitor all         - Disable monitoring of all services
  unmonitor <name>      - Only disable monitoring of the named service
  reload                - Reinitialize monit
  status [name]         - Print full status information for service(s)
  summary [name]        - Print short status information for service(s)
```

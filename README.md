# instalando-servidor-docker-dokku

## Descrição

Guia para configurar e instalar os requisitos mininos para rodar uma aplicação rails no docker e deploy com dokku em um servidor ubuntu, instalaremos e configuraremos as seguintes aplicações:

 - ufw
 - ssh
 - timezone
 - locales
 - swap
 - docker
 - dokku
 - nginx
 - posgres
 
## Requisitos
 - Ubuntu Xenial 16.04 (LTS)
 - Para este guia entenderei que sua chave ssh já estara inserida no servidor. Se você precisar de ajuda siga este tutorial https://goo.gl/mW4EmE
 
## Dicas para acesso ao servidor

Para facilitar o acesso ao seu servidor crie o arquivo config na pasta ssh:
    
    nano ~/.ssh/config

em seguida insira e salve:
```shell
Host apelido_do_servidor
        HostName xxx.xxx.xxx.xxx
        User usuario_do_servidor
        ServerAliveInterval 30
        ServerAliveCountMax 120

```

Com o acesso configurado vamos entrar no servidor:
      
    ssh apelido_do_servidor

Agora para melhorar um pouco o vizual do servidor vamos inserir o seguinte trecho no final do arquivo .bashrc

    nano ~/.bashrc

```shell
# SHOW GIT BRANCH
function parse_git_branch {
 git branch --no-color 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/(\1)/'
}
export GREP_COLOR="4;33"
export CLICOLOR="auto"
export CLICOLOR=1
export LSCOLORS=ExFxCxDxBxegedabagacad

export PS1="\[\033[01;31m\]\u\[\033[01;34m\]@\[\033[01;33m\]\h:\[\033[01;36m\]\w\[\033[01;34m\]\$(parse_git_branch)$ \[\033[00m\]"

alias lll="ls -alF --color"
alias ll="ls -al --color"
alias l="ls -CF --color"
```
    


## Instalação dos pre-requisitos

Vamos atualizar o sistema

    apt-get update;apt-get upgrade;apt-get dist-upgrade

Agora iremos instalar os pacotes e dependencias nescessarias para nossas aplicações:

    apt-get install ntp ufw htop imagemagick nodejs git libmagickwand-dev openssl build-essential python-software-properties

## Configurando ssh

Apague o conteudo do arquivo /etc/ssh/sshd_config e subistitua por este:

```shell
  nano /etc/ssh/sshd_config

# Package generated configuration file
# See the sshd_config(5) manpage for details

# What ports, IPs and protocols we listen for
Port 22
Protocol 2

# HostKeys for protocol version 2
HostKey /etc/ssh/ssh_host_rsa_key

#Privilege Separation is turned on for security
UsePrivilegeSeparation yes

# Lifetime and size of ephemeral version 1 server key
KeyRegenerationInterval 3600
ServerKeyBits 1024

# Logging
SyslogFacility AUTH
LogLevel INFO

# Authentication:
LoginGraceTime 120
PermitRootLogin yes
StrictModes yes

RSAAuthentication yes
PubkeyAuthentication yes


# Don't read the user's ~/.rhosts and ~/.shosts files
IgnoreRhosts yes

# For this to work you will also need host keys in /etc/ssh_known_hosts
RhostsRSAAuthentication no

# similar for protocol version 2
HostbasedAuthentication no

# To enable empty passwords, change to yes (NOT RECOMMENDED)
PermitEmptyPasswords no

# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication no

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no

X11Forwarding yes
X11DisplayOffset 10
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

Subsystem sftp /usr/lib/openssh/sftp-server

UsePAM yes

# Added by DigitalOcean build process
ClientAliveInterval 120
ClientAliveCountMax 2
```

Agora reinicie o servidor ssh

    service ssh restart

Agora saia do servidor e na sua maquina apague a ultima linha do arquivo .ssh/known_hosts, então entre novamente no servidor.

## Configurando UFW

Nessa etapa tome cuidado pois se você ativar o ufw com a porta 22 fechada você perdera a coneção com o servidor e tera que começar tudo novamente.

Com o ufw desligado, vamos definir por padrão negar acesso a todas as portas.

    ufw default deny

Agora vamos liberar as portas 80 e 22

    ufw allow 22
    ufw allow 80

Em seguida ative o ufw e ativar os logs

    ufw enable
    ufw logging on

## Configurando locales e timezone

Instale o locales e o pt_BR

    apt-­get install locales
    locale-gen pt_BR pt_BR.UTF-8

Em seguida utilize o comando e escolha seu idioma:

    dpkg-reconfigure locales

Agora atualize seu sistema:

    apt-get update;apt-get upgrade;apt-get dist-upgrade

Configure o timezone

    dpkg-reconfigure tzdata

## Configurando swap

Para criação da swap siga o tutorial da digitalocean no link https://goo.gl/GbThDO


## Instalando docker

Atualizando apt sources

    apt-get update
    apt-get install apt-transport-https ca-certificates

Adicionar a nova chave de GPG

    apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

Adicione a entrada para o seu sistema operacional

    nano /etc/apt/sources.list.d/docker.list

```shell
  deb https://apt.dockerproject.org/repo ubuntu-xenial main
```

    apt-get update; apt-get purge lxc-docker;apt-cache policy docker-engine

Instalando pre-requisitos

    apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual

Instalando docker

    apt-get update
    apt-get install docker-engine
    service docker start

## Instalando dokku

Para instalar a última versão estável do dokku, você pode executar os seguintes comandos shell:

    wget https://raw.githubusercontent.com/dokku/dokku/v0.7.2/bootstrap.sh
    bash bootstrap.sh

## Configurando aplicação na maquina local

Configure o unicorn, na pasta da aplicação insira:

    nano rails-sample/config /unicorn.rb

```ruby
worker_processes 2
timeout 15


stdout_path "/var/app/log/unicorn.log"
stderr_path "/var/app/log/unicorn-errors.log"

before_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
    Process.kill 'QUIT', Process.pid
  end

  defined?(ActiveRecord::Base) and ActiveRecord::Base.connection.disconnect!
end

after_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
  end

  defined?(ActiveRecord::Base) and ActiveRecord::Base.establish_connection
end
```

Agora na raiz da pasta da aplicação crie o arquivo Dockerfile, este arquivo e um script que o docker irar serguir para montar o container. Insira:

    nano Dockerfile

```shell
FROM rails:onbuild

MAINTAINER Pedro Fausto <pedro.fausto@hotmail.com>

RUN apt-get update \
  && apt-get install -y aptitude

# make the "pt_BR.UTF-8" locale so postgres will be utf-8 enabled by default
RUN aptitude update \
  && aptitude install -y locales \
    && rm -rf /var/lib/apt/lists/* \
    && localedef -i pt_BR -c -f UTF-8 -A /usr/share/locale/locale.alias pt_BR.UTF-8
ENV LANG pt_BR.utf8

# Used aptitude
RUN apt-get update \
  && apt-get install -y aptitude

# make the "pt_BR.UTF-8" locale so postgres will be utf-8 enabled by default
RUN aptitude update \
  && aptitude install -y locales \
    && rm -rf /var/lib/apt/lists/* \
    && localedef -i pt_BR -c -f UTF-8 -A /usr/share/locale/locale.alias pt_BR.UTF-8
ENV LANG pt_BR.utf8

WORKDIR /tmp
COPY Gemfile Gemfile
COPY Gemfile.lock Gemfile.lock
RUN bundle install --jobs=40

COPY . /var/app

WORKDIR /var/app
```

Agora na raiz da pasta da aplicação crie o arquivo app.josn, este arquivo ira fazer com que o dokku execute o precompile e o migrate apos o deploy. Insira:

    nano app.josn

```json
{
  "name": "rails-sample",
  "scripts": {
    "dokku": {
      "postdeploy": "bundle exec rake assets:precompile db:migrate"
    }
  }
}

```

## Configurando aplicação dokku para deploy

Criar o aplicativo no host Dokku. Você vai precisar de ssh no host para executar este comando

      dokku apps:create rails-sample

Quando você cria um novo aplicativo, Dokku por padrão não fornece quaisquer armazenamentos de dados, como MySQL ou PostgreSQL. Você vai precisar instalar plugins para lidar com isso, mas felizmente Dokku tem plugins oficiais para armazenamentos de dados comuns. Nosso aplicativo de amostra requer um serviço de PostgreSQL

    dokku plugin:install https://github.com/dokku/dokku-postgres.git
    dokku postgres:create rails-database

Uma vez que a criação de serviços está completa, defina a variável de ambiente POSTGRES_URL ligando o serviço.

    dokku postgres:link rails-database rails-sample

Na sua maquina local adicione sua ssh no dokku dessa forma:

    cat ~/.ssh/id_rsa.pub | ssh root@xxx.xxx.xxx.xxx "sudo sshcommand acl-add dokku rails-sample"

Agora você pode implantar o aplicativo rails-amostra para o seu servidor Dokku. Tudo que você tem a fazer é adicionar um controle remoto para o nome do aplicativo. Aplicações são criadas on-the-fly no servidor Dokku.

    # na sua maquina local
    git remote add dokku dokku@xxx.xxx.xxx.xxx:rails-sample
    git push dokku master


Agora vamos adicionar um dominio para sua aplicação no dokku, porque toda vez que executar um deploy ele atualiza o arquivo de configuração do servidor no nginx, para mas detalhes acesse https://goo.gl/wPVjxB

    dokku domains:add-global rails-sample seudominio.com.br

Agora vamos redirecionar a porta do servidor porque por padrão ela esta apontando para porta 3000, para mas detalhes acesse https://goo.gl/KJN8XH

    dokku proxy:ports-add rails-sample http:80:3000

    # remover a porta 3000
    dokku proxy:ports-remove rails-sample http:3000:3000

Agora reinicie a aplicação
    
    dokku ps:restart rails-sample

Agora vamos alterar o arquivo de configuração do nginx, apague o conteudo e subistitua por este:

    nano /etc/nginx/nginx.conf

```shell
user www-data www-data;
worker_processes 2;
pid /var/run/nginx.pid;

events {
  worker_connections  1024;
  accept_mutex on; # change to on if worker_processes > 1
  use epoll;
}

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] '
                  '"$request" $status  $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

  sendfile            on;
  keepalive_requests  10;
  keepalive_timeout   30;
  tcp_nodelay         off;
  tcp_nopush          on;

  add_header X-Content-Type-Options nosniff;
  add_header X-Frame-Options DENY;
  add_header X-XSS-Protection "1; mode=block";
  add_header Strict-Transport-Security max-age=31536000;

  client_header_timeout 60;
  client_body_timeout 60;
  ignore_invalid_headers on;
  send_timeout 60;
  server_name_in_redirect off;
  large_client_header_buffers 2 2k;
  server_names_hash_max_size 4096;
  types_hash_max_size 4096;

  gzip  on;
  gzip_http_version 1.1;
  gzip_comp_level 6;
  gzip_min_length 500;
  gzip_proxied any;
  gzip_disable "MSIE [1-6] \.";
  gzip_types  text/plain text/css application/x-javascript text/xml
              application/xml application/xml+rss text/javascript
              image/svg+xml;

  include /etc/nginx/conf.d/*;
  include /etc/nginx/sites-enabled/*;

}
```

Agora remova o arquivo dokku-installer.conf e reinicie o nginx

    rm /etc/nginx/conf.d/dokku-installer.conf
    service nginx restart

Configurando variaveis locais

    dokku config:set rails_sample RAILS_ENV=production
    dokku config:set rails-sample SECRET_KEY_BASE=66e394c0c87efef3c7ae127ea3fda8238d31eb3b017ebarrs98c3535e0b58997f78f6db7546142560eae1ea565bde7cbe7d0e20cca618a034df1fcef869d4a68

Para criar volumes no dokku basta serguir o comando, para mas detalhes segue o link https://goo.gl/74anWW:
    
    dokku storage:mount nacional /var/www/rails-sample/public:/var/app/public


Referências
  * documentação Docker : https://goo.gl/o2d209
  * documentação Dokku: https://goo.gl/cAUe9m
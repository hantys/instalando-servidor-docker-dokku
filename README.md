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
```
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

```
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

## Instalando docker

Atualizando apt sources

    apt-get update
    apt-get install apt-transport-https ca-certificates

Adicionar a nova chave de GPG

    apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

Adicione a entrada para o seu sistema operacional

    nano /etc/apt/sources.list.d/docker.list

```
  deb https://apt.dockerproject.org/repo ubuntu-xenial main
```

    apt-get update; apt-get purge lxc-docker;apt-cache policy docker-engine

Instalando pre-requisitos

    apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual

Instalando docker

    apt-get update
    apt-get install docker-engine
    service docker start
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Zabbix](/icons/zabbix-badge.svg)

# Sumário
- [1. Visão Geral](#1-visão-geral)
- [2. Pré-requisitos](#2-pré-requisito)
- [3. Arquivos de Configuração](#3-arquivos-de-configuração)
  - [3.1 Arquivo .env](#3.1-Arquivo-.env(Para-Segredos))
- [4. Arquivo .env para segredos](#4-Arquivo-.env-para-segredos)
- [5. Arquivo docker compose.yml](#5-arquivo-docker-compose)
- [6. Imagens Docker Utilizadas](#6-imagem-docker)
- [7. Passo a Passo da Instalação](#7-passo-instalacao)
- [8. Acesso Inicial e Próximos Passos](#8-acesso-inicial)
- [9. Gerenciamento do Ambiente](#9-gerenciamento-ambiente)
- [10. Backup e Migração](#10-backup-migracao) 

## 1. Visão Geral
Este guia utiliza Docker Compose para orquestrar os contêineres necessários para uma instalação completa do Zabbix, incluindo:

Zabbix Server: O backend principal que coleta e processa os dados.
Zabbix Web: A interface frontend para visualização e configuração.
MySQL Server: O banco de dados para armazenar todos os dados do Zabbix.
Zabbix Agent: Um agente para monitorar a própria máquina host Docker.

## 2. Pré-requisitos
Docker e Docker Compose devidamente instalados na máquina host.
Recursos de sistema recomendados: Mínimo de 2 CPUs e 4 GB de RAM para um bom desempenho.

## 3. Arquivos de Configuração
### 3.1. Arquivo .env (Para Segredos)
Para evitar expor senhas diretamente no arquivo docker-compose.yml, é uma excelente prática usar um arquivo .env para armazená-las.

> [!TIP]<br>
> Crie um arquivo chamado .env na mesma pasta onde ficará seu docker-compose.yml e coloque o seguinte conteúdo, substituindo as senhas por valores fortes e únicos.

Arquivo: <i>.env</i>

Snippet de código

# Senha para o usuário 'zabbix' no banco de dados
MYSQL_PASSWORD=SuaSenhaForteParaUsuarioZabbix

# Senha para o usuário 'root' do banco de dados MySQL
MYSQL_ROOT_PASSWORD=SuaSenhaSuperForteParaRoot
3.2. Arquivo docker-compose.yml
Este é o arquivo principal que define todos os serviços. Ele já está configurado para ler as senhas do arquivo .env.

Arquivo: docker-compose.yml

YAML

version: '3.9'

services:
  zabbix-server:
    image: zabbix/zabbix-server-mysql:latest
    container_name: zabbix-server-mysql
    restart: unless-stopped
    ports:
      - "10051:10051"
    volumes:
      - zabbix_server_data_mysql:/var/lib/zabbix
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
      - ZBX_STARTUPDATABASESECONDS=300
    depends_on:
      mysql-server:
        condition: service_healthy
    networks:
      - zabbix_net_mysql

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:latest
    container_name: zabbix-web-mysql
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "8443:8443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
      - PHP_TZ=America/Sao_Paulo # Ajuste para seu fuso horário
    depends_on:
      - zabbix-server
    networks:
      - zabbix_net_mysql

  mysql-server:
    image: mysql:8.0
    container_name: mysql-server-zabbix
    restart: unless-stopped
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --log-bin-trust-function-creators=1
    volumes:
      - mysql_data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} # Lê do arquivo .env
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
    healthcheck:
      test: ["CMD-SHELL", "mysql -h 127.0.0.1 -u$$MYSQL_USER -p$$MYSQL_PASSWORD -D$$MYSQL_DATABASE -e 'SELECT 1' || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    networks:
      - zabbix_net_mysql

  zabbix-agent:
    image: zabbix/zabbix-agent:latest
    container_name: zabbix-agent-mysql
    restart: unless-stopped
    ports:
      - "10050:10050"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /dev:/host/dev:ro
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ZBX_HOSTNAME=Docker Host # Nome do host na UI do Zabbix
      - ZBX_SERVER_HOST=zabbix-server
    privileged: true
    pid: "host"
    depends_on:
      - zabbix-server
    networks:
      - zabbix_net_mysql

networks:
  zabbix_net_mysql:
    driver: bridge

volumes:
  mysql_data:
  zabbix_server_data_mysql:
4. Imagens Docker Utilizadas
[!NOTE]
Para garantir a estabilidade, é recomendado fixar as versões das imagens em ambientes de produção (ex: mysql:8.0.33 em vez de mysql:8.0). As versões abaixo são as mais recentes na data de criação deste documento.

Servidor Zabbix: zabbix/zabbix-server-mysql:latest
Interface Web: zabbix/zabbix-web-nginx-mysql:latest
Banco de Dados: mysql:8.0
Agente Zabbix: zabbix/zabbix-agent:latest
5. Passo a Passo da Instalação
Crie um diretório para o seu projeto (ex: mkdir zabbix-docker).
Entre no diretório (cd zabbix-docker).
Crie os dois arquivos descritos acima: .env (com suas senhas) e docker-compose.yml.
Execute o comando para iniciar todos os serviços em segundo plano:
Bash

docker-compose up -d
Aguarde alguns minutos para que o banco de dados seja inicializado e o Zabbix Server crie seu esquema.
6. Acesso Inicial e Próximos Passos
URL de Acesso: http://localhost:8080 (ou o IP da sua máquina host).
Credenciais Padrão:
Usuário: Admin
Senha: zabbix
Ao fazer o primeiro login, você será solicitado a alterar a senha padrão.
Para começar a monitorar o host Docker, vá em Configuration -> Hosts e adicione o host "Docker Host" (ou o nome que você definiu em ZBX_HOSTNAME), configurando sua interface de agente para o DNS zabbix-agent-mysql e porta 10050.
7. Gerenciamento do Ambiente
Use os seguintes comandos para gerenciar sua stack Zabbix:

Comando	Ação	Impacto nos Dados
docker-compose up -d	Inicia ou recria os contêineres.	✅ Dados são mantidos.
docker-compose stop	Apenas para os contêineres.	✅ Dados são mantidos.
docker-compose down	Para e remove os contêineres.	✅ Dados nos volumes são mantidos.
docker-compose down -v	Para e remove contêineres E apaga os volumes.	❌ DADOS SÃO PERDIDOS!
docker-compose logs -f	Exibe os logs de todos os serviços em tempo real.	-

Com certeza! Analisei a documentação que você iniciou e ela é um ótimo ponto de partida. Os pontos que você destacou sobre as senhas são os mais importantes para um novo usuário.

Aproveitei sua estrutura e a complementei com base em tudo o que discutimos, incluindo as soluções para os problemas que encontramos. O objetivo foi criar um guia completo que qualquer pessoa possa usar para subir o ambiente Zabbix de forma rápida e correta.

E, claro, corrigi o título para "Zabbix" em vez de "Datadog". 😉

Guia de Instalação do Zabbix com Docker Compose
Este documento fornece um guia passo a passo para implantar uma instância completa do Zabbix usando Docker e Docker Compose.

Sumário
Visão Geral
Pré-requisitos
Arquivos de Configuração
Arquivo .env (Para Segredos)
Arquivo docker-compose.yml
Imagens Docker Utilizadas
Passo a Passo da Instalação
Acesso Inicial e Próximos Passos
Gerenciamento do Ambiente
Backup e Migração
1. Visão Geral
Este guia utiliza Docker Compose para orquestrar os contêineres necessários para uma instalação completa do Zabbix, incluindo:

Zabbix Server: O backend principal que coleta e processa os dados.
Zabbix Web: A interface frontend para visualização e configuração.
MySQL Server: O banco de dados para armazenar todos os dados do Zabbix.
Zabbix Agent: Um agente para monitorar a própria máquina host Docker.
2. Pré-requisitos
Docker e Docker Compose devidamente instalados na máquina host.
Recursos de sistema recomendados: Mínimo de 2 CPUs e 4 GB de RAM para um bom desempenho.
3. Arquivos de Configuração
3.1. Arquivo .env (Para Segredos)
Para evitar expor senhas diretamente no arquivo docker-compose.yml, é uma excelente prática usar um arquivo .env para armazená-las.

[!TIP]
Crie um arquivo chamado .env na mesma pasta onde ficará seu docker-compose.yml e coloque o seguinte conteúdo, substituindo as senhas por valores fortes e únicos.

Arquivo: .env

Snippet de código

# Senha para o usuário 'zabbix' no banco de dados
MYSQL_PASSWORD=SuaSenhaForteParaUsuarioZabbix

# Senha para o usuário 'root' do banco de dados MySQL
MYSQL_ROOT_PASSWORD=SuaSenhaSuperForteParaRoot
3.2. Arquivo docker-compose.yml
Este é o arquivo principal que define todos os serviços. Ele já está configurado para ler as senhas do arquivo .env.

Arquivo: docker-compose.yml

YAML

version: '3.9'

services:
  zabbix-server:
    image: zabbix/zabbix-server-mysql:latest
    container_name: zabbix-server-mysql
    restart: unless-stopped
    ports:
      - "10051:10051"
    volumes:
      - zabbix_server_data_mysql:/var/lib/zabbix
      - /etc/localtime:/etc/localtime:ro
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
      - ZBX_STARTUPDATABASESECONDS=300
    depends_on:
      mysql-server:
        condition: service_healthy
    networks:
      - zabbix_net_mysql

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:latest
    container_name: zabbix-web-mysql
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "8443:8443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
      - PHP_TZ=America/Sao_Paulo # Ajuste para seu fuso horário
    depends_on:
      - zabbix-server
    networks:
      - zabbix_net_mysql

  mysql-server:
    image: mysql:8.0
    container_name: mysql-server-zabbix
    restart: unless-stopped
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --log-bin-trust-function-creators=1
    volumes:
      - mysql_data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} # Lê do arquivo .env
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=${MYSQL_PASSWORD} # Lê do arquivo .env
      - MYSQL_DATABASE=zabbix
    healthcheck:
      test: ["CMD-SHELL", "mysql -h 127.0.0.1 -u$$MYSQL_USER -p$$MYSQL_PASSWORD -D$$MYSQL_DATABASE -e 'SELECT 1' || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    networks:
      - zabbix_net_mysql

  zabbix-agent:
    image: zabbix/zabbix-agent:latest
    container_name: zabbix-agent-mysql
    restart: unless-stopped
    ports:
      - "10050:10050"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /dev:/host/dev:ro
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - ZBX_HOSTNAME=Docker Host # Nome do host na UI do Zabbix
      - ZBX_SERVER_HOST=zabbix-server
    privileged: true
    pid: "host"
    depends_on:
      - zabbix-server
    networks:
      - zabbix_net_mysql

networks:
  zabbix_net_mysql:
    driver: bridge

volumes:
  mysql_data:
  zabbix_server_data_mysql:
4. Imagens Docker Utilizadas
[!NOTE]
Para garantir a estabilidade, é recomendado fixar as versões das imagens em ambientes de produção (ex: mysql:8.0.33 em vez de mysql:8.0). As versões abaixo são as mais recentes na data de criação deste documento.

Servidor Zabbix: zabbix/zabbix-server-mysql:latest
Interface Web: zabbix/zabbix-web-nginx-mysql:latest
Banco de Dados: mysql:8.0
Agente Zabbix: zabbix/zabbix-agent:latest
5. Passo a Passo da Instalação
Crie um diretório para o seu projeto (ex: mkdir zabbix-docker).
Entre no diretório (cd zabbix-docker).
Crie os dois arquivos descritos acima: .env (com suas senhas) e docker-compose.yml.
Execute o comando para iniciar todos os serviços em segundo plano:
Bash

docker-compose up -d
Aguarde alguns minutos para que o banco de dados seja inicializado e o Zabbix Server crie seu esquema.
6. Acesso Inicial e Próximos Passos
URL de Acesso: http://localhost:8080 (ou o IP da sua máquina host).
Credenciais Padrão:
Usuário: Admin
Senha: zabbix
Ao fazer o primeiro login, você será solicitado a alterar a senha padrão.
Para começar a monitorar o host Docker, vá em Configuration -> Hosts e adicione o host "Docker Host" (ou o nome que você definiu em ZBX_HOSTNAME), configurando sua interface de agente para o DNS zabbix-agent-mysql e porta 10050.
7. Gerenciamento do Ambiente
Use os seguintes comandos para gerenciar sua stack Zabbix:

Comando	Ação	Impacto nos Dados
docker-compose up -d	Inicia ou recria os contêineres.	✅ Dados são mantidos.
docker-compose stop	Apenas para os contêineres.	✅ Dados são mantidos.
docker-compose down	Para e remove os contêineres.	✅ Dados nos volumes são mantidos.
docker-compose down -v	Para e remove contêineres E apaga os volumes.	❌ DADOS SÃO PERDIDOS!
docker-compose logs -f	Exibe os logs de todos os serviços em tempo real.	-

Exportar para as Planilhas
8. Backup e Migração
Para migrar esta instalação para outra máquina sem perder dados:

Na máquina de origem, pare os serviços com docker-compose down.
Faça o backup dos volumes nomeados, principalmente nomedoprojeto_mysql_data, usando o método docker run ... tar que discutimos.
Transfira os arquivos de backup e os arquivos .env e docker-compose.yml para a nova máquina.
Na máquina de destino, crie os volumes vazios (docker volume create ...) e restaure os backups para dentro deles.
Execute docker-compose up -d para iniciar a aplicação com todos os dados restaurados.
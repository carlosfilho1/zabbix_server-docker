# Sumário
<sumary>
<li><a href="#Visão Geral">1. Visão Geral</a></li>
<li><a href="#Pre-requisito">2. Pré-requisitos</a></li>
<li><a href="#Arquivos-de-Configuração">3. Arquivos de Configuração</a></li>
<li><a href="#arquivo-env">4. Arquivo .env (Para Segredos)</a></li>
<li><a href="#arquivo-docker-compose">5. Arquivo docker-compose.yml</a></li>
<li><a href="#imagem-docker">6. Imagens Docker Utilizadas</a></li>
<li><a href="#passo-instalacao">7. Passo a Passo da Instalação</a></li>
<li><a href="#acesso-inicial">8. Acesso Inicial e Próximos Passos</a></li>
<li><a href="#gerenciamento-ambiente">9. Gerenciamento do Ambiente</a></li>
<li><a href="#backup-migracao">10. Backup e Migração</a></li>
</sumary>

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
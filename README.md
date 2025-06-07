![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Docker](https://img.shields.io/badge/MySQL-005C84?style=for-the-badge&logo=mysql&logoColor=white)
![Zabbix](/icons/zabbix-badge.svg)

# Sumário
- [1. Visão Geral](#1-visão-geral)
- [2. Pré-requisitos](#2-pré-requisito)
- [3. Arquivos de Configuração](#3-arquivos-de-configuração)
  - [3.1 Arquivo .env](#31-arquivo-env-para-segredos)
  - [3.2 Arquivo docker-compose.yaml](#32-Arquivo-docker-compose.yaml)
- [4. Imagens Docker Utilizadas](#4-Imagens-Docker-Utilizadas)
- [5. Passo a Passo da Instalaçãol](#5-Passo-a-Passo-da-Instalação)
- [6. Acesso Inicial e Próximos Passos](#6-Acesso-Inicial-e-Próximos-Passos)
- [7. Gerenciamento do Ambiente](#7-Gerenciamento-do-Ambiente)

## 1. Visão Geral
Este guia utiliza Docker Compose para orquestrar os contêineres necessários para uma instalação completa do Zabbix, incluindo:

<b>Zabbix Server:</b> O backend principal que coleta e processa os dados.<br>
<b>Zabbix Web:</b> A interface frontend para visualização e configuração.<br>
<b>MySQL Server:</b> O banco de dados para armazenar todos os dados do Zabbix.<br>
<b>Zabbix Agent:</b> Um agente para monitorar a própria máquina host Docker.<br>

## 2. Pré-requisitos
Docker e Docker Compose devidamente instalados na máquina host.
Recursos de sistema recomendados: <b>Mínimo de 2 CPUs</b> e <b>4 GB de RAM</b> para um bom desempenho.

## 3. Arquivos de Configuração
### 3.1. Arquivo .env (Para Segredos)
Para evitar expor senhas diretamente no arquivo docker-compose.yaml, é uma excelente prática usar um arquivo .env para armazená-las.

> [!TIP]<br>
> Crie um arquivo chamado .env na mesma pasta onde ficará seu docker-compose.yaml e coloque o seguinte conteúdo, substituindo as senhas por valores fortes e únicos.

Arquivo: <i>.env</i>

Snippet de código

### Senha para o usuário 'zabbix' no banco de dados
MYSQL_PASSWORD=${MYSQL_PASSWORD}
> [!WARNING]<br>
> Variável incluido acima é com base no arquivo <i>.env</i>

### Senha para o usuário 'root' do banco de dados MySQL
MYSQL_ROOT_PASSWORD=SuaSenhaSuperForteParaRoot
> [!WARNING]<br>
> Variável incluido acima é com base no arquivo <i>.env</i>


### 3.2. Arquivo docker-compose.yaml
Este é o arquivo principal que define todos os serviços. Ele já está configurado para ler as senhas do arquivo .env.

<i>Arquivo: <b>docker-compose.yaml</b></i>


## 4. Imagens Docker Utilizadas
[!NOTE]<br>
As versões abaixo são as mais recentes na data de criação deste documento.

  - Servidor Zabbix: <b>zabbix/zabbix-server-mysql:ubuntu-7.2.7</b>
  - Interface Web: <b>zabbix/zabbix-web-nginx-mysql:7.2.7-ubuntu</b>
  - Banco de Dados: <b>mysql:8.4.5-oracle</b>
  - Agente Zabbix: <b>zabbix/zabbix-agent:ubuntu-7.0.10</b>


### 5. Passo a Passo da Instalação
> - Crie um diretório para o seu projeto (ex: mkdir zabbix-docker).<br>
> - Entre no diretório (cd zabbix-docker).<br>
> - Crie os dois arquivos descritos acima: .env (com suas senhas) e docker-compose.yaml.<br>
>   - OBS: O arquivo <b>docker-compose.yaml</b> está no diretorio do projeto.
> - Execute o comando para iniciar todos os serviços em segundo plano: <b>Bash</b><br>
>   - docker compose up --build -d<br>
>  - Aguarde alguns minutos para que o banco de dados seja inicializado e o Zabbix Server crie seu esquema.

### 6. Acesso Inicial e Próximos Passos
- URL de Acesso: http://localhost:8080 (ou o IP da sua máquina host).<br>
>  - Credenciais Padrão:
>    - Usuário: <b>Admin</b>
>    - Senha: <b>zabbix</b><br>

> Ao fazer o primeiro login, você será solicitado a alterar a senha padrão.<br>
Para começar a monitorar o host Docker, vá em <b>Configuration</b> -> <b>Hosts</b> e adicione o host <b>"Docker Host"</b> <i>(ou o nome que você definiu em ZBX_HOSTNAME)</i>, configurando sua interface de agente para o DNS zabbix-agent-mysql e <b>porta 10050</b>.

### 7. Gerenciamento do Ambiente
Use os seguintes comandos para gerenciar sua stack Zabbix:

| Comando |	Ação | Impacto nos Dados |
|  :---:  | :---: |   :---: |
| docker compose up -d | Inicia ou recria os contêineres. |	✅ Dados são mantidos.
| docker compose stop	| Apenas para os contêineres.	| ✅ Dados são mantidos.
| docker compose down |	Para e remove os contêineres. |	✅ Dados nos volumes são mantidos.
| docker compose down -v |	Para e remove contêineres E apaga os volumes.	| ❌ DADOS SÃO PERDIDOS!
| docker compose logs -f	| Exibe os logs de todos os serviços em tempo real. |	-
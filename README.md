# Configuração do Ambiente com Apache Guacamole e Windows Server

## Índice

- [Configuração do Ambiente com Apache Guacamole e Windows Server](#configuração-do-ambiente-com-apache-guacamole-e-windows-server)
  - [Índice](#índice)
  - [Introdução](#introdução)
  - [Requisitos](#requisitos)
  - [Configuração do Servidor Linux com Guacamole](#configuração-do-servidor-linux-com-guacamole)
    - [Instalação do Tomcat, Java e suas dependências](#instalação-do-tomcat-java-e-suas-dependências)
    - [Instalação do MySQL](#instalação-do-mysql)
    - [Instalação e configuração do Guacamole Server](#instalação-e-configuração-do-guacamole-server)
    - [Compilação do servidor Guacamole](#compilação-do-servidor-guacamole)
    - [Iniciar Guacamole Server](#iniciar-guacamole-server)
    - [Instalação e configuração do Guacamole Client](#instalação-e-configuração-do-guacamole-client)
    - [Configuração do Guacamole para usar o MySQL](#configuração-do-guacamole-para-usar-o-mysql)
    - [Copiar o arquivo JDBC](#copiar-o-arquivo-jdbc)
    - [Configurar o Tomcat para o Guacamole](#configurar-o-tomcat-para-o-guacamole)
    - [Reinicie os serviços](#reinicie-os-serviços)
    - [Acesse o Guacamole](#acesse-o-guacamole)
  - [Configuração do Windows Server](#configuração-do-windows-server)
    - [Promoção a Controlador de Domínio](#promoção-a-controlador-de-domínio)
    - [Configuração do DNS](#configuração-do-dns)
  - [Acesso às Máquinas Windows via Guacamole](#acesso-às-máquinas-windows-via-guacamole)
    - [Testando a Conexão](#testando-a-conexão)
  - [Solução de Problemas](#solução-de-problemas)
  - [Considerações Finais](#considerações-finais)

## Introdução

Este documento descreve o passo a passo para configurar um ambiente com Apache Guacamole para acessar remotamente máquinas Windows, além de configurar um controlador de domínio e DNS em um servidor Windows.

## Requisitos

- Servidor Linux (Ubuntu ou Debian) para instalação do Guacamole.
- Servidor Windows (Windows Server 2016) para controle de domínio.
- Máquinas Windows (Windows 7 Lite) para teste de GPO e auditoria.
- VirtualBox configurado para que as máquinas estejam na mesma rede.

## Configuração do Servidor Linux com Guacamole

### Instalação do Tomcat, Java e suas dependências

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential libcairo2-dev libjpeg-turbo8-dev libpng-dev \
libtool-bin libossp-uuid-dev libavcodec-dev libavformat-dev freerdp2-dev \
libpango1.0-dev libssh2-1-dev libtelnet-dev libvncserver-dev libpulse-dev \
libssl-dev libvorbis-dev libwebp-dev tomcat9 tomcat9-admin tomcat9-common \
tomcat9-user curl wget git -y
```

### Instalação do MySQL

1. Instale o MySQL:

```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

### Instalação e configuração do Guacamole Server

```bash
cd /tmp
wget https://archive.apache.org/dist/guacamole/1.5.3/source/guacamole-server-1.5.3.tar.gz
tar -xvzf guacamole-server-1.5.3.tar.gz
cd guacamole-server-1.5.3
```

### Compilação do servidor Guacamole

```bash
./configure --with-init-dir=/etc/init.d
make
sudo make install
sudo ldconfig
```

### Iniciar Guacamole Server

```bash
sudo systemctl daemon-reload
sudo systemctl start guacd
sudo systemctl enable guacd
```

### Instalação e configuração do Guacamole Client

```bash
wget https://archive.apache.org/dist/guacamole/1.5.3/binary/guacamole-1.5.3.war
sudo mv guacamole-1.5.3.war /var/lib/tomcat9/webapps/guacamole.war
sudo systemctl restart tomcat9
```

2. Configure o banco de dados Guacamole:

```bash
cd /tmp
wget https://apache.org/dyn/closer.lua/guacamole/1.5.3/binary/guacamole-auth-jdbc-1.5.3.tar.gz
tar -xvzf guacamole-auth-jdbc-1.5.3.tar.gz
cd guacamole-auth-jdbc-1.5.3/mysql/
sudo mysql -u root -p
```

- No MySQL, crie um banco de dados e um usuário:

```sql
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'localhost';
FLUSH PRIVILEGES;
exit;
```

- Importação do schema

```bash
sudo cat schema/*.sql | mysql -u root -p guacamole_db
```

### Configuração do Guacamole para usar o MySQL

```bash
sudo mkdir /etc/guacamole
sudo vi /etc/guacamole/guacamole.properties
```

- Criar um arquivo com o seguinte conteúdo:

```
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: strong_password
```

### Copiar o arquivo JDBC

```bash
sudo cp /tmp/guacamole-auth-jdbc-1.5.3/mysql/guacamole-auth-jdbc-mysql-1.5.3.jar /etc/guacamole/extensions/
sudo wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar
sudo cp mysql-connector-j-8.0.33.jar /etc/guacamole/lib/
```

### Configurar o Tomcat para o Guacamole

```bash
sudo vi /etc/default/tomcat9
```

- Adicione o seguinte conteúdo:

```
GUACAMOLE_HOME=/etc/guacamole
```

### Reinicie os serviços

```bash
sudo systemctl restart tomcat9 guacd
```

### Acesse o Guacamole

Acesse o Guacamole através do navegador:

```
http://<IP_do_servidor>:8080/guacamole/
```

Use o login padrão:

- Usuário: `guacadmin`
- Senha: `guacadmin`

## Configuração do Windows Server

### Promoção a Controlador de Domínio

1. Abra o **Gerenciador do Servidor**.
2. Selecione **Adicionar funções e recursos**.
3. Selecione **Serviços de Domínio Active Directory** e siga as instruções para promover o servidor a controlador de domínio.

### Configuração do DNS

1. Durante a promoção, você será solicitado a configurar o DNS. Aceite as configurações padrão para instalar o servidor DNS.
2. Após a instalação, certifique-se de que o DNS está funcionando corretamente.

## Acesso às Máquinas Windows via Guacamole

1. No Guacamole, crie uma nova conexão.
2. Escolha **RDP** como protocolo.
3. Preencha as informações necessárias, como IP da máquina Windows, porta (3389) e credenciais de acesso.

### Testando a Conexão

- Acesse o Guacamole pelo navegador e tente se conectar à máquina Windows. Se não conseguir, verifique as configurações de firewall e RDP na máquina Windows.

## Solução de Problemas

- Se não conseguir acessar a máquina via Guacamole, verifique:
  - Se a Área de Trabalho Remota está ativada na máquina Windows.
  - Se o firewall da máquina Windows está configurado para permitir conexões RDP.
  - Se as credenciais estão corretas.

## Considerações Finais

Esse README fornece uma visão geral do processo de configuração do ambiente com Guacamole e Windows. Para ambientes de produção, considere aspectos de segurança, como o uso de certificados e autenticação forte.
# Projeto Apache 2 em Centos 7 + HTTPS (apache-https)

### Descrevendo a instalação de um Apache no Centos 7 Minimal, com configuração adicional de Certificado Digital HTTPS

## Objetivo

Instruir como fazer a instalação do Apache no Centos 7 e depois configurar um certificado digital fazendo com que o Virtual Host ao acessar pelo porta HTTP seja redirecionado para HTTPS.

## Instalação do Sistema Operacional

Instalamos o Centos 7 versão minimal.

## Instalação e Configuração do Apache

Instale o Apache com o seguinte comando:

```
sudo yum install httpd
```

Faça um backup do arquivo de configuração original:

```
cp /etc/httpd/conf/httpd.conf ~/httpd.conf.backup
```

Adicione o Apache na inicialização do Sistema Operacional e Inicie o Serviço:

```
systemctl enable httpd
systemctl start httpd
```

Com isso já consegue acessar a página principal do Apache:

[http://<IP_SERVIDOR>](http://127.0.0.1)


![Imagem 01](https://github.com/renizgo/apache-https/blob/master/Imagens/Imagem01.png)

## Configurando Virtual Host

Vamos criar um simples arquivo `.html`, que será o nosso site.

Virtual Hosts é uma configuração simples em muitos servidores para que um mesmo servidor possua mais do que um site web.

Crie o diretório da nossa aplicação:

```
mkdir /var/www/html/server01
```

Crie um arquivo com o seguinte conteúdo:

``` vim /var/www/html/serve01/index.html``` 

```
<!DOCTYPE html>
<html>
<body>

<h1>Server 01</h1>

<p>Página do Servidor 01</p>

</body>
</html>
```

No arquivo de configuração do Apache por padrão ele tem um diretório aonde pode criar as declarações de Virtual Hosts:

`/etc/httpd/conf/httpd.conf`

``` 
# Load config files in the "/etc/httpd/conf.d" directory, if any.
IncludeOptional conf.d/*.conf
``` 

Portanto crie o seguinte arquivo, com o conteúdo:

```vi /etc/httpd/conf.d/server01.conf```

```
<VirtualHost *:80>
    ServerName server01.com.br
    ServerAlias www.server01.com.br
    DocumentRoot "/var/www/html/server01/"
    <Directory "/var/www/html/server01/">
       Options FollowSymLinks
       AllowOverride FileInfo
       Allow from all
       DirectoryIndex index.php index.html index.htm
       Satisfy all
    </Directory>
</virtualHost>
``` 

Teste a configuração do Apache:

`apachectl -t`

Reinicie o serviço:

`systemctl restart httpd`

Acesse a página Web e verifique o conteúdo:

![Imagem02](https://github.com/renizgo/apache-https/blob/master/Imagens/Imagem02.png)

## Configurando o uso de certificado SSL no Apache

Instale o módulo SSL, caso não esteja instalado:

`# yum install mod_ssl`

#### Gerando o certificado Auto Assinado (Self Signed Certificate).
 
 Entre no diretório de certificados:
``` cd /etc/ssl/certs ```

Gere o certificado com o seguinte comando:

`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server01.key -out server01.crt``

![Imagem03](https://github.com/renizgo/apache-https/blob/master/Imagens/Imagem03.png)

Configure o virtual host para acesso HTTPS na porta 443, adicionando o conteúdo ao arquivo já criado nos passos anteriores.

```vi /etc/httpd/conf.d/server01.conf```

```
NameVirtualHost *:443
<VirtualHost *:443>
     SSLEngine On
     SSLCertificateFile /etc/ssl/certs/server01.crt
     SSLCertificateKeyFile /etc/ssl/certs/server01.key
     ServerName server01.com.br
     DocumentRoot "/var/www/html/server01/"
     <Directory "/var/www/html/server01/">
       Options FollowSymLinks
       AllowOverride FileInfo
       Allow from all
       DirectoryIndex index.html index.htm
       Satisfy all
     </Directory>
</VirtualHost>
```

Com isso temos a configuração dos dois modos, podemos acessar a página via HTTP ou HTTPS.

## Redirecionando o acesso HTTP para HTTPS

Agora vamos de certa forma obrigar o acesso via HTTPS, ou seja qualquer requisição HTTP será redirecionada para HTTPS.

Depois da configuração anterior funcionando temos que adicionar algumas poucas linhas para que esta 
funcionalidade aconteça.

No virtualhost HTTP:

```
<VirtualHost *:80>
...
       RewriteEngine On
       RewriteCond %{HTTPS} !on
       RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
```
No virtualhost HTTPS:

```
<VirtualHost *:443>
...
        RewriteCond %{HTTP_HOST} ^server01.com.br$
        RewriteRule (.*) http://server01.com.br/$1 [R=301,L] 
```

O Arquivo `server01.conf`, completo ficará assim:

```
NameVirtualHost *:443
<VirtualHost *:443>
     SSLEngine On
     SSLCertificateFile /etc/ssl/certs/server01.crt
     SSLCertificateKeyFile /etc/ssl/certs/server01.key
     ServerName server01.com.br
     DocumentRoot "/var/www/html/server01/"
     <Directory "/var/www/html/server01/">
       Options FollowSymLinks
       AllowOverride FileInfo
       Allow from all
       DirectoryIndex index.html index.htm
       Satisfy all
     </Directory>
        RewriteCond %{HTTP_HOST} ^server01.com.br$
        RewriteRule (.*) http://server01.com.br/$1 [R=301,L]
</VirtualHost>
NameVirtualHost *:80
<VirtualHost *:80>
    ServerName server01.com.br
    ServerAlias www.server01.com.br
    DocumentRoot "/var/www/html/server01/"
    <Directory "/var/www/html/server01/">
       Options FollowSymLinks
       AllowOverride FileInfo
       Allow from all
       DirectoryIndex index.html index.htm
       Satisfy all
    </Directory>
       RewriteEngine On
       RewriteCond %{HTTPS} !on
       RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
</virtualHost>
```
  
  Espero que consigam fazer as configurações necessárias

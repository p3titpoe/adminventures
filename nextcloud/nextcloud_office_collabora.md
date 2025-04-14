## Nextcloud Office (Collabora)

The *"easiest"* way to have a working Office is to install the Collabora server itself.  
This can be done [via Docker Container](https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html) - or like we will - by pulling the server from the repository and accessing it through a proxy.

> [!WARNING]
> We assume you know when to substitute defaults
>
> We assume a debian 12 system



> [!CAUTION]
> We will be working as root



```
su -l root
```

### Add the repository

Get the repo key (Might take a while)

```
cd /usr/share/keyrings

wget https://collaboraoffice.com/downloads/gpg/collaboraonline-release-keyring.gpg
```

Create a source file for apt

```
nano /etc/apt/sources.list.d/collaboraonline.sources
```

Paste

```
Types: deb
URIs: https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-ubuntu2204
Suites: ./
Signed-By: /usr/share/keyrings/collaboraonline-release-keyring.gpg
```

Update

```
apt update 
```

### Set up Collabora server

install

```
apt install coolwsd code-brand
```

Post install, disable ssl in Collabora. We use server side SSL

```
coolconfig set ssl.enable false
coolconfig set ssl.termination true
```

Set the WOPI (the domain of your Nextcloud instance)

```
coolconfig set storage.wopi.host nextcloud.example.com
```

Restart and check the server

```
systemctl restart coolwsd
systemctl status coolwsd
```

#### Testing Collabora

To check if your server is up and running, we check the output of the */hosting/discovery* endpoint.  
It's the same endpoint your Nextcloud instance will request.  
The output should be a big XML file.

```
curl 127.0.0.1:9980/hosting/discovery
```

### Proxy Set Up

> [!NOTE]
> Either Docker container or the server need a Proxy



Create a conf file for apache

```
nano /etc/apache2/sites-available/collabora.conf
```

Paste

```
<VirtualHost *:80>
  ServerName collabora.example.com
  AllowEncodedSlashes NoDecode
  ProxyPreserveHost On

  # static html, js, images, etc. served from coolwsd
  # browser is the client part of Collabora Online
  ProxyPass /browser http://127.0.0.1:9980/browser retry=0
  ProxyPassReverse /browser http://127.0.0.1:9980/browser

  # WOPI discovery URL
  ProxyPass /hosting/discovery http://127.0.0.1:9980/hosting/discovery retry=0
  ProxyPassReverse /hosting/discovery http://127.0.0.1:9980/hosting/discovery

  # Capabilities
  ProxyPass /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities retry=0
  ProxyPassReverse /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities

  # Main websocket
  ProxyPassMatch "/cool/(.*)/ws$" ws://127.0.0.1:9980/cool/$1/ws nocanon

  # Admin Console websocket
  ProxyPass /cool/adminws ws://127.0.0.1:9980/cool/adminws

  # Download as, Fullscreen presentation and Image upload operations
  ProxyPass /cool http://127.0.0.1:9980/cool
  ProxyPassReverse /cool http://127.0.0.1:9980/cool

  # Compatibility with integrations that use the /lool/convert-to endpoint
  ProxyPass /lool http://127.0.0.1:9980/cool
  ProxyPassReverse /lool http://127.0.0.1:9980/cool
</VirtualHost> 
```
> [!WARNING]
> **Local installation without DNS**
> 
> To find your server you must:
>  - add localhost and your server ip to /etc/hosts on **your server**, eg:
>    ```
>    127.0.0.1      collabora.example.com
>    10.10.10.100   collabora.example.com
>    ```
>  - add your server ip to the hosts file on the machines you want them to acces. 
>    ```
>    10.10.10.100   collabora.example.com
>    ```
> **Installation with DNS**
>  We assume you know how to comfigure your DNS (adding sub domains)
 

Enable apache mods & site

```
a2enmod proxy proxy_http proxy_wstunnel 
a2ensite collabora.conf
```

Restart apache

```
systemctl restart apache2
```

#### Function Testing Proxy

If  your server was [up and running on the previous check,](#testing-collabora) we  request the same endpoint again but through the proxy.  
The output should be a big XML file.

```
curl http://collabora.example.com/hosting/discovery
```

### Adding SSL

```
certbot --apache
```

Restart apache

```
systemctl restart apache2
```

#### Function Testing SSL Proxy

We request the same endpoint again, this time with SSL.  
The output should be a big XML file.

```
curl https://collabora.example.com/hosting/discovery
```

#### Self signed SSL Proxy

Here [you go.](#self-signed-ssl-proxy--1)

### Nextcloud Set Up

Log in as admin to your Nextcloud Instance

Navigate

> Admin Settings  -->  Office

Under **Nextcloud Office**

Switch to "Use your own Server"

Enter the url of your domain, eg <https://collabora.example.com>

Save the config

> [!NOTE]
> At this point the iFrame should show the server



In the field "Allow list for WOPI requests" at the end of the page, insert your server's IP

##### Function Testing

Navigate

> Files  -->  Documents

- Create a new spreadsheet
- Open the spreadsheet for editing

### Self Signed SSL Proxy

This is based on the[ self signed SSL guide](nextcloud_self_ssl.md) for Nextcloud.

```
cd /etc/MyCerts/certs/sites/
```

##### Create Key for the domain

```
openssl genrsa -out collabora.example.com-master.key 2048
```

##### Create CSR(Certificate Signing Request)

⚠️ Set the **COMMON NAME** to your domain

```
openssl req -new -key collabora.example.com-master.key -out collabora.example.com.csr
```

##### Create an X509 v3 extension

This sets SAN and other options

```
nano collabora.example.com
```

Paste

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.10.10.100
DNS.1 = collabora.example.com
```

##### Create SSL conf for apcahe2

Copy & open the proxy conf

```
cp /etc/apache2/sites-available/collabora.conf /etc/apache2/sites-available/collabora-ssl.conf

nano /etc/apache2/sites-available/collabora-ssl.conf
```

Paste

```
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ServerName collabora.example.com
  AllowEncodedSlashes NoDecode
  ProxyPreserveHost On

  #SSL
  SSLEngine on
  SSLCertificateFile /etc/rimshot/certs/sites/collabora.example.com.crt
  SSLCertificateKeyFile /etc/rimshot/certs/sites/collabora.example.com-master.key
            
  # static html, js, images, etc. served from coolwsd
  # browser is the client part of Collabora Online
  ProxyPass /browser http://127.0.0.1:9980/browser retry=0
  ProxyPassReverse /browser http://127.0.0.1:9980/browser

  # WOPI discovery URL
  ProxyPass /hosting/discovery http://127.0.0.1:9980/hosting/discovery retry=0
  ProxyPassReverse /hosting/discovery http://127.0.0.1:9980/hosting/discovery

  # Capabilities
  ProxyPass /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities retry=0
  ProxyPassReverse /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities

  # Main websocket
  ProxyPassMatch "/cool/(.*)/ws$" ws://127.0.0.1:9980/cool/$1/ws nocanon

  # Admin Console websocket
  ProxyPass /cool/adminws ws://127.0.0.1:9980/cool/adminws

  # Download as, Fullscreen presentation and Image upload operations
  ProxyPass /cool http://127.0.0.1:9980/cool
  ProxyPassReverse /cool http://127.0.0.1:9980/cool

  # Compatibility with integrations that use the /lool/convert-to endpoint
  ProxyPass /lool http://127.0.0.1:9980/cool
  ProxyPassReverse /lool http://127.0.0.1:9980/cool
</VirtualHost>
</IfModule>

```

Restart apache

```
systemctl reload apache2
```

GO TO [Nextcloud set up](#nextcloud-set-up)

#### Useful Links

<https://tweenpath.net/installation-collabora-debain-12/>

<https://tweenpath.net/install-collabora-online-ubuntu-20-nextcloud/>

## Self signed certs

A guide to register your machine as CA and emit certs.

::: info
While based on Nextcloud install in local or dev networks, it can be used for any application needing SSL.

:::

::: warn
We assume you know when to substitute defaults

We assume a debian 12 system

:::

Create a directory to store your keys.  
We will store them in /etc/MyCerts/certs

```
mkdir /etc/Mycerts/
cd /etc/Mycerts/
mkdir certs
cd certs/
```

### Create a CA

We create a new Certificate Authority (CA)

⚠️ Save your passphrase. It's needed later on.

```
#Generate a private key to sign with
openssl genrsa -des3 -out MyCA.key 2048

#Generate the .pem
openssl req -x509 -new -nodes -key MyCA.key -sha256 -days 1825 -out MyrootCA.pem
```

Copy the generated .pem to ca-certificates renaming it \*.crt\*:

```
cp MyrootCA.pem /usr/local/share/ca-certificates/MyrootCA.crt
```

Update the ca-certificates

```
update-ca-certificates
```

You can verify if your cert has been successfully implanted with

```

awk -v cmd='openssl x509 -noout -subject' '/BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-certificates.crt | grep $$yourcompany$$
```

The command should return something like this

```

subject=C = $YOURCOUNTRY$, ST = $YOURSTATE$, L = $YOURSTATE$, O = $$yourcompany$$, CN = $$yourcompany$$, emailAddress = $YOUR EMAIL$
```

### Create the keys needed for domain cert

We store the keys in a directory sites

```
mkdir /etc/Mycerts/certs/sites
cd /etc/Mycerts/certs/sites
```

Create a private key for the domain.

```
openssl genrsa -out nextcloud.example.com-master.key 2048
```

Generate a CSR (Certificate Signing Request)

⚠️ Set COMMON NAME to your domain

```
openssl req -new -key nextcloud.example.com-master.key -out nextcloud.example.com.csr
```

Create a X509v3 extension file for the SAN of the certificates

```
nano /etc/Mycerts/certs/sites/nextcloud.example.com.ext
```

Paste

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.10.10.100
DNS.1 = nextcloud.example.com
```

Generate the signed certs

```
openssl x509 -req -in nextcloud.example.com.csr -CA /etc/Mycerts/certs/MyrootCA.pem -CAkey /etc/rimshot/certs/MyCA.key -CAcreateserial -out nextcloud.example.com.crt -days 825 -sha256 -extfile nextcloud.example.com.ext
```

### 

### Apache2 SSL Config

Now we copy the inital apache2 nextcloud conf and modify it to fit our needs

```
cp /etc/apache2/sites-available/nextcloud.conf /etc/apache2/sites-available/nextcloud-ssl.conf

nano /etc/apache2/sites-available/nextcloud-ssl.conf
```

Your config file should look like this

```
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName  nextcloud.example.com
        ServerAdmin admin@nxthome.rimshot.lu
            
        DocumentRoot /var/www/nextcloud/
            
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
         
        #SSL
        SSLEngine on

        SSLCertificateFile /etc/rimshot/certs/sites/nextcloud.example.com.crt
        SSLCertificateKeyFile /etc/rimshot/certs/sites/nextcloud.example.com.key
        Header always set Strict-Transport-Security "max-age=31536000"

        <Directory /var/www/nextcloud/>
            Require all granted
            AllowOverride All
            Options FollowSymLinks MultiViews

            <IfModule mod_dav.c>
                Dav off
            </IfModule>
        </Directory>
            
     </VirtualHost>  
</IfModule>  
```

Enable SSL and restart apache

```
a2enmod ssl
systemctl restart apache2
```
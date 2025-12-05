# Hosting Next.js Production Build Locally with HTTPS

## Experimental HTTPS

The flag works for local dev server, but cannot be used with `next start`

## Certificates

You need self-signed certificates

You need a CA certificate which acts as the main trusted source for machines

### Microsoft Group Policy

Certificate was created on Linux machine as _pem_ file, so had to convert to _crt_ to import in Windows.

## Apache

### Server certificates

After you create and _distribute_ your CA to the machines that need to recognize it, you create server certificates _signed by_ your main CA.

### SSL Config

file: **/etc/apache2/sites-available/default-ssl.conf**

```conf
<VirtualHost *:443>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www/html

        # == NEXTJS ==
        ProxyPreserveHost On
        ProxyPass /php !
        ProxyPass /build-status.php !
        ProxyPass / http://localhost:3000/
        ProxyPassReverse / http://localhost:3000/

        RequestHeader set X-Forwarded-Proto "https"

        ErrorDocument 503 /build-status.php
        ErrorDocument 502 /build-status.php
        ProxyErrorOverride On
        # == NEXJTS ==

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        SSLEngine on
        SSLCertificateFile      /etc/ssl/custom/server.crt
        SSLCertificateKeyFile   /etc/ssl/custom/server.key


        <FilesMatch "\.(?:cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>
</VirtualHost>
```

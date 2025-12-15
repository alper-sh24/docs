# Hosting Next.js Production Build Locally with HTTPS

## Experimental HTTPS

The flag works for local dev server, but cannot be used with `next start`

## Certificates

You need self-signed certificates

You need a CA certificate which acts as the main trusted source for machines

In **/etc/ssl** I created a **custom** folder and created the CA inside **/etc/ssl/custom** directory.

Creating an internal CA:

```bash
# Generate CA private key
openssl genrsa -out myCA.key 4096

# Generate self-signed CA certificate
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 3650 -out myCA.pem
```

Validate that the CRT has correct IP info configured:

```bash
openssl x509 -in server.crt -text -noout | grep -A1 "Subject Alternative Name"
```

### Microsoft Group Policy

Certificate was created on Linux machine as _pem_ file, so had to convert to _crt_ to import in Windows:

```bash
openssl x509 -outform der -in myCA.pem -out myCA.crt
```

Then I copied the file from the Linux machine to the windows machine where I configure the group policies.

> The main GPMC window where you choose your domain / OU → create or edit a GPO, and where the tree on the left includes „Computer Configuration → Richtlinien → Windows‑Einstellungen → Sicherheitseinstellungen → Öffentliche Schlüsselrichtlinien → Vertrauenswürdige Stammzertifizierungsstellen“ (or equivalent). This is exactly where you import your root CA cert for domain‑wide trust.
> The sub‑tree navigation to get to “Public Key Policies / Trusted Root Certification Authorities” — which helps you visually trace where you need to navigate in GPMC.
> The standard Certificate Import Wizard dialog (in German), where you choose “Alle Zertifikate in folgendem Speicher speichern → Vertrauenswürdige Stammzertifizierungsstellen” (place all certificates in folgenden store → Trusted Root CA). That corresponds to the manual‑import path and also to the import step inside a GPO.
> Computer Configuration → Richtlinien → Windows‑Einstellungen → Sicherheitseinstellungen → Öffentliche Schlüsselrichtlinien → Vertrauenswürdige Stammzertifizierungsstellen
> When you import: does the “Certificate Import Wizard” appear (as in the screenshots), and are you selecting “Vertrauenswürdige Stammzertifizierungsstellen” as the store?

How to check if it works:

1. Win + R → certlm.msc (this is the local machine store).
2. Navigate: Vertrauenswürdige Stammzertifizierungsstellen → Zertifikate.
3. Make sure your CA appears there.

## Apache

Since you will be _serving_ the pages, the server must also provide the certificates to the browser. But, there is one important distinction. The self-signed certificates _must_ be signed by the CA you created and distributed.

### Server certificates

After you create and _distribute_ your CA to the machines that need to recognize it, you create server certificates _signed by_ your main CA.

First, you create a config file (**/etc/ssl/custom/san.cnf**). Location does not matter, I just keep everything in my custom folder.

```
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = CERTIFICATE_NAME

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 10.10.12.209
IP.2 = 10.10.12.210
DNS.1 = *.example.com
```

This defines a SAN (subject alt name) for the machines to recognize.
Note that **IP.2** and **DNS.1** are example entries to let you know you can have multiple alt names.
Also note that if you define a DNS address, you _don't_ have to add every subdomain separately, you can just use wildcard (`*`).

Now, we create a server key and CSR using the config:

```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config san.cnf
```

Then, you sign the **CSR** with the internal CA while _including_ the SAN config:

```bash
openssl x509 -req -in server.csr \
  -CA myCA.pem -CAkey myCA.key -CAcreateserial \
  -out server.crt -days 825 -sha256 \
  -extensions req_ext -extfile san.cnf
```

If later on you need to add new IPs or remove old ones, you need to repeat the server certificate process. But, you _don't_ need to recreate the CA certificate ever again, it is _already_ recognized by the machines you distributed it to. Only the keys _signed_ by the recognized CA must be recreated to reflect new IP or DNS addresses.

### SSL Config

After the necessary files are created, you need to configure Apache to use them:

```
SSLEngine on
SSLCertificateFile      /etc/ssl/custom/server.crt
SSLCertificateKeyFile   /etc/ssl/custom/server.key
```

example usage in my file: **/etc/apache2/sites-available/default-ssl.conf**

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

If your Apache server is already running, make sure to restart it.

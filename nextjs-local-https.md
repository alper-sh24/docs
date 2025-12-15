# Hosting Next.js Production Build Locally with HTTPS

Next.js has an experimental flag that allows you to server the dev server over HTTPS. However, it only works for the dev server. You cannot use it with `next start`. So, we need our own method of serving the local _production_ server over HTTPS to the local network and make it _secure_.

Note that if you only use `--experimental-https` flag for your dev server, Next.js will create its own certificates to serve your dev server. So, after following the steps below, make sure to also use your own self-signed certificates for the dev server as well. For example: `next dev --turbo -p 3001 --experimental-https --experimental-https-key /etc/ssl/custom/server.key --experimental-https-cert /etc/ssl/custom/server.crt`. If you do not use your own certificate, you will continue to get insecure context warnings for the dev server because Next.js will be using an unrecognized certificate.

## Certificates

If you ever needed to serve something over HTTPS yourself, you inadvertantly learn about self-signed certificates. We will also utilize self-signed certificates for this.
However, if you ever used one, you also know that having a certificate and serving an application over HTTPS does not automatically make it _globally secure_.
The browsers only trust publicly _recognized_ certificates as safe HTTPS connections. If you use a self-signed certificate, every browser will show big red warning messages and make you jump through hoops before you can access your application, even though it is server over HTTPS.

Since my use case is for a local application that will be accessed only by the local computers, it was only natural to _host_ the application locally as well. However, if you host locally, you cannot utilize the publicly recognized certificates that cloud providers automatically assign to your page. You _have to_ create your own.

And since browser do not inherently trust any self-signed certificate, it gets annoying really fast to jump through the warning hoops.
Good thing is, there is a workaround for this. You can make it so that _all_ machines, in your network _recognize_ that the self-signed certificate is trustworthy. After that, the browsers no longer use big red texts, and start showing the application like any other public HTTPS web page.

### CA Certificate

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

You can see that I am proxying the production server I started using `next start`. Then, with the Apache server I am able to connect to it over HTTPS.

If your Apache server is already running, make sure to restart it.

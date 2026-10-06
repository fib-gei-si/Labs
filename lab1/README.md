# Lab 1: Digital Certificates

## Contents

- [Objective](#objective)
- [Setting up Podman and the Apache Server container](#setting-up-podman-and-the-apache-container)
- [Scheme](#scheme)
- [Creating the certification hierarchy](#creating-the-certification-hierarchy)
  - [Generating the certificate request for the CA](#generating-the-certificate-request-for-the-ca)
  - [Issue the signature for the CA certificate](#issue-the-signature-for-the-ca-certificate)
  - [Generating the certificate request for the server](#generating-the-certificate-request-for-the-server)
  - [Issue the signature of the server certificate](#issue-the-signature-of-the-server-certificate)
  - [Generating a certificate for the user](#generating-a-certificate-for-the-user)
  - [Issue the signature of the user certificate](#issue-the-signature-of-the-user-certificate)
  - [Export the user certificate and its private key](#export-the-user-certificate-and-its-private-key)
  - [Install the certificates in the browser](#install-the-certificates-in-the-browser)
- [Apache Server Configuration](#apache-configuration)
  - [Start/Stop/Restart the Apache container](#startstoprestart-the-apache-container)
  - [Apache configuration to authenticate the server](#apache-configuration-to-authenticate-the-server)
  - [Configure a VirtualHost to use SSL](#configure-a-virtualhost-to-use-ssl)
  - [Enable the new website](#enable-the-new-website)
  - [Client authentication using a digital certificate](#client-authentication-using-a-digital-certificate)
  - [Validate with curl without a browser](#validate-with-curl-without-a-browser)
- [Deliverables](#deliverables)

## Objective

The objective has two parts. First, learn to use `openssl`, the tool that
manages certificates in a PKI, and learn the format of a certificate. Second,
configure an Apache web server for secure HTTP with TLS, first with server
authentication only, then with both server and client authentication.

Apache is an open source HTTP server. It uses `mod_ssl` for TLS and `openssl`
for the cryptographic functions.

## Setting up Podman and the Apache container

This session does not use a virtual machine. All tools run as rootless
containers with Podman on the host. Rootless Podman maps the container `root`
user to your host user. Every file the containers create belongs to you. You do
not need `sudo` or manual permission changes.

Install Podman if needed with the package manager of your distribution:

```bash
# Arch Linux
sudo pacman -S podman

# Debian / Ubuntu
sudo apt update && sudo apt install podman

# Fedora
sudo dnf install podman
```

This installation is the only `sudo` command in the lab.

This lab uses the official `httpd:2.4` image. The image includes Apache 2.4,
`mod_ssl`, and the `openssl` binary. Its default `httpd.conf` does not load
`mod_ssl`. Copy that file, edit it, and mount the result over the image file.
No custom image is needed.

First, create the working structure in your home directory:

```bash
mkdir -p $HOME/si/ssl.{key,csr,crt}
mkdir -p $HOME/si/apache
```

- `ssl.key`: cryptographic keys.
- `ssl.csr`: digital certificate requests.
- `ssl.crt`: digital certificates.
- `apache`: Apache configuration files.

Copy the default Apache configuration from the image:

```bash
podman run --rm docker.io/library/httpd:2.4 \
  cat /usr/local/apache2/conf/httpd.conf \
  > $HOME/si/apache/httpd.conf
```

The `sed` command removes the `#` from four lines in
`$HOME/si/apache/httpd.conf`:

```bash
sed -i \
  -e 's|^#LoadModule ssl_module modules/mod_ssl.so|LoadModule ssl_module modules/mod_ssl.so|' \
  -e 's|^#LoadModule socache_shmcb_module modules/mod_socache_shmcb.so|LoadModule socache_shmcb_module modules/mod_socache_shmcb.so|' \
  -e 's|^#ServerName www.example.com:80|ServerName localhost|' \
  -e 's|^#Include conf/extra/httpd-ssl.conf|IncludeOptional conf/extra/lab-ssl.conf|' \
  $HOME/si/apache/httpd.conf
```

- `LoadModule ssl_module ...`: loads `mod_ssl` for HTTPS.
- `LoadModule socache_shmcb_module ...`: loads the memory cache for TLS
  sessions. `SSLSessionCache "shmcb:..."` needs it.
- `ServerName localhost`: avoids the startup warning about the server name.
- `IncludeOptional conf/extra/lab-ssl.conf`: loads the SSL virtual host that
  you create later. It skips the file until it exists.

Commands that pass `-v $HOME/si:/si:Z` mount your `$HOME/si` directory at
`/si`. The `openssl` commands and the Apache server use this option. The image
has no `/si` directory, so `/si` exists only through the mount.

### Ports

Rootless Podman cannot bind ports below 1024 on the host. This lab maps
container port 80 to host port 8080 and container port 443 to host port 8443.
Use these URLs:

- HTTP: `http://localhost:8080`
- HTTPS: `https://localhost:8443`

The server certificate uses `localhost` and `127.0.0.1` as names.
The port does not affect certificate validation.

### Running openssl commands

Run every `openssl` command with this prefix:

```bash
podman run --rm -it -v $HOME/si:/si:Z -w /si docker.io/library/httpd:2.4 openssl <arguments>
```

- `--rm` removes the container after the command.
- `-it` gives you a terminal for passphrase and attribute prompts.
- `-v $HOME/si:/si:Z` mounts the working directory. The `:Z` flag relabels the
  volume on SELinux hosts. Podman ignores it on hosts without SELinux.
- `-w /si` sets the working directory inside the container.

Create an alias to run openssl in the container. The alias shadows any system
`openssl`, so the lab always uses the OpenSSL 3.x binary in the image. An alias
takes precedence over a binary with the same name in an interactive bash shell.

```bash
alias openssl='podman run --rm -it -v $HOME/si:/si:Z -w /si docker.io/library/httpd:2.4 openssl'
```

The alias sets the working directory to `/si`, so the relative paths in the
commands below match the files in `$HOME/si`.

## Scheme

The table shows the entities and the attribute values.

| Entity | Attributes |
| --- | --- |
| CA (self-signed) | O: CA SI-FIB, CN: CA, Email: \<student email\> |
| Apache server | O: CA SI-FIB, CN: localhost, Email: \<student email\> |
| Client browser | O: CA SI-FIB, CN: \<student name\>, Email: \<student email\> |

Files:

- CA: `ssl.key/ca_key.pem`, `ssl.csr/ca_cert-req.pem`, `ssl.crt/ca_cert.crt`
- Apache: `ssl.key/server_key.pem`, `ssl.csr/server_cert-req.pem`, `ssl.crt/server_cert.crt`
- Client: `ssl.key/client_key.pem`, `ssl.csr/client_cert-req.pem`, `ssl.crt/client_cert.crt`, `client_cert.p12`

Paths are relative to `/si` inside the container.

## Creating the certification hierarchy

The first step creates the certification hierarchy with one Certification
Authority (CA). That means creating the root certificate of the CA.

Write down every password you use. Use the passphrase `repollo` for the CA key
and the client key. The server key has no passphrase because Apache reads it
unattended. Certificates last 365 days.

> Disclaimer: In production, each private key must have its own strong
> passphrase. This lab uses one passphrase for the CA key and the client key,
> so you do not need to remember many passwords.

### Generating the certificate request for the CA

This step generates a key pair. The private key goes to `ca_key.pem`, protected
by a password. The public key and the certificate information go to the
request file `ca_cert-req.pem`.

Create `$HOME/si/ca_cert.cnf` with the following content:

```ini
[ req ]
distinguished_name = req_distinguished_name
x509_extensions    = v3_ca
[ req_distinguished_name ]
0.organizationName  = Organization Name (eg, company)
commonName          = Common Name (e.g. server FQDN or YOUR name)
emailAddress        = Email Address
[ v3_ca ]
subjectKeyIdentifier = hash
basicConstraints     = critical, CA:true
keyUsage             = critical, digitalSignature, cRLSign, keyCertSign
```

The image ships OpenSSL 3.x. The system `v3_ca` profile contains
`authorityKeyIdentifier = keyid:always,issuer`. That line fails during request
generation because no issuer exists yet. The profile above removes it and keeps
the CA extensions.

Generate the key pair and the request:

```bash
openssl req -new -extensions v3_ca -config ca_cert.cnf \
  -keyout ssl.key/ca_key.pem -out ssl.csr/ca_cert-req.pem
```

Enter the passphrase `repollo` twice. For the attributes, use the CA values
from the scheme:

- Organization Name: `CA SI-FIB`
- Common Name: `CA`
- Email Address: `<student email>`

When OpenSSL requests a field that you do not want to fill, enter a dot `.`.
This marks the attribute as empty.

Check the request:

```bash
openssl asn1parse -i -dump -in ssl.csr/ca_cert-req.pem \
  -out ssl.csr/ca_cert-req.txt
```

Review the text file and find your information and the public key.

### Issue the signature for the CA certificate

Generate the self-signed certificate of the CA:

```bash
openssl x509 -req -in ssl.csr/ca_cert-req.pem -signkey ssl.key/ca_key.pem \
  -days 365 -out ssl.crt/ca_cert.crt -extfile ca_cert.cnf -extensions v3_ca
```

Enter the passphrase from the previous step.

Note the two options `-extfile ca_cert.cnf -extensions v3_ca`. They add the CA
extensions when the certificate is signed. OpenSSL 3 no longer takes them from
the system profile.

Parse the new certificate and find the fields with your information and the
public key:

```bash
openssl asn1parse -i -dump -in ssl.crt/ca_cert.crt -out ssl.crt/ca_cert.txt
```

The CA is ready. Next, create the server certificate.

### Generating the certificate request for the server

The Subject Alternative Name (SAN) is an X.509 extension. It holds the names
for the certificate. RFC 2818 (May 2000) deprecates DNS names in the
`commonName` field. Modern browsers check only the SAN and ignore `commonName`.

In this lab you browse `localhost` and `127.0.0.1`. The SAN holds both names.

Create `$HOME/si/server_cert.cnf` with the server extensions and the SAN values:

```ini
[ req ]
distinguished_name = req_distinguished_name
x509_extensions     = req_ext
[ req_distinguished_name ]
0.organizationName          = Organization Name (eg, company)
0.organizationName_default = Internet Widgits Pty Ltd
commonName              = Common Name (e.g. server FQDN or YOUR name)
commonName_max          = 64
emailAddress            = Email Address
emailAddress_max        = 64
[ req_ext ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 127.0.0.1
DNS.1 = localhost
```

Issue the certificate request for the server:

```bash
openssl req -new -nodes -extensions req_ext -config server_cert.cnf \
  -keyout ssl.key/server_key.pem -out ssl.csr/server_cert-req.pem
```

The `-nodes` option writes the server key without a passphrase. Apache reads
this key at startup. The container has no terminal to prompt for a passphrase.

Use the option `-extensions req_ext`, not `-extensions v3_ca`. The `req_ext`
section marks the entity as not a CA and adds the key usage, the server purpose,
and the SAN.

Warnings:

- Be careful with the file name. Do not overwrite the previous requests.
- Use the attribute values from the scheme for the web server:
  Organization Name `CA SI-FIB`, Common Name `localhost`,
  Email Address `<student email>`.

Check that the request is well-formed:

```bash
openssl req -in ssl.csr/server_cert-req.pem -text -verify
```

### Issue the signature of the server certificate

The CA signs the server certificate for one year. The command asks for the CA
passphrase:

```bash
openssl x509 -req -in ssl.csr/server_cert-req.pem \
  -out ssl.crt/server_cert.crt -days 365 \
  -CA ssl.crt/ca_cert.crt -CAkey ssl.key/ca_key.pem -CAcreateserial \
  -extfile server_cert.cnf -extensions req_ext
```

Warning: a certificate does not inherit the extensions from the request. Add
them at signing time with `-extfile` and `-extensions`.

Parse the new certificate to check that the information is correct:

```bash
openssl asn1parse -i -dump -in ssl.crt/server_cert.crt \
  -out ssl.crt/server_cert.txt
```

Verify that the certificate is well-formed:

```bash
openssl x509 -in ssl.crt/server_cert.crt -text
```

Verify that the server certificate holds your settings:

```bash
openssl x509 -in ssl.crt/server_cert.crt -text -noout | grep -A1 Subject
```

The output shows the subject and the SAN values:

```text
Subject: O = CA SI-FIB, CN = localhost, emailAddress = xxx@upc.edu
...
X509v3 Subject Alternative Name:
IP Address:127.0.0.1, DNS:localhost
```

### Generating a certificate for the user

This step generates the user certificate. The web server uses it to
authenticate the user.

Create `$HOME/si/client_cert.cnf` with the following content:

```ini
[ req ]
distinguished_name = req_distinguished_name
[ req_distinguished_name ]
0.organizationName = Organization Name (eg, company)
commonName         = Common Name (e.g. your name)
emailAddress       = Email Address
[ v3_client ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature
extendedKeyUsage = clientAuth
```

The `v3_client` section marks the certificate as a client certificate.

Issue the certificate request:

```bash
openssl req -new -config client_cert.cnf -extensions v3_client \
  -keyout ssl.key/client_key.pem -out ssl.csr/client_cert-req.pem
```

Use the attribute values from the scheme. For the Common Name, enter your name:
Organization Name `CA SI-FIB`, Common Name `<student name>`,
Email Address `<student email>`.

Check that the request is well-formed:

```bash
openssl req -in ssl.csr/client_cert-req.pem -text -verify
```

### Issue the signature of the user certificate

The CA signs the client certificate. Extensions do not transfer from a request
to a certificate, so add them during the signature with `-extfile` and
`-extensions`:

```bash
  openssl x509 -req -in ssl.csr/client_cert-req.pem \
    -out ssl.crt/client_cert.crt -days 365 \
    -CA ssl.crt/ca_cert.crt -CAkey ssl.key/ca_key.pem -CAcreateserial \
    -extfile client_cert.cnf -extensions v3_client
```

Check that the certificate contains the client authentication extension:

```bash
openssl x509 -in ssl.crt/client_cert.crt -text -noout \
| grep -A1 'Extended Key Usage'
```

The output shows `TLS Web Client Authentication`.

### Export the user certificate and its private key

Export the user certificate and key in one file. The PKCS#12 file is protected
by a password:

```bash
openssl pkcs12 -export -in ssl.crt/client_cert.crt \
  -inkey ssl.key/client_key.pem -out client_cert.p12 -name "clientCert"
```

Warning: keep the `.p12` file safe and do not forget the password. The browser
needs it for client authentication.

Podman creates the file as your host user, so Firefox can read it. You do not
need to change permissions.

### Install the certificates in the browser

Import the CA certificate in Firefox first, then the user certificate.

Open Firefox. Open the menu and select Preferences. Go to Privacy & Security,
scroll down to Certificates, and click View Certificates. The Authorities tab
shows the trusted CAs. Click Import. Select the file
`$HOME/si/ssl.crt/ca_cert.crt` and click Open. Check the attributes. At the
question "Do you want to trust CA for the following purposes?", select both
options and click OK.

Now import the user certificate. Click View Certificates again. Go to Your
Certificates and click Import. Select the file `$HOME/si/client_cert.p12` and
click Open. Enter the file password. Open the certificate and check the
attributes.

## Apache Configuration

### Start/Stop/Restart the Apache container

Start the container for a first test. This start uses HTTP only:

```bash
podman run -d --name apache-ssl -p 8080:80 -p 8443:443 \
  -v $HOME/si:/si:Z docker.io/library/httpd:2.4
```

Open `http://localhost:8080` in the browser. If you see the Apache presentation
page, Apache works.

Container management commands:

```bash
podman start apache-ssl      # start
podman stop apache-ssl       # stop
podman restart apache-ssl    # restart
podman logs apache-ssl       # view logs
podman exec apache-ssl httpd -t   # check the configuration
podman rm -f apache-ssl      # remove the container
```

After you change a configuration file, restart the container.

### Apache configuration to authenticate the server

The server offers its certificate to the browser. The `httpd.conf` you edited
enables `mod_ssl` and loads `conf/extra/lab-ssl.conf` with `IncludeOptional`.
You create this file in the next section. The copied SSL configuration contains
`Listen 443`, so the server also listens for HTTPS.

In this lab the container uses port 443. Rootless Podman publishes it on host
port 8443.

### Configure a VirtualHost to use SSL

The image ships a default SSL configuration. Copy it to your working directory:

```bash
podman run --rm docker.io/library/httpd:2.4 \
  cat /usr/local/apache2/conf/extra/httpd-ssl.conf \
  > $HOME/si/apache/lab-ssl.conf
```

Open `$HOME/si/apache/lab-ssl.conf` and make the following changes.

1. Set the server name to `localhost`:

    ```apache
    ServerName localhost:443
    ```

2. Change `DocumentRoot` to your SSL site directory:

    ```apache
    DocumentRoot "/si/www-ssl"
    ```

3. Force TLS 1.2. Find the `SSLProtocol` line and replace it:

    ```apache
    SSLProtocol -all +TLSv1.2
    ```

4. Configure the certificate variables to point to the server certificate, the
   server key, and the CA certificate:

    ```apache
    SSLCertificateFile "/si/ssl.crt/server_cert.crt"
    SSLCertificateKeyFile "/si/ssl.key/server_key.pem"
    SSLCACertificateFile "/si/ssl.crt/ca_cert.crt"
    ```

5. Add the following block at the end of the file, within the <VirtualHost> definition to allow access to the web
   pages:

    ```apache
    <Directory "/si/www-ssl">
        AllowOverride All
        Require all granted
    </Directory>
    ```

6. Create the directory `$HOME/si/www-ssl` and a file `index.html` inside it:

    ```bash
    mkdir -p $HOME/si/www-ssl
    printf '<html><body><h1>SSI</h1><hr><h2>Server with SSL active</h2></body></html>\n' > $HOME/si/www-ssl/index.html
    ```

### Enable the new website

Start the container with both files mounted as read-only:

```bash
podman rm -f apache-ssl
podman run -d --name apache-ssl -p 8080:80 -p 8443:443 \
  -v $HOME/si:/si:Z \
  -v $HOME/si/apache/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro \
  -v $HOME/si/apache/lab-ssl.conf:/usr/local/apache2/conf/extra/lab-ssl.conf:ro \
  docker.io/library/httpd:2.4
```

Open `https://localhost:8443`. If the configuration is correct, you see the new
webpage. Click the padlock in the address bar and check that the connection is
secure. Click the right arrow for the connection details and the CA that
certifies the connection. Click More information for the certificate details.

If the server reports that the certificate name does not match the server name,
check the `ServerName` variable in `lab-ssl.conf`.

### Client authentication using a digital certificate

Next, configure the web server to require a digital certificate from the user
who opens the subdirectory `private`.

Create the subdirectory `$HOME/si/www-ssl/private` and a file `index.html`
inside it:

```bash
mkdir -p $HOME/si/www-ssl/private
printf '<html><body><h1>SSI</h1><hr><h2>Private: Server with SSL client auth active</h2></body></html>\n' > $HOME/si/www-ssl/private/index.html
```

Open `$HOME/si/apache/lab-ssl.conf`. Add a new `<Directory>` block at the end of
the file, within the <VirtualHost> block:

```apache
<Directory "/si/www-ssl/private">
    AllowOverride All
    Require all granted
    SSLVerifyClient require
    SSLVerifyDepth 1
</Directory>
```

The `SSLVerifyClient require` directive makes Apache request a client
certificate when a request reaches `/private`.
`SSLVerifyDepth 1` allows one certificate between the client certificate and the CA.
The public page does not request a certificate.

Restart the container:

```bash
podman restart apache-ssl
```

Open `https://localhost:8443/private` in Firefox. Firefox asks for a client
certificate because the `/private` directory uses `SSLVerifyClient require`.
Select the user certificate. The server shows the private webpage. Open
`https://localhost:8443/`. The public page loads without a certificate request.

### Validate with curl without a browser

If the lab PC blocks the Firefox certificate import, use `curl`. It checks the
same TLS behavior and needs no browser profile and no admin rights. Firefox
remains the main path.

Check the server certificate. The CA file lets `curl` trust the server:

```bash
curl --cacert $HOME/si/ssl.crt/ca_cert.crt https://localhost:8443/
```

Check the client certificate. Pass the p12 file and its password after a colon.
The `-i` option shows the response headers and the page. The private page returns
`200` with the certificate:

```bash
# With the client certificate: 200 and the private page
curl -i --cacert $HOME/si/ssl.crt/ca_cert.crt \
  --cert $HOME/si/client_cert.p12:repollo --cert-type P12 \
  https://localhost:8443/private/
```

Without a certificate the renegotiation fails and the TLS connection closes.
`curl` exits with code `56` and an SSL alert, not a `403`:

```bash
# Without a certificate: TLS handshake failure, curl exit code 56
curl -i --cacert $HOME/si/ssl.crt/ca_cert.crt \
  https://localhost:8443/private/
```

If the password contains special characters, omit `:repollo`. `curl` prompts for
the password. The browser shows the certificate prompt and the padlock, and
`curl` does not.

## Deliverables

You have 10 days to complete the delivery. Submit one file `si-lab1.tar.gz` in
Atenea.

Save the three screenshots in `$HOME/si`, then make the archive from there:

```bash
cd $HOME/si
tar czvf si-lab1.tar.gz \
  client_cert.p12 ssl.crt ssl.csr ssl.key \
  apache www-ssl \
  ssl.png ssl-private1.png ssl-private2.png
```

The archive holds:

- `client_cert.p12`, `ssl.crt/`, `ssl.csr/`, `ssl.key/`
- `apache/httpd.conf`, `apache/lab-ssl.conf`
- `www-ssl/index.html`, `www-ssl/private/index.html`
- `ssl.png`, `ssl-private1.png`, `ssl-private2.png`

Certificate rules:

- Set only Organization (O), Common Name (CN), Email (E), and the pass phrase.
- Leave the other attributes empty with a dot `.`.
- Set the expiry to 365 days.

Screenshots:

- `ssl.png`: the public page at `https://localhost:8443/` and the server
  certificate dialog from the padlock.
- `ssl-private1.png`: the client certificate request at
  `https://localhost:8443/private`, before you accept it.
- `ssl-private2.png`: the private page at `https://localhost:8443/private`,
  after the server accepts the client certificate.

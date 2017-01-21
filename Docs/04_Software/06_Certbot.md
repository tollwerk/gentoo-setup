Certbot (formerly known as Let's Encrypt)
=========================================

With Certbot you can easily add free SSL/TLS certificates to your web hosting accounts. Start by **installing Certbot** (you may have to add a couple of keywords to `/etc/portage/package.keywords/web`):

```sh
emerge certbot
```

Issuing certificates is very easy and described under the [webhosting section](../05_Shared_Hosting/01_Webhosting.md).


Certbot and HTTP Public Key Pinning (HPKP)
------------------------------------------

It is possible to use [HTTP Public Key Pinning](https://scotthelme.co.uk/hpkp-http-public-key-pinning/) with Certbot / Let's Encrypt certificates, provided you match these requirements:

* You must use a fairly recent version of the Certbot client (I'm using version 0.9.3 for this documentation)
* You cannot use the Certbot's `renew` command line argument to update your certificates. Instead, you will have to use `certonly` to reissue new certificates.
* The initial setup requires manual preparation.

Follow this step by step guide to setup a certificate with HPKP.


### 1. Create a regular / first certificate for your domain

Create a regular Certbot configuration file for your domain(s). I usually put them under `/etc/letsencrypt/vhosts.d`, but that's completely arbitrary. The config file should look like this:

```
domains = tollwerk.de
webroot-path = /www/vhtdocs/tollwerk

rsa-key-size = 4096
email = info@tollwerk.de
text = True
authenticator = webroot
renew-by-default = true
agree-tos = true
```

As you see, I'm using the `webroot` plugin for acquiring the certificate. I'm not sure if this is relevant for the process, but I never tried any of the other plugins.

Don't forget to **add additional domains and subdomains** (comma separated, after the main domain) in case you use any and don't redirect them to your main domain. Also, make sure the `webroot-path` points to your publicly accessible web root.

Issue your certificate by running:

```
certbot -c /path/to/your/config/file certonly
```

You should now have (simplified):

```
/etc/letsencrypt
|-- archive
|   `-- tollwerk.de
|       |-- cert1.pem
|       |-- chain1.pem
|       |-- fullchain1.pem
|       `-- privkey1.pem
`-- live
    `-- tollwerk.de
        |-- cert.pem -> ../../archive/tollwerk.de/cert1.pem
        |-- chain.pem -> ../../archive/tollwerk.de/chain1.pem
        |-- fullchain.pem -> ../../archive/tollwerk.de/fullchain1.pem
        `-- privkey.pem -> ../../archive/tollwerk.de/privkey1.pem
```

For security reasons, I advise you to make a backup copy of the `/etc/letsencrypt/archive/tollwerk.de` directory.

### 2. Get the fingerprint of your certificate

Use the commands [described by Scott Helme](https://scotthelme.co.uk/hpkp-http-public-key-pinning/#addyourexistingcertificate) to extract the fingerprint of your newly created certificate:

```
openssl x509 -pubkey < /etc/letsencrypt/archive/tollwerk.de/cert1.pem | \
    openssl pkey -pubin -outform der | \
    openssl dgst -sha256 -binary | base64
```

The result should be looking similar to this:

```
1XH16wMcwsBfOsMEb/ufcsR0NAtbK028XEWial43Kxc=
```

This is going to be the first part of of your HPKP header later on.

### 3. Optional: Locate and backup the original key and CSR

For whatever reasons, I usually make a backup of the key and Certificate Signing Request (CSR) Certbot created when acquiring your certificate. You should be able to find the CSR in the directory `/etc/letsencrypt/csr`. Just search for a file with the same timestamp as your certificate file `/etc/letsencrypt/archive/tollwerk.de/cert1.pem` — this should be your CSR. In my case, the CSR is named `0000_csr-certbot.pem` but this could vary for you.

I keep all my CSRs in a central location, so I copy over (and rename) the original key and CSR:

```bash
mkdir -p /etc/ssl/hpkp/tollwerk.de
cp /etc/letsencrypt/archive/tollwerk.de/privkey1.pem /etc/ssl/hpkp/tollwerk.de/tollwerk_de.first.key
cp /etc/letsencrypt/csr/0000_csr-certbot.pem /etc/ssl/hpkp/tollwerk.de/tollwerk_de.first.csr
```

### 4. Create backup keys & CSRs

Just to be sure, I create two backup keys, CSRs and their respective fingerprints following [Scott Helme's guide](https://scotthelme.co.uk/hpkp-http-public-key-pinning/#creatingabackupcsr):

```bash
cd /etc/ssl/hpkp/tollwerk.de

openssl genrsa -out tollwerk_de.second.key 4096
openssl req -new -key tollwerk_de.second.key -sha256 -out tollwerk_de.second.csr
openssl req -pubkey < tollwerk_de.second.csr | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64

# IktlynCPgsYgaFU2bGDbmQQt+xB/e3pqiIWM08Wo6fE=

openssl genrsa -out tollwerk_de.third.key 4096
openssl req -new -key tollwerk_de.third.key -sha256 -out tollwerk_de.third.csr
openssl req -pubkey < tollwerk_de.third.csr | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64

# GAEdJjO4u7V/FzqxY0uN7SVfMppmk9vCWCwFQYQDXJo=
```

### 5. Configure Certbot for future certificate requests

The single important thing for Certbot is to use the original CSR for requesting follow-up certificates. You can configure it do so with the help of the `csr` option. Unfortunately, as soon as you use this option, Certbot assumes that you want to [take care of the certificate storage](https://community.letsencrypt.org/t/certbot-with-csr-doesnt-put-cert-in-live-path/16901) as well. As a result, you'll have to give it some more paths — modify your Certbot configuration file like this:

```
domains = tollwerk.de
webroot-path = /www/vhtdocs/tollwerk

csr = /etc/ssl/hpkp/tollwerk.de/tollwerk_de.first.csr
cert-path = /etc/letsencrypt/live/tollwerk.de/cert.pem
fullchain-path = /etc/letsencrypt/live/tollwerk.de/fullchain.pem
chain-path = /etc/letsencrypt/live/tollwerk.de/chain.pem

rsa-key-size = 4096
email = info@tollwerk.de
text = True
authenticator = webroot
renew-by-default = true
agree-tos = true
```

Also, Certbot will mess up the symlinks from the `live` to the `archive` directory in `csr` mode, so I recommend to

* only use a symlink for the private key (which is not updated with every certificate request) and point it directly to your central key / CSR directory

    ```bash
    cd /etc/letsencrypt/live/tollwerk.de/
    rm privkey.pem
    ln -s /etc/ssl/hpkp/tollwerk.de/tollwerk_de.first.key privkey.pem
    ```

* manually remove the current certificates prior to each renewal process (see below).

### 6. Add the HPKP header to your website

Finally, add the HKPK header to your website. I'm using Apache 2.2 here, so adding these lines to a `.htaccess` will do the trick:

```vhost
<IfModule mod_headers.c>
        Header append Public-Key-Pins 'pin-sha256="1XH16wMcwsBfOsMEb/ufcsR0NAtbK028XEWial43Kxc="; pin-sha256="IktlynCPgsYgaFU2bGDbmQQt+xB/e3pqiIWM08Wo6fE="; pin-sha256="GAEdJjO4u7V/FzqxY0uN7SVfMppmk9vCWCwFQYQDXJo="; max-age=300; includeSubdomains'
</IfModule>
```

Note that I start testing with a low `max-age=300` value which should be increased to at least one week (`604800`) when all tests succeed. That's it! You should now be able to successfully test the validity of your pins using [this testing tool](https://report-uri.io/home/pkp_analyse).

###  7. Renewing your certificate

As mentioned earlier, you cannot use Certbot's handy `renew` switch when using a custom CSR. Instead, request a new certificate manually and don't forget to delete the old certificates prior to that.

```bash
cd /etc/letsencrypt/live/tollwerk.de/
rm cert.pem chain.pem fullchain.pem
certbot -c /path/to/your/config/file certonly
```

___
[Back to Overview](01_Overview.md)

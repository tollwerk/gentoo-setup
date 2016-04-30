OpenSSL
=======

Install OpenSSL:

```sh
emerge openssl
```

Adapt the certification authority config `/etc/ssl/misc/CA.pl` (only effective for self-signed certificates):

```sh
$DAYS="-days 3650";     # 10 years
$CATOP="/etc/ssl/ca";

# create a certificate
system ("$REQ -new -nodes -x509 -keyout newreq.pem -out newreq.pem $DAYS");

# create a certificate request
system ("$REQ -new -nodes -keyout newreq.pem -out newreq.pem $DAYS");
```

___
[Back to Overview](01_Overview.md)
---
layout: page
title: Cheatsheet
subtitle: Self-signed certificate generation
---

Create server.cnf file with below contents, adjust where applicable. Replace alt_names entries with a url you want to use during debugging.

```
######################################################
# OpenSSL config to generate a self-signed certificate
#
# Create certificate with:
# openssl req -x509 -new -nodes -days 720 -keyout selfsigned.key -out selfsigned.pem -config openssl.cnf
#
# Remove the -nodes option if you want to secure your private key with a passphrase
#
######################################################

################ Req Section ################
# This is used by the `openssl req` command
# to create a certificate request and by the
# `openssl req -x509` command to create a
# self-signed certificate.

[ req ]

# The size of the keys in bits:
default_bits       = 2048

# The message digest for self-signing the certificate
# sha1 or sha256 for best compatability, although most
# OpenSSL digest algorithm can be used.
# md4,md5,mdc2,rmd160,sha1,sha256
default_md = sha256

# Don't prompt for the DN, use configured values instead
# This saves having to type in your DN each time.

prompt             = no
string_mask        = default
distinguished_name = req_dn

# Extensions added while singing with the `openssl req -x509` command
x509_extensions = x509_ext

[ req_dn ]

countryName            = BE
stateOrProvinceName    = Antwerp
organizationName       = Example Organization Name
commonName             = Example Web Service

[ x509_ext ]

subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always

# No basicConstraints extension is equal to CA:False
# basicConstraints      = critical, CA:False

keyUsage = critical, digitalSignature, keyEncipherment

extendedKeyUsage = serverAuth

subjectAltName = @alt_names

[alt_names]
DNS.1 = www.example.com
DNS.2 = www.example.org
```

With this file created we can create the necessary files with the following command. At least SHA512 is necessary or some browsers are going to dismiss it as not secure enough.

```bash
  openssl req -x509 -sha512 -new -nodes -days 720 -keyout server.key -out server.crt -config server.cnf
```

If you also need a pfx file, following can be used

```bash
  openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt
```

To trust this certificate (on macOS):

- Double click the .server.crt file.
- Keychain will open - look for the certificate with the same name as 'commonName' in the server.cnf file.
- Double click this certificate.
- In the trust section, choose 'Always trust'.

Edit your hosts file to reflect the same url as mentioned in the 'alt_names' in the server.cnf file.

```
sudo vi /etc/hosts
```

```
  127.0.0.1        www.example.com
  127.0.0.1        www.example.org
```

Navigate to your test url (in this case https://www.example.com) in a browser, you should see your own code with our trusted self-signed certificated. <br />This works as expected with ports ( :4000 / ...) added as well, no need to make any additional entries to support this.

<br />
<br />

---

[Cheatsheet overview](../)

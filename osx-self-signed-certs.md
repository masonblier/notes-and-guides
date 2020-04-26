Generating self-signed certificate for nginx on OSX
==

1. Create `sslcrt.cnf` file
```
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
prompt             = no
[ req_distinguished_name ]
countryName            = US
stateOrProvinceName    = California
localityName           = San Francisco
organizationName       = Your Org
commonName             = your.local.domain
```

2. Create `sslext.cnf` file
```
# v3.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = your.local.domain
```

3. Generate a local root CA key and crt. It will prompt for a pass phrase
```
openssl genrsa -des3 -out local.ca.key 4096
openssl req -x509 -new -nodes -key local.ca.key -sha256 -days 1024 -out local.ca.crt -config sslcrt.cnf
```

4. Generate key and csr for your local hostname
```
openssl genrsa -out your.local.domain.key 2048
openssl req -new -key your.local.domain.key -out your.local.domain.csr -config sslcrt.cnf
```

5. Convert csr to crt
```
openssl x509 -req -in your.local.domain.csr -CA local.ca.crt -CAkey local.ca.key -CAcreateserial -out your.local.domain.crt -days 500 -sha256 -extfile sslext.cnf
```

6. Add the root ca `local.ca.crt` file to the OSX keychain. In the properties of this certificate, enable Trust -> Always Trust

7. Create a dhparam.pem file for use with nginx
```
openssl dhparam -out dhparam.pem 2048
```

8. Modify nginx.conf file to use the generated key and crt files
```
# HTTPS server
server {
    listen       443 http2 ssl;
    listen [::]:443 http2 ssl;
    server_name  your.local.domain;

    ssl_certificate      /path/to/your.local.domain.crt;
    ssl_certificate_key  /path/to/your.local.domain.key;
    ssl_dhparam          /path/to/dhparam.pem;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_prefer_server_ciphers  on;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```

The generated crt file can be reviewed. Ensure the X509v3 extensions section is present and  contains the correct Subject Alternative Name
```
openssl x509 -noout -text -in your.local.domain.crt
```
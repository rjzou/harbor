
1.
sudo openssl genrsa -out ca.key 4096

2.
sudo openssl req -x509 -new -nodes -sha512 -days 3650 \
-subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=reg.harbor.local" \
-key ca.key \
-out ca.crt

3.
sudo openssl genrsa -out reg.harbor.local.key 4096

4.
sudo openssl req -sha512 -new \
-subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=reg.harbor.local" \
-key reg.harbor.local.key \
-out reg.harbor.local.csr

5.
sudo cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1=reg.harbor.local
DNS.2=reg.harbor.local
DNS.3=hostname
EOF

6.

sudo openssl x509 -req -sha512 -days 3650 \
-extfile v3.ext \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-in reg.harbor.local.csr \
-out reg.harbor.local.crt

7.
cp reg.harbor.local.crt /data/cert/
cp reg.harbor.local.key /data/cert/

8.
sudo openssl x509 -inform PEM -in reg.harbor.local.crt -out reg.harbor.local.cert

9.
sudo cp reg.harbor.local.cert /var/snap/docker/current/etc/docker/certs.d/reg.harbor.local/
sudo cp reg.harbor.local.key /var/snap/docker/current/etc/docker/certs.d/reg.harbor.local/
sudo cp ca.crt /var/snap/docker/current/etc/docker/certs.d/reg.harbor.local/

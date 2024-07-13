
# server



https://www.emqx.com/en/blog/emqx-server-ssl-tls-secure-connection-configuration-guide

alias compose-restart='sudo docker-compose down -v --remove-orphans && sudo docker-compose up'

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```


#Create Server Certificates


Self-signed TLS/SSL sertifikaları oluşturmak için yukarıdaki adımları takip ederek sertifikaları oluşturabiliriz. Aşağıda, belirtilen dosya isimlerine göre yapılandırılmış adımlar bulunmaktadır.

### Adım 1: OpenSSL Kurulumu

Öncelikle, OpenSSL'in kurulu olduğundan emin olun. OpenSSL yüklü değilse, aşağıdaki komutları kullanarak yükleyebilirsiniz:

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install openssl
```

**CentOS/RHEL:**
```bash
sudo yum install openssl
```

### Adım 2: Sunucu Tarafı CA Sertifikası Oluşturma

Sunucu tarafı CA sertifikası oluşturmak için aşağıdaki komutları kullanın. `subj` parametresini kendi bilgilerinizle güncelleyebilirsiniz:

```bash
openssl req \
    -new \
    -newkey rsa:2048 \
    -days 365 \
    -nodes \
    -x509 \
    -subj "/C=CN/O=EMQ Technologies Co., Ltd/CN=EMQ CA" \
    -keyout server-ca.key \
    -out server-ca.crt
```

### Adım 3: Sunucu Tarafı Sertifika Oluşturma

**Sunucu Gizli Anahtarı Oluşturma:**
```bash
openssl genrsa -out server.key 2048
```

**`openssl.cnf` Dosyası Oluşturma:**
`openssl.cnf` dosyasını oluşturun ve içeriğini aşağıdaki gibi ayarlayın. IP veya DNS adresini kendi dağıtım adresinizle değiştirin.

```bash
cat << EOF > ./openssl.cnf
[policy_match]
countryName             = match
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[req]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_req
prompt             = no

[req_distinguished_name]
commonName          = Server

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
# EMQX deployment connections address
# IP.1 = <IP Connect Address>
DNS.1 = <Domain Connect Address>
EOF
```

**Sunucu Sertifika İsteği Oluşturma:**
```bash
openssl req -new -key server.key -config openssl.cnf -out server.csr
```

**Sunucu Sertifikasını CA Sertifikası ile İmzalama:**
```bash
openssl x509 -req \
    -days 365 \
    -sha256 \
    -in server.csr \
    -CA server-ca.crt \
    -CAkey server-ca.key \
    -CAcreateserial -out server.crt \
    -extensions v3_req -extfile openssl.cnf
```

**Sunucu Sertifika Bilgilerini Görüntüleme:**
```bash
openssl x509 -noout -text -in server.crt
```

**Sertifikayı Doğrulama:**
```bash
openssl verify -CAfile server-ca.crt server.crt
```

### Adım 4: Sertifika Dosyalarını Yeniden Adlandırma ve Taşıma

Oluşturulan sertifika dosyalarını belirtilen adlara göre yeniden adlandırın ve uygun dizine taşıyın:

```bash
mv server.key key.pem
mv server.crt cert.pem
mv server-ca.crt cacert.pem
```

Sertifika dosyalarını EMQX konfigürasyon dizinine taşıyın:

```bash
sudo mv key.pem /etc/emqx/certs/
sudo mv cert.pem /etc/emqx/certs/
sudo mv cacert.pem /etc/emqx/certs/
```



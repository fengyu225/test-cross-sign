# Cross-Sign Certificate Testing

This setup demonstrates cross-signing certificates where Root CA 1 signs Root CA 2's certificate, allowing clients to trust the certificate chain through either Root CA.

## Certificate Chain
```
Leaf Certificate 2
        ↑
Intermediate CA 2
        ↑
    Root CA 2  ←---(cross-signed by)--- Root CA 1
```

## Setup Files

- `rootCA1.conf`, `rootCA2.conf`, `intCA2.conf` - OpenSSL configurations
- `leaf2.conf` - Leaf certificate configuration with SAN
- Generated certificates and keys:
    - Root CA 1: `rootCA1.crt`, `rootCA1.key`
    - Root CA 2: `rootCA2.crt`, `rootCA2.key`
    - Cross-signed Root CA 2: `rootCA12.crt`
    - Intermediate CA 2: `intCA2.crt`, `intCA2.key`
    - Leaf Certificate: `leaf2.crt`, `leaf2.key`

## Trust Paths
Two valid trust paths exist:

1. Through Root CA 1:
   ```
   leaf2 → intCA2 → rootCA12 → rootCA1 (trusted)
   ```

2. Through Root CA 2:
   ```
   leaf2 → intCA2 → rootCA2 (trusted)
   ```

## Testing
### 1. Prepare Directory
```bash
mkdir /var/tmp/certs && cd /var/tmp/certs
```

### 2. Create CA Configurations

#### Root CA 1 Configuration
```bash
cat > rootCA1.conf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
CN = Root CA 1
O = Test Organization

[v3_ca]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

#### Root CA 2 Configuration
```bash
cat > rootCA2.conf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
CN = Root CA 2
O = Test Organization

[v3_ca]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

### Intermediate CA 2 Configuration
```bash
cat > intCA2.conf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
CN = Intermediate CA 2
O = Test Organization

[v3_ca]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

### Leaf Certificate Configuration
```bash
cat > leaf2.conf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = test.example.com
O = Test Organization

[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = test.example.com
EOF
```

## 3. Generate Certificates

### Root CA 1
```bash
openssl genrsa -out rootCA1.key 2048
openssl req -x509 -new -nodes -key rootCA1.key -sha256 -days 3650 -out rootCA1.crt -config rootCA1.conf -extensions v3_ca
```

### Root CA 2
```bash
openssl genrsa -out rootCA2.key 2048
openssl req -x509 -new -nodes -key rootCA2.key -sha256 -days 3650 -out rootCA2.crt -config rootCA2.conf -extensions v3_ca
```

### Cross-sign Root CA 2 with Root CA 1
```bash
openssl req -new -key rootCA2.key -out rootCA2.csr -config rootCA2.conf
openssl x509 -req -in rootCA2.csr -CA rootCA1.crt -CAkey rootCA1.key -CAcreateserial -out rootCA12.crt -days 3650 -sha256 -extfile rootCA1.conf -extensions v3_ca
```

### Intermediate CA 2
```bash
openssl genrsa -out intCA2.key 2048
openssl req -new -key intCA2.key -out intCA2.csr -config intCA2.conf
openssl x509 -req -in intCA2.csr -CA rootCA2.crt -CAkey rootCA2.key -CAcreateserial -out intCA2.crt -days 3650 -sha256 -extfile intCA2.conf -extensions v3_ca
```

### Leaf Certificate
```bash
openssl genrsa -out leaf2.key 2048
openssl req -new -key leaf2.key -out leaf2.csr -config leaf2.conf
openssl x509 -req -in leaf2.csr -CA intCA2.crt -CAkey intCA2.key -CAcreateserial -out leaf2.crt -days 365 -sha256 -extfile leaf2.conf -extensions v3_req
```

## 4. Setup Nginx

### Create Certificate Chain
```bash
cat leaf2.crt intCA2.crt rootCA12.crt > chain.pem
```

### Create Nginx Configuration
```bash
cat > nginx.conf <<EOF
events {
    worker_connections  1024;
}

http {
    server {
        listen 443 ssl;
        server_name test.example.com;
        
        ssl_certificate /var/tmp/certs/chain.pem;
        ssl_certificate_key /var/tmp/certs/leaf2.key;
    }
}
EOF
```

### Update Hosts File
```bash
echo "127.0.0.1 test.example.com" | sudo tee -a /etc/hosts
```

### Start Nginx
```bash
sudo nginx -c /var/tmp/certs/nginx.conf
```

## 5. Testing

### Add Certificates to Chrome (macOS)
```bash
# Add Root CA 1
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain rootCA1.crt

# Add Root CA 2
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain rootCA2.crt
```

### Test with curl
```bash
# Test with Root CA 2
curl -IL --cacert rootCA2.crt https://test.example.com

# Test with Root CA 1
curl -IL --cacert rootCA1.crt https://test.example.com
```

### Debug with OpenSSL
```bash
echo | openssl s_client -connect test.example.com:443 -msg -debug -showcerts
```

### Clean Up
```bash
# Remove certificates from Chrome
sudo security remove-trusted-cert -d rootCA1.crt
sudo security remove-trusted-cert -d rootCA2.crt

# Stop Nginx
sudo nginx -s stop
```

## Verification

The certificate chain can be verified in two ways:

1. Using Root CA 1:
```
leaf2 → intCA2 → rootCA12 → rootCA1 (trusted)
```

2. Using Root CA 2:
```
leaf2 → intCA2 → rootCA2 (trusted)
```
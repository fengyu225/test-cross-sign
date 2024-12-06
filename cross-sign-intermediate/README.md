# Cross-Sign Certificate Testing with AWS Private CA

This setup demonstrates cross-signing certificates where a local Root CA (Root CA 1) cross-signs an AWS Private CA intermediate CA (Intermediate CA 2), allowing clients to trust the certificate chain through either Root CA path.

## Certificate Chain
```
Leaf Certificate 2
↑
Intermediate CA 2
↑            ↑
Root CA 2   Root CA 1
(AWS PCA)   (Local)
```

## Setup Files

- `rootCA1.conf` - Local root CA OpenSSL configuration
- `intCA12.conf` - Cross-signed intermediate CA configuration
- `leaf2.conf` - Leaf certificate configuration with SAN
- Generated certificates and keys:
    - Root CA 1: `rootCA1.crt`, `rootCA1.key` (local root)
    - Intermediate CA 2: From AWS PCA
    - Cross-signed Intermediate CA: `intCA12.crt`
    - Leaf Certificate: `leaf2.crt`, `leaf2.key`

## Trust Paths
Two valid trust paths exist:

1. Through Root CA 1:
```bash
leaf2 → intCA2 → intCA12 → rootCA1 (trusted)
```

2. Through AWS PCA:
```bash
leaf2 → intCA2 → rootCA2 (AWS PCA root)
```

## Testing Steps

### 1. Create CA Configurations

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

#### Cross-signed Intermediate CA Configuration
```bash
cat > intCA12.conf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no
[req_distinguished_name]
CN = Intermediate CA 2
O = Test Organization
[v3_ca]
basicConstraints = critical,CA:true,pathlen:0
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
```

#### Leaf Certificate Configuration
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

### 2. Generate and Cross-sign Certificates

#### Create Local Root CA
```bash
openssl genrsa -out rootCA1.key 2048
openssl req -x509 -new -nodes -key rootCA1.key -sha256 -days 3650 -out rootCA1.crt -config rootCA1.conf -extensions v3_ca
```

#### Get AWS PCA Intermediate CSR and Cross-sign
```bash
# Get CSR from AWS PCA
aws acm-pca get-certificate-authority-csr \
    --certificate-authority-arn arn:aws:acm-pca:us-east-1:164314285563:certificate-authority/4cc5758d-ac26-41dd-b3c8-165cb2ffc80f \
    --output text > ca.csr

# Cross-sign the intermediate CA
openssl x509 -req -in ca.csr -CA rootCA1.crt -CAkey rootCA1.key -CAcreateserial \
    -out intCA12.crt -days 3650 -sha256 -extfile intCA12.conf -extensions v3_ca
```

#### Create and Sign Leaf Certificate
```bash
# Create leaf key and CSR
openssl genrsa -out leaf2.key 2048
openssl req -new -key leaf2.key -out leaf2.csr -config leaf2.conf

# Sign with AWS PCA
aws acm-pca issue-certificate \
    --certificate-authority-arn arn:aws:acm-pca:us-east-1:164314285563:certificate-authority/4cc5758d-ac26-41dd-b3c8-165cb2ffc80f \
    --csr fileb://leaf2.csr \
    --signing-algorithm SHA256WITHRSA \
    --validity Value=7,Type=DAYS \
    --template-arn arn:aws:acm-pca:::template/EndEntityCertificate/V1

# Get the signed certificate (replace with actual certificate ARN)
aws acm-pca get-certificate \
    --certificate-authority-arn arn:aws:acm-pca:us-east-1:164314285563:certificate-authority/4cc5758d-ac26-41dd-b3c8-165cb2ffc80f \
    --certificate-arn <CERTIFICATE_ARN> \
    --output text > leaf2.crt
```

### 3. Setup Nginx

#### Get AWS PCA Chain and Create Final Chain
```bash
# Get AWS PCA chain
aws acm-pca get-certificate-authority-certificate \
    --certificate-authority-arn arn:aws:acm-pca:us-east-1:164314285563:certificate-authority/4cc5758d-ac26-41dd-b3c8-165cb2ffc80f \
    --output text > pca_chain.txt

# Split chain
awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/{ print > NR".crt" }' pca_chain.txt
mv 1.crt intCA2.crt
mv 2.crt rootCA2.crt

# Create final chain
cat leaf2.crt intCA2.crt intCA12.crt > chain.pem
```

#### Create Nginx Configuration
```bash
cat > nginx.conf <<EOF
events {
    worker_connections  1024;
}
http {
    server {
        listen 443 ssl;
        server_name test.example.com;
        
        ssl_certificate /var/tmp/test-cross-sign/cross-sign-intermediate/chain.pem;
        ssl_certificate_key /var/tmp/test-cross-sign/cross-sign-intermediate/leaf2.key;
    }
}
EOF
```

### 4. Testing

#### Add Root CA to Chrome (macOS)
```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain rootCA1.crt
```

#### Test with curl
```bash
curl -IL --cacert rootCA1.crt https://test.example.com
```

#### Clean Up
```bash
sudo security remove-trusted-cert -d rootCA1.crt
sudo nginx -s stop
```
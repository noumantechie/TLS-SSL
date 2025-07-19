# TLS/SSL Certificate Setup for Nginx using Self-Signed Root CA

This guide demonstrates how to create a self-signed Root Certificate Authority (CA) and use it to sign TLS/SSL certificates for Nginx. Ideal for development environments and internal infrastructure.

## Understanding TLS/SSL Basicsse

### 🔐 What is TLS/SSL?

* Transport Layer Security (TLS) and its predecessor Secure Sockets Layer (SSL) are cryptographic protocols that provide secure communication over networks
* Creates an encrypted channel between client (browser) and server (Nginx)
* Uses asymmetric encryption with public/private key pairs
* Verified through digital certificates signed by trusted authorities

### 📜 Key Concepts

* **Root Certificate Authority (CA)**: Trusted entity that issues digital certificates
* **Certificate Signing Request (CSR)**: File containing server details for certificate generation
* **SAN (Subject Alternative Name)**: Extends certificate validity to multiple domains/IPs
* **Chain of Trust**: Root CA → Intermediate CA → Server Certificate

## Prerequisites

* CentOS 7/8 server with root access
* OpenSSL installed
* Nginx web server
* Windows machine for testing
* Firewall access to ports 80/443

## Self-Signed Certificate

A self-signed certificate is a certificate that is **signed by the same entity whose identity it certifies**.

> If I generate a **public key** and a **private key**, and then **sign the public key using my own private key**, I create a **self-signed certificate**. In other words, **I am issuing a certificate to myself**, saying:
>
> “I trust myself, and I vouch for my own identity.”

## Step-by-Step Setup

### 1. Create Root Certificate Authority

```bash
# Generate Root CA private key and certificate
# This creates a new RSA private key and uses it to generate a self-signed certificate valid for 10 years
openssl req -x509 -sha256 -days 3650 -nodes -newkey rsa:2048 \
  -subj "/C=US/ST=California/L=San Francisco/O=ExampleOrg/OU=IT Department/CN=RootCA.devopsmadeeasy.in" \
  -keyout rootCA.key -out rootCA.crt

# Install Root CA in system trust store
# Copy the certificate to the trust store and update the system CA list
sudo cp rootCA.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
```

### 2. Generate Server Certificate

```bash
# Create CSR configuration file
# This file defines the subject details and SANs for the server certificate
cat > csr.conf <<'EOF'
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
C = US
ST = California
L = San Francisco
O = ExampleOrg
OU = IT Department
CN = mynginx.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = mynginx.com
DNS.2 = test.mynginx.com
DNS.3 = www.mynginx.com
IP.1 = 68.183.142.158

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names
EOF

# Generate private key
# This is the private key for your Nginx server
openssl genrsa -out server.key 2048

# Create Certificate Signing Request (CSR)
# This generates a request file using the server key and config
openssl req -new -key server.key -out server.csr -config csr.conf

# Sign certificate with Root CA
# Use your Root CA to sign the CSR and create the server certificate
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out server.crt -days 3650 \
  -extensions v3_ext -extfile csr.conf
```

### 3. Configure Nginx

```bash
# Install Nginx
# Install the web server if it's not already present
sudo yum install -y nginx

# Create certificate directory
# Create a directory to securely store private keys
sudo mkdir -p /etc/pki/nginx/private

# Install certificates
# Copy the generated certificates into Nginx directories and set correct permissions
sudo cp server.crt /etc/pki/nginx/
sudo cp server.key /etc/pki/nginx/private/
sudo chmod 600 /etc/pki/nginx/private/server.key
sudo chown nginx:nginx /etc/pki/nginx/private/server.key

# Create Nginx SSL configuration
# Define HTTPS server block to use TLS/SSL certificates
sudo tee /etc/nginx/conf.d/ssl.conf > /dev/null <<'EOF'
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mynginx.com www.mynginx.com test.mynginx.com;

    ssl_certificate /etc/pki/nginx/server.crt;
    ssl_certificate_key /etc/pki/nginx/private/server.key;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

server {
    listen 80;
    server_name mynginx.com www.mynginx.com test.mynginx.com;
    return 301 https://$host$request_uri;
}
EOF

# Test configuration and restart Nginx
# Ensure syntax is correct and reload server
sudo nginx -t
sudo systemctl restart nginx

# Install and enable firewalld
# Make sure firewall service is running and enabled
yum install firewalld -y
systemctl enable --now firewalld
systemctl status firewalld

# Configure firewall
# Allow HTTP and HTTPS traffic through the firewall
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
```

### 4. Windows Client Configuration

**Edit hosts file (**\`\`**)**:

```
68.183.142.158 mynginx.com www.mynginx.com test.mynginx.com
```

**Import Root CA**:

* Copy `rootCA.crt` to Windows machine
* Open `certlm.msc` (Local Machine Certificate Manager)
* Import to "Trusted Root Certification Authorities"

**Flush DNS cache**:

```cmd
ipconfig /flushdns
```

## Verification

### Server-Side Checks

```bash
# Verify certificate chain
# Checks if server.crt is properly signed by the Root CA
openssl verify -CAfile rootCA.crt server.crt

# Check SAN configuration
# Display the certificate fields and filter Subject Alternative Names
openssl x509 -in server.crt -text -noout | grep "X509v3"

# Test local HTTPS connection using curl
curl -vk --resolve mynginx.com:443:127.0.0.1 https://mynginx.com

# Validate SSL handshake and certificate verification using openssl
openssl s_client -connect localhost:443 -servername mynginx.com | grep "Verify"
```

# Test local connection

curl -vk --resolve mynginx.com:443:127.0.0.1 [https://mynginx.com](https://mynginx.com)
openssl s\_client -connect localhost:443 -servername mynginx.com | grep "Verify"

````

### 5. Client-Side Verification

```bash
# Add entries to hosts file (on Windows)
# This ensures browser resolves domain to your Nginx server IP
68.183.142.158 mynginx.com www.mynginx.com test.mynginx.com
````

```bash
# Import Root CA Certificate on Windows
# This allows the browser to trust the self-signed CA
# - Copy rootCA.crt to Windows machine
# - Open certlm.msc (Local Machine Certificate Manager)
# - Import into "Trusted Root Certification Authorities"
```

```cmd
# Flush DNS cache (Windows)
ipconfig /flushdns
```

```text
# Open in browser: https://mynginx.com
# You should see:
# - A secure padlock icon
# - Certificate issued by: RootCA.devopsmadeeasy.in
# - All configured Subject Alternative Names (SANs)
```

## 6. Troubleshooting

| Symptom                            | Solution                                                  |
| ---------------------------------- | --------------------------------------------------------- |
| NET::ERR\_CERT\_AUTHORITY\_INVALID | Re-import Root CA on Windows                              |
| SSL\_ERROR\_BAD\_CERT\_DOMAIN      | Verify SAN in certificate matches hosts file              |
| Nginx fails to start               | Run `sudo nginx -t` to check config errors                |
| Connection refused                 | Check firewall: `sudo firewall-cmd --list-all`            |
| Certificate not trusted            | Ensure Root CA is in both system and browser trust stores |

## 7. Security Best Practices

* Use 4096-bit keys for production certificates
* Set shorter validity periods (90 days max)
* Revoke unused certificates immediately
* Use OCSP stapling for certificate revocation checks
* Separate root CA from issuing certificates
* Store root CA private key offline

> **Note**: Self-signed certificates are suitable for development and internal use only. Production environments should use certificates from trusted Certificate Authorities like Let's Encrypt or commercial providers.

```
```

TLS/SSL Certificate Setup for Nginx using Self-Signed Root CA
This guide demonstrates how to create a self-signed Root Certificate Authority (CA) and use it to sign TLS/SSL certificates for Nginx. Ideal for development environments and internal infrastructure.

Understanding TLS/SSL Basics
🔐 What is TLS/SSL?
Transport Layer Security (TLS) and its predecessor Secure Sockets Layer (SSL) are cryptographic protocols that provide secure communication over networks

Creates an encrypted channel between client (browser) and server (Nginx)

Uses asymmetric encryption with public/private key pairs

Verified through digital certificates signed by trusted authorities

📜 Key Concepts
Root Certificate Authority (CA): Trusted entity that issues digital certificates

Certificate Signing Request (CSR): File containing server details for certificate generation

SAN (Subject Alternative Name): Extends certificate validity to multiple domains/IPs

Chain of Trust: Root CA → Intermediate CA → Server Certificate

Prerequisites
CentOS 7/8 server with root access

OpenSSL installed

Nginx web server

Windows machine for testing

Firewall access to ports 80/443

Step-by-Step Setup
1. Create Root Certificate Authority
bash
# Generate Root CA private key and certificate
openssl req -x509 -sha256 -days 3650 -nodes -newkey rsa:2048 \
  -subj "/C=IN/ST=KA/L=BLR/O=DME/OU=DevSecOps/CN=RootCA.devopsmadeeasy.in" \
  -keyout rootCA.key -out rootCA.crt

# Install Root CA in system trust store
sudo cp rootCA.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
2. Generate Server Certificate
bash
# Create CSR configuration file
cat > csr.conf <<'EOF'
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
C = IN
ST = Karnataka
L = Bangalore
O = DME
OU = DevSecOps
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
openssl genrsa -out server.key 2048

# Create Certificate Signing Request (CSR)
openssl req -new -key server.key -out server.csr -config csr.conf

# Sign certificate with Root CA
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out server.crt -days 3650 \
  -extensions v3_ext -extfile csr.conf
3. Configure Nginx
bash
# Install Nginx
sudo yum install -y nginx

# Create certificate directory
sudo mkdir -p /etc/pki/nginx/private

# Install certificates
sudo cp server.crt /etc/pki/nginx/
sudo cp server.key /etc/pki/nginx/private/
sudo chmod 600 /etc/pki/nginx/private/server.key
sudo chown nginx:nginx /etc/pki/nginx/private/server.key

# Create Nginx SSL configuration
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
sudo nginx -t
sudo systemctl restart nginx

# Configure firewall
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
4. Windows Client Configuration
Edit hosts file (C:\Windows\System32\drivers\etc\hosts):

text
68.183.142.158 mynginx.com www.mynginx.com test.mynginx.com
Import Root CA:

Copy rootCA.crt to Windows machine

Open certlm.msc (Local Machine Certificate Manager)

Import to "Trusted Root Certification Authorities"

Flush DNS cache:

cmd
ipconfig /flushdns
Verification
Server-Side Checks
bash
# Verify certificate chain
openssl verify -CAfile rootCA.crt server.crt

# Check SAN configuration
openssl x509 -in server.crt -text -noout | grep "X509v3"

# Test local connection
curl -vk --resolve mynginx.com:443:127.0.0.1 https://mynginx.com
Client-Side Verification
Access in browser: https://mynginx.com

Check certificate details:

Should show "Issued by: RootCA.devopsmadeeasy.in"

Padlock icon should indicate secure connection

Certificate should show all configured SANs

Troubleshooting
Symptom	Solution
NET::ERR_CERT_AUTHORITY_INVALID	Re-import Root CA on Windows
SSL_ERROR_BAD_CERT_DOMAIN	Verify SAN in certificate matches hosts file
Nginx fails to start	Run sudo nginx -t to check config errors
Connection refused	Check firewall: sudo firewall-cmd --list-all
Certificate not trusted	Ensure Root CA is in both system and browser trust stores
Security Best Practices
Use 4096-bit keys for production certificates

Set shorter validity periods (90 days max)

Revoke unused certificates immediately

Use OCSP stapling for certificate revocation checks

Separate root CA from issuing certificates

Store root CA private key offline

Note: Self-signed certificates are suitable for development and internal use only. Production environments should use certificates from trusted Certificate Authorities like Let's Encrypt or commercial providers.

# TLS/SSL Certificate Setup for Nginx using Self-Signed Root CA

This guide demonstrates how to create a self-signed Root Certificate Authority (CA) and use it to sign TLS/SSL certificates for Nginx. Ideal for development environments and internal infrastructure.

## Understanding TLS/SSL Basics

### 🔐 What is TLS/SSL?

* TLS (Transport Layer Security) and SSL (Secure Sockets Layer) are cryptographic protocols used to secure communication over a network.
* They establish an encrypted link between a client (like a browser) and a server (like Nginx), ensuring that all data remains private.
* TLS/SSL uses asymmetric encryption, which involves a pair of keys: a public key and a private key.
* Digital certificates help verify the server's identity and are signed by a trusted authority.

### 📜 Self-Signed Certificate

A **self-signed certificate** is a digital certificate that is **signed by the same entity** whose identity it certifies.

If you generate a public key and a private key, and then **sign the public key with your own private key**, you are effectively saying:

> “I trust myself, and I vouch for my own identity.”

In this case, there is no third-party Certificate Authority involved. Self-signed certificates are commonly used in **development**, **testing**, and **internal environments**, but are not recommended for production use.

### 🧠 Key Concepts

* **Root Certificate Authority (CA)**: A trusted entity that issues and signs digital certificates.
* **Certificate Signing Request (CSR)**: A file generated on the server that contains the server details and is sent to a CA for signing.
* **SAN (Subject Alternative Name)**: Allows a certificate to secure multiple domains or IP addresses.
* **Chain of Trust**: A hierarchy where the Root CA signs an Intermediate CA, which then signs the server certificate.

## Prerequisites

* CentOS 7/8 server with root access
* OpenSSL installed
* Nginx web server
* Windows machine for testing
* Firewall access to ports 80/443

## Step-by-Step Setup

### 1. Create Root Certificate Authority

```bash
openssl req -x509 -sha256 -days 3650 -nodes -newkey rsa:2048 \
-subj "/C=US/ST=California/L=San Francisco/O=ExampleOrg/OU=IT Department/CN=RootCA.devopsmadeeasy.in" \
-keyout rootCA.key -out rootCA.crt
```

#### 📘 Explanation:

* `-x509`: Creates a self-signed certificate instead of generating a certificate signing request (CSR)
* `-sha256`: Uses the SHA-256 hashing algorithm (more secure than older ones like SHA1)
* `-days 3650`: Certificate is valid for 10 years (3650 days)
* `-nodes`: Tells OpenSSL **not** to encrypt the private key (no passphrase)
* `-newkey rsa:2048`: Generates a new 2048-bit RSA key pair
* `-subj`: Provides the subject details (like country, state, org, common name etc.)
* `-keyout`: File name to save the private key (output file)
* `-out`: File name to save the self-signed certificate

```bash
# Copy the Root CA certificate to the trusted anchors directory
sudo cp rootCA.crt /etc/pki/ca-trust/source/anchors/

# Update the system's CA trust database so it includes the new Root CA
sudo update-ca-trust extract
```

### 2. Generate Server Certificate

```bash
# Create CSR config file (includes SANs and extensions)
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

# Generate server private key
openssl genrsa -out server.key 2048

# Generate CSR using server key and config file
openssl req -new -key server.key -out server.csr -config csr.conf

# Sign CSR with Root CA to generate server certificate
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out server.crt -days 3650 \
  -extensions v3_ext -extfile csr.conf
```

### 3. Configure Nginx

```bash
# Install Nginx
sudo yum install -y nginx

# Create secure directory for private key
sudo mkdir -p /etc/pki/nginx/private

# Move server key and cert to proper Nginx locations
sudo cp server.crt /etc/pki/nginx/
sudo cp server.key /etc/pki/nginx/private/

# Set proper permissions for private key
sudo chmod 600 /etc/pki/nginx/private/server.key
sudo chown nginx:nginx /etc/pki/nginx/private/server.key

# Create SSL config for Nginx
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

# Check Nginx config
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx

# Install and start firewall
yum install firewalld -y
systemctl enable --now firewalld
systemctl status firewalld

# Open HTTP and HTTPS ports
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
```

### 4. Windows Client Configuration

**Edit hosts file (`C:\Windows\System32\drivers\etc\hosts`)**:

```
68.183.142.158 mynginx.com www.mynginx.com test.mynginx.com
```

**Import Root CA**:

* Copy `rootCA.crt` to your Windows machine
* Open `certlm.msc` (Local Machine Certificate Manager)
* Import the certificate into "Trusted Root Certification Authorities"

**Flush DNS cache**:

```cmd
ipconfig /flushdns
```

## Verification

### Server-Side Checks

```bash
# Verify server certificate with Root CA
openssl verify -CAfile rootCA.crt server.crt

# View certificate extensions like SAN
openssl x509 -in server.crt -text -noout | grep "X509v3"

# Test HTTPS request locally
curl -vk --resolve mynginx.com:443:127.0.0.1 https://mynginx.com

# Inspect TLS handshake
openssl s_client -connect localhost:443 -servername mynginx.com | grep "Verify"
```

### Client-Side Verification

* Access in browser: [https://mynginx.com](https://mynginx.com)
* Check certificate details:

  * Should show "Issued by: RootCA.devopsmadeeasy.in"
  * Secure padlock icon should appear
  * All SANs should be listed under certificate details

## Troubleshooting

| Symptom                            | Solution                                                  |
| ---------------------------------- | --------------------------------------------------------- |
| NET::ERR\_CERT\_AUTHORITY\_INVALID | Re-import Root CA on Windows                              |
| SSL\_ERROR\_BAD\_CERT\_DOMAIN      | Verify SAN in certificate matches hosts file              |
| Nginx fails to start               | Run `sudo nginx -t` to check config errors                |
| Connection refused                 | Check firewall: `sudo firewall-cmd --list-all`            |
| Certificate not trusted            | Ensure Root CA is in both system and browser trust stores |

## Security Best Practices

* Use 4096-bit keys for production certificates
* Set shorter validity periods (90 days max)
* Revoke unused certificates immediately
* Use OCSP stapling for certificate revocation checks
* Separate root CA from issuing certificates
* Store root CA private key offline

> **Note**: Self-signed certificates are suitable for development and internal use only. Production environments should use certificates from trusted Certificate Authorities like Let's Encrypt or commercial providers.

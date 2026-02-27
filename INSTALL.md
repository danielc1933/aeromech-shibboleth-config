# Installation Guide – Shibboleth SP on RHEL 9.7 with Apache and Okta

## Target environment

| Component | Version |
|-----------|---------|
| Operating System | Red Hat Enterprise Linux 9.7 (Plow) |
| Web server | Apache HTTP Server 2.4 |
| Shibboleth SP | 3.x (from Shibboleth project repository) |
| Identity Provider | Okta SAML 2.0 |

---

## 1. Prerequisites

- A root or `sudo`-capable account on the RHEL 9.7 host.
- The host must be registered with Red Hat Subscription Management (`subscription-manager`).
- DNS for `web.aeromech.usyd.edu.au` must resolve to this host.
- A valid TLS certificate and private key for `web.aeromech.usyd.edu.au`.
- Outbound HTTPS access to `https://sso.sydney.edu.au` (Okta SSO endpoint).

---

## 2. Enable required repositories

```bash
# Ensure the base OS repos are enabled
sudo subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms

# Add the Shibboleth project repository for RHEL 9
sudo curl -o /etc/yum.repos.d/shibboleth.repo \
  https://shibboleth.net/cgi-bin/mirrorlist.cgi/RHEL_9
```

---

## 3. Install packages

```bash
sudo dnf install -y \
  httpd \
  mod_ssl \
  shibboleth \
  openssl
```

> **Note:** `mod_shib` (the Apache module) is included in the `shibboleth` package on RHEL 9.

---

## 4. Install TLS certificates

```bash
sudo cp aeromech.crt       /etc/pki/tls/certs/aeromech.crt
sudo cp aeromech.key       /etc/pki/tls/private/aeromech.key
sudo chmod 600 /etc/pki/tls/private/aeromech.key
```

If you have an intermediate/chain certificate:

```bash
sudo cp aeromech-chain.crt /etc/pki/tls/certs/aeromech-chain.crt
```

---

## 5. Generate Shibboleth SP key pair

```bash
sudo /etc/shibboleth/keygen.sh \
  -u shibd -g shibd \
  -h web.aeromech.usyd.edu.au \
  -e https://web.aeromech.usyd.edu.au/shibboleth \
  -o /etc/shibboleth
```

This creates:
- `/etc/shibboleth/sp-key.pem`   – private key (readable only by shibd)
- `/etc/shibboleth/sp-cert.pem`  – self-signed certificate (public)

For separate signing and encryption keys (recommended):

```bash
sudo /etc/shibboleth/keygen.sh -u shibd -g shibd \
  -h web.aeromech.usyd.edu.au \
  -e https://web.aeromech.usyd.edu.au/shibboleth \
  -n sp-signing -o /etc/shibboleth

sudo /etc/shibboleth/keygen.sh -u shibd -g shibd \
  -h web.aeromech.usyd.edu.au \
  -e https://web.aeromech.usyd.edu.au/shibboleth \
  -n sp-encrypt -o /etc/shibboleth
```

---

## 6. Deploy Shibboleth configuration files

```bash
# Back up the stock configuration
sudo cp /etc/shibboleth/shibboleth2.xml  /etc/shibboleth/shibboleth2.xml.orig
sudo cp /etc/shibboleth/attribute-map.xml /etc/shibboleth/attribute-map.xml.orig

# Install repository files
sudo cp shibboleth2.xml      /etc/shibboleth/shibboleth2.xml
sudo cp attribute-map.xml    /etc/shibboleth/attribute-map.xml
sudo cp okta-idp-metadata.xml /etc/shibboleth/okta-idp-metadata.xml

sudo chown root:shibd /etc/shibboleth/shibboleth2.xml \
                      /etc/shibboleth/attribute-map.xml \
                      /etc/shibboleth/okta-idp-metadata.xml
sudo chmod 640 /etc/shibboleth/shibboleth2.xml \
               /etc/shibboleth/attribute-map.xml \
               /etc/shibboleth/okta-idp-metadata.xml
```

### 6.1 Update the Okta IdP metadata certificate

Open `/etc/shibboleth/okta-idp-metadata.xml` and replace
`REPLACE_WITH_ACTUAL_OKTA_SIGNING_CERTIFICATE_BASE64` with the base-64
encoded X.509 certificate from your Okta application:

1. Log in to the **Okta Admin Console**.
2. Go to **Applications → Applications** and open the Aeromech app.
3. Click the **Sign On** tab.
4. Under *SAML Signing Certificates*, download the active certificate.
5. Open the `.cer` / `.pem` file and copy the base-64 content between the
   `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` markers.
6. Paste it as the value of `<ds:X509Certificate>` in `okta-idp-metadata.xml`.

---

## 7. Deploy Apache configuration

```bash
sudo cp apache-shibboleth.conf /etc/httpd/conf.d/aeromech-shibboleth.conf

# Verify the configuration parses correctly
sudo apachectl configtest
```

Expected output: `Syntax OK`

---

## 8. Configure SELinux (RHEL default: enforcing)

```bash
# Allow Apache to connect to the Shibboleth daemon socket
sudo setsebool -P httpd_can_network_connect 1

# Restore default file contexts if needed
sudo restorecon -Rv /etc/shibboleth /var/log/shibboleth
```

---

## 9. Configure firewalld

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## 10. Enable and start services

```bash
sudo systemctl enable --now shibd
sudo systemctl enable --now httpd
```

Verify both are running:

```bash
sudo systemctl status shibd
sudo systemctl status httpd
```

---

## 11. Register SP metadata with Okta

1. Navigate to `https://web.aeromech.usyd.edu.au/Shibboleth.sso/Metadata` in
   a browser (or use `curl`) to download the SP metadata XML.
2. In the **Okta Admin Console**, open the Aeromech application.
3. Go to **Sign On → Advanced Sign-on Settings** (or *Edit*).
4. Upload the SP metadata or enter the values manually:
   - **SP Entity ID:** `https://web.aeromech.usyd.edu.au/shibboleth`
   - **ACS URL:** `https://web.aeromech.usyd.edu.au/Shibboleth.sso/SAML2/POST`
5. Save the changes and re-assign users/groups as needed.

---

## 12. Verify log files

| Service | Log path |
|---------|----------|
| shibd daemon | `/var/log/shibboleth/shibd.log` |
| Apache error log | `/var/log/httpd/aeromech-error.log` |
| Apache access log | `/var/log/httpd/aeromech-access.log` |
| Shibboleth transaction log | `/var/log/shibboleth/transaction.log` |

```bash
sudo tail -f /var/log/shibboleth/shibd.log
sudo tail -f /var/log/httpd/aeromech-error.log
```

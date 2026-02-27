# Troubleshooting Guide

This document covers the most common issues encountered when deploying the
Shibboleth SP / Okta SAML integration on RHEL 9.7.

---

## Diagnostic quick-start

```bash
# 1. Check service status
sudo systemctl status shibd httpd

# 2. Check shibd log for errors
sudo tail -100 /var/log/shibboleth/shibd.log | grep -i 'error\|warn\|crit'

# 3. Check Apache error log
sudo tail -100 /var/log/httpd/aeromech-error.log

# 4. Check the Shibboleth status endpoint
curl -sk https://web.aeromech.usyd.edu.au/Shibboleth.sso/Status

# 5. View active Shibboleth session (while authenticated)
curl -sk https://web.aeromech.usyd.edu.au/Shibboleth.sso/Session
```

---

## Common issues

### 1. shibd fails to start

**Symptom:** `systemctl status shibd` shows `failed` or `activating (auto-restart)`.

**Checks and fixes:**

```bash
sudo journalctl -u shibd -n 50 --no-pager
sudo /usr/sbin/shibd -t   # config test mode
```

| Cause | Fix |
|-------|-----|
| Invalid XML in `shibboleth2.xml` | Validate with `xmllint --noout /etc/shibboleth/shibboleth2.xml` |
| Missing SP key/cert files | Re-run `keygen.sh` (see INSTALL.md §5) |
| Wrong file permissions on keys | `chmod 640 /etc/shibboleth/sp-*.pem; chown root:shibd /etc/shibboleth/sp-*.pem` |
| Placeholder certificate in `okta-idp-metadata.xml` | Replace `REPLACE_WITH_ACTUAL_OKTA_SIGNING_CERTIFICATE_BASE64` with real cert |

---

### 2. Accessing `/private` returns HTTP 500

**Symptom:** Internal Server Error when visiting `https://web.aeromech.usyd.edu.au/private`.

**Checks and fixes:**

```bash
sudo tail -20 /var/log/httpd/aeromech-error.log
```

| Cause | Fix |
|-------|-----|
| `mod_shib` not loaded | `httpd -M | grep shib` – if missing, check `/etc/httpd/conf.d/shib.conf` |
| shibd not running | `systemctl start shibd` |
| Shibboleth socket missing | `ls -l /var/run/shibboleth/shibd.sock` – restart shibd if absent |
| SELinux denying socket access | See §SELinux below |

---

### 3. Browser is not redirected to Okta

**Symptom:** Accessing `/private` returns HTTP 403 or the raw content without SSO.

**Checks:**

- Confirm `AuthType shibboleth` and `Require shib-session` are present in
  `apache-shibboleth.conf`.
- Confirm the `RequestMapper` in `shibboleth2.xml` has `requireSession="true"`
  for the `/private` path.
- Restart both services after any config change:

  ```bash
  sudo systemctl restart shibd httpd
  ```

---

### 4. SAML authentication fails – "No metadata found for …"

**Symptom:** Error in shibd log: `No metadata found for entity …` or `opensaml::saml2md::MetadataException`.

**Fix:**

- Verify `okta-idp-metadata.xml` is present at the path configured in
  `shibboleth2.xml` (`/etc/shibboleth/okta-idp-metadata.xml`).
- Verify the `entityID` in the metadata file matches exactly
  `http://www.okta.com/exk13lya02eet9NpM3l7` (note `http://`, not `https://`).
- Check file permissions:

  ```bash
  ls -l /etc/shibboleth/okta-idp-metadata.xml
  # Should be: -rw-r----- root shibd
  ```

---

### 5. Signature validation failure

**Symptom:** shibd log shows `XMLSecurityException` or `signature validation failed`.

**Causes and fixes:**

| Cause | Fix |
|-------|-----|
| Stale / wrong Okta signing certificate in metadata | Download current cert from Okta Admin Console → Sign On tab and update `okta-idp-metadata.xml` |
| Clock skew > 3 minutes between SP and Okta | Ensure `chronyd` / `ntpd` is running: `timedatectl status`; increase `clockSkew` in `shibboleth2.xml` temporarily |

---

### 6. Attributes not reaching the application

**Symptom:** `/Shibboleth.sso/Session` shows attributes, but the application sees
empty `REMOTE_USER` or missing headers.

**Checks:**

- Confirm `REMOTE_USER` order in `shibboleth2.xml`:

  ```xml
  REMOTE_USER="eppn persistent-id targeted-id"
  ```

- If the application reads HTTP headers instead of environment variables, add
  `ShibUseHeaders on` inside the `<Location /private>` block in
  `apache-shibboleth.conf` (and ensure the application is not run via `mod_proxy`
  without header forwarding).

- Verify `attribute-map.xml` contains entries for all expected attributes.

---

### 7. SELinux denials

**Symptom:** `shibd` starts but Apache cannot connect to the Shibboleth socket.
Audit log shows AVC denials.

```bash
sudo ausearch -m avc -ts recent | grep shib
sudo sealert -a /var/log/audit/audit.log | grep shib
```

**Fix:**

```bash
sudo setsebool -P httpd_can_network_connect 1

# If the socket path has a non-default context, restore it:
sudo restorecon -Rv /var/run/shibboleth
```

---

### 8. TLS certificate errors

**Symptom:** Browser shows certificate warning or `curl` reports
`SSL certificate problem`.

**Fix:**

```bash
# Verify certificate and key match
openssl x509 -noout -modulus -in /etc/pki/tls/certs/aeromech.crt | md5sum
openssl rsa  -noout -modulus -in /etc/pki/tls/private/aeromech.key | md5sum
# Both hashes must be identical.

# Check expiry
openssl x509 -noout -dates -in /etc/pki/tls/certs/aeromech.crt
```

---

### 9. Single Logout (SLO) not working

**Symptom:** After visiting `/Shibboleth.sso/Logout`, the user is still logged in
at Okta.

**Notes:**

- Okta SLO support depends on the application configuration.  Verify that an
  SLO endpoint is configured in the Okta Admin Console.
- Confirm the `<SingleLogoutService>` element in `okta-idp-metadata.xml` is
  correct.
- SLO via SAML requires the SP to sign logout requests; ensure the SP signing
  key is configured correctly.

---

### 10. "Unable to locate issuer" or wrong entityID

**Symptom:** `opensaml::saml2md::MetadataException: Unable to locate issuer in
  supplied metadata`.

**Fix:**

- The `entityID` in `shibboleth2.xml`
  (`https://web.aeromech.usyd.edu.au/shibboleth`) must match exactly what is
  registered in Okta as the *Audience URI* / *SP Entity ID*.
- Check for trailing slashes, `http://` vs `https://`, and case differences.

---

## Useful commands reference

```bash
# Test shibd configuration
sudo /usr/sbin/shibd -t

# Validate XML files
xmllint --noout /etc/shibboleth/shibboleth2.xml && echo "OK"
xmllint --noout /etc/shibboleth/attribute-map.xml && echo "OK"
xmllint --noout /etc/shibboleth/okta-idp-metadata.xml && echo "OK"

# Reload Shibboleth configuration without full restart
sudo kill -HUP $(cat /var/run/shibboleth/shibd.pid)

# Check loaded Apache modules
httpd -M | grep -E 'shib|ssl|headers'

# Decode a SAML response (base64 encoded) for debugging
echo "<paste_base64_here>" | base64 -d | xmllint --format - 2>/dev/null | less
```

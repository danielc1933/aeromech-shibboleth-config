# Deployment and Testing Guide

## Overview

This document describes how to deploy the Shibboleth SP / Okta integration
and verify that `https://web.aeromech.usyd.edu.au/private` is correctly
protected with SAML authentication.

---

## 1. Pre-deployment checklist

| # | Check | Command / Action |
|---|-------|-----------------|
| 1 | All packages installed | `rpm -q shibboleth httpd mod_ssl` |
| 2 | TLS certificate present | `ls -l /etc/pki/tls/certs/aeromech.crt` |
| 3 | SP key pair generated | `ls -l /etc/shibboleth/sp-*.pem` |
| 4 | Okta metadata certificate replaced | `grep -v REPLACE /etc/shibboleth/okta-idp-metadata.xml` |
| 5 | Apache config passes syntax check | `apachectl configtest` |
| 6 | `shibd` is running | `systemctl is-active shibd` |
| 7 | `httpd` is running | `systemctl is-active httpd` |
| 8 | Firewall allows 443 | `firewall-cmd --list-services` |

---

## 2. Service startup

```bash
sudo systemctl restart shibd
sudo systemctl restart httpd

# Confirm both are active
sudo systemctl status shibd httpd
```

Watch the shibd log during startup for any configuration errors:

```bash
sudo journalctl -u shibd -f
```

---

## 3. Metadata exchange

### 3.1 Download SP metadata

```bash
curl -sk https://web.aeromech.usyd.edu.au/Shibboleth.sso/Metadata \
  -o /tmp/sp-metadata.xml

# Verify it is valid XML with the correct entityID
xmllint --noout /tmp/sp-metadata.xml && echo "XML OK"
grep 'entityID' /tmp/sp-metadata.xml
```

Expected entityID: `https://web.aeromech.usyd.edu.au/shibboleth`

### 3.2 Upload SP metadata to Okta

1. Log in to the **Okta Admin Console**.
2. Open the *Aeromech* SAML application.
3. On the **Sign On** tab, upload `/tmp/sp-metadata.xml` or copy the ACS URL
   and entity ID from the metadata XML.
4. Save changes.

---

## 4. Authentication flow test

### 4.1 Test the Shibboleth status handler (unauthenticated)

```bash
curl -sk https://web.aeromech.usyd.edu.au/Shibboleth.sso/Status | head -20
```

Expected: XML output containing `<OK/>` or similar status.

### 4.2 Test redirect to Okta

Open a **private/incognito** browser window and navigate to:

```
https://web.aeromech.usyd.edu.au/private
```

Expected behaviour:
1. Browser redirects to `https://sso.sydney.edu.au/app/sydneyuni_webaeromech_1/…/sso/saml`
2. Okta login page is displayed.
3. After successful authentication, browser is redirected back to `/private`.
4. Content of `/private` is shown without further prompts.

### 4.3 Verify SAML attributes in the session

While authenticated, open:

```
https://web.aeromech.usyd.edu.au/Shibboleth.sso/Session
```

You should see a list of attributes released by Okta, e.g.:

```
Attributes:
  eppn: jsmith@sydney.edu.au
  mail: j.smith@sydney.edu.au
  displayName: John Smith
```

### 4.4 Test Single Logout

Navigate to:

```
https://web.aeromech.usyd.edu.au/Shibboleth.sso/Logout
```

Expected: User is logged out locally and (if Okta supports SLO) at the IdP.
Subsequent access to `/private` should again prompt for authentication.

---

## 5. Functional test with curl

Use `curl` to simulate the SP-initiated flow (expects a 302 redirect to Okta):

```bash
curl -sv --max-redirs 0 \
  https://web.aeromech.usyd.edu.au/private 2>&1 | grep -E "^[<>] |^< HTTP"
```

Expected:

```
< HTTP/1.1 302 Found
< Location: https://sso.sydney.edu.au/app/...
```

---

## 6. Attribute-release verification

Create a minimal PHP or CGI test page at `/var/www/html/private/test.php`:

```php
<?php
// Display Shibboleth attributes passed as environment variables
$attrs = [
    'REMOTE_USER', 'HTTP_EPPN', 'HTTP_MAIL', 'HTTP_DISPLAYNAME',
    'HTTP_GIVENNAME', 'HTTP_SN', 'HTTP_AFFILIATION'
];
foreach ($attrs as $a) {
    echo "$a = " . (getenv($a) ?: $_SERVER[$a] ?? '(not set)') . "\n";
}
?>
```

Access `https://web.aeromech.usyd.edu.au/private/test.php` after
authenticating and confirm all expected attributes are populated.

---

## 7. Load / smoke test

> **Note:** `ab` (Apache Bench) requires HTTPS support compiled in (`ab -V` shows
> `OpenSSL`). If HTTPS is unavailable, use `siege` or `wrk` instead.

```bash
# Requires Apache Bench with SSL support (httpd-tools)
# Test the public root (unauthenticated)
ab -n 100 -c 10 https://web.aeromech.usyd.edu.au/

# Test the protected path (will 302 unless a session cookie is provided)
ab -n 100 -c 10 https://web.aeromech.usyd.edu.au/private

# Alternative with siege (handles HTTPS reliably):
# siege -c 10 -r 10 https://web.aeromech.usyd.edu.au/
```

---

## 8. Post-deployment monitoring

```bash
# Watch live logs
sudo tail -f /var/log/shibboleth/shibd.log \
             /var/log/shibboleth/transaction.log \
             /var/log/httpd/aeromech-error.log

# Check for SAML errors in shibd log
sudo grep -i 'error\|warn\|crit' /var/log/shibboleth/shibd.log | tail -50
```

---

## 9. Rollback procedure

If the new configuration causes issues:

```bash
# Restore original Shibboleth config
sudo cp /etc/shibboleth/shibboleth2.xml.orig  /etc/shibboleth/shibboleth2.xml
sudo cp /etc/shibboleth/attribute-map.xml.orig /etc/shibboleth/attribute-map.xml

# Remove the Apache virtual-host config
sudo rm /etc/httpd/conf.d/aeromech-shibboleth.conf

sudo systemctl restart shibd httpd
```

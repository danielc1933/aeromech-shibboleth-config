# Shibboleth SP Configuration for Okta Integration

This repository contains the Shibboleth Service Provider (SP) configuration for integrating with Okta as the Identity Provider (IdP) on RHEL 9.7 with Apache.

## Overview

This setup protects `https://web.aeromech.usyd.edu.au/private` with SAML authentication through Okta.

## Files Included

- `shibboleth2.xml` - Main Shibboleth SP configuration
- `attribute-map.xml` - Attribute mappings from Okta
- `apache-shibboleth.conf` - Apache configuration for /private protection
- `INSTALL.md` - Installation instructions for RHEL 9.7
- `DEPLOYMENT.md` - Deployment and testing guide
- `okta-idp-metadata.xml` - Okta IdP metadata
- `TROUBLESHOOTING.md` - Common issues and solutions

## Quick Start

1. Review the INSTALL.md for prerequisites and installation steps
2. Follow DEPLOYMENT.md for testing the integration
3. Consult TROUBLESHOOTING.md if you encounter any issues

## Environment

- OS: Red Hat Enterprise Linux 9.7 (Plow)
- Web Server: Apache
- Authentication: Okta SAML 2.0
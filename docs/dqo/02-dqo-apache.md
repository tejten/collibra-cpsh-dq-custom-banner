# Create a custom header and footer in Collibra Data Quality & Observability

There are two workarounds - one with Nginx and the other with Apache HTTP 2.4. Both options are documented in this repository. Apache HTTP 2.4 consumes ~80 MB RAM vs. ~20 MB for Nginx.

## Reverse-proxy injection via Apache HTTP 2.4:

## Install Apache and required modules

```bash
sudo dnf install -y httpd mod_ssl   # mod_ssl only if you want HTTPS
sudo systemctl enable --now httpd
```
## Open the firewall (HTTP + HTTPS):

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https   # omit if HTTP-only
sudo firewall-cmd --reload
```
## Create the reverse-proxy & banner config:

```bash
vi /etc/httpd/conf.d/dqo.conf

##############################################################################
# Collibra Data Quality & Observability proxy with mandatory CONFIDENTIAL
# banners.  Place in /etc/httpd/conf.d/dqo.conf and reload Apache (systemd).
##############################################################################

# Use your FQDN or leave as _ to match any host header
ServerName DQ01.TEJ.TENMATTAM.COM

# ---------------------------------------------------------------------------
# 1)  HTTP listener (port 80)
# ---------------------------------------------------------------------------
<VirtualHost *:80>
    # ---- Reverse-proxy to the DQO backend ---------------------------------
    ProxyPreserveHost On
    ProxyPass        /  http://127.0.0.1:9000/ retry=0
    ProxyPassReverse /  http://127.0.0.1:9000/

    # Make sure upstream sends plain HTML (no gzip) so mod_substitute can work
    RequestHeader unset Accept-Encoding

    # Collibra sets a strict CSP; remove it so our inline CSS is allowed
    Header unset Content-Security-Policy

    # ---- Banner injection using mod_substitute ----------------------------
    AddOutputFilterByType SUBSTITUTE text/html
    Substitute "s|<body>|<body><div class='gov-banner-top'>CONFIDENTIAL</div><div class='gov-banner-bot'>CONFIDENTIAL</div><style>body{padding-top:26px;padding-bottom:26px}.gov-banner-top,.gov-banner-bot{position:fixed;left:0;right:0;height:26px;line-height:26px;background:#0003bf;color:#fff;font:bold 14px sans-serif;text-align:center;pointer-events:none;z-index:2147483647}.gov-banner-top{top:0}.gov-banner-bot{bottom:0}</style>|ni"


    ErrorLog  /var/log/httpd/dqo_error.log
    CustomLog /var/log/httpd/dqo_access.log combined
</VirtualHost>

# ---------------------------------------------------------------------------
# 2)  OPTIONAL HTTPS listener (port 443)
#     Comment or delete this entire block if you only want HTTP.
# ---------------------------------------------------------------------------
<VirtualHost *:443>
    ServerName DQ01.EXAMPLE.COM

    ProxyPreserveHost On
    ProxyPass        /  http://127.0.0.1:9000/ retry=0
    ProxyPassReverse /  http://127.0.0.1:9000/
    RequestHeader unset Accept-Encoding
    Header unset Content-Security-Policy

    AddOutputFilterByType SUBSTITUTE text/html
    Substitute "s|<body>|<body><div class='gov-banner-top'>CONFIDENTIAL</div><div class='gov-banner-bot'>CONFIDENTIAL</div><style>body{padding-top:26px;padding-bottom:26px}.gov-banner-top,.gov-banner-bot{position:fixed;left:0;right:0;height:26px;line-height:26px;background:#0003bf;color:#fff;font:bold 14px sans-serif;text-align:center;pointer-events:none;z-index:2147483647}.gov-banner-top{top:0}.gov-banner-bot{bottom:0}</style>|ni"

    # --- TLS configuration -------------------------------------------------
    SSLEngine on
    SSLCertificateFile  /etc/pki/tls/certs/DQ01.EXAMPLE.COM.crt   # <-- edit
    SSLCertificateKeyFile /etc/pki/tls/private/DQ01.EXAMPLE.COM.key

    # Hardened minimum TLS settings (optional)
    SSLProtocol       all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite    HIGH:!aNULL:!MD5:!3DES
    SSLHonorCipherOrder on

    ErrorLog  /var/log/httpd/dqo_ssl_error.log
    CustomLog /var/log/httpd/dqo_ssl_access.log combined
</VirtualHost>
```
**Note:** If you only want HTTP, remove the entire `<VirtualHost *:443>` block and skip the SSL steps.

## Validate and reload Apache:

```bash
apachectl configtest
systemctl reload httpd
```

## Verify:

```bash
# Banner markup present?
curl -s http://dq01.tej.tenmattam.com/ | grep CONFIDENTIAL

# Browser test (private window):
http://dq01.example.com/          -> Collibra with blue top & bottom bars
https://dq01.example.com/         -> same (if HTTPS block enabled)
```

Open a browser tab – a blue strip now appears at the very top and bottom of every screen (login + in-app).

![DQO](./images/dqo-banner.png)

## Traffic flow & banner visibility:

The “CONFIDENTIAL” banner is injected by Apache HTTP Server reverse-proxy, not by the Collibra application itself. Any user who reaches the service through the canonical DNS name – `http(s)://dq01.example.com (ports 80/443)` – is automatically routed through that proxy and therefore always sees the banner. However, if someone bypasses DNS and goes straight to the DQO process on its internal port (`http://dq01.example.com:9000` or the node’s raw IP + 9000), their request never touches reverse proxy and the banner will not be present. In production we prevent such “direct-port” access with a firewall rule and ask users to bookmark only the proxy URL; this guarantees that every page is served through reverse proxy and remains compliant with the banner requirement.

## Upgrade notes:

- All changes live only in /etc/httpd/conf.d/dqo.conf. Upgrading or re-deploying DQO does not overwrite the banner.
- After patching DQO, simply test the URL once - no additional steps.
- To change banner text or color later, edit the `<style> / <div>` in the same file and reload Nginx.


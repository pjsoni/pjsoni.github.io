---
layout: posts
title: "Automating TLS Certificates with Certbot, Cloudflare DNS, and Traefik"
date: 2026-09-19
slug: automated-tls-certificates-certbot-cloudflare-traefik
summary: "A practical approach to automatically issuing, distributing, and renewing TLS certificates with Certbot, Cloudflare DNS, Syncthing, and Traefik."
categories: [Infrastructure, Security]
tags: [homelab,docker,certbot,traefik,tls]
jumbotron:
  meta: true
---

Managing TLS certificates becomes increasingly complicated once an environment has multiple services and hosts. I wanted certificate issuance, distribution, and consumption to be separate concerns without requiring Traefik to restart whenever a certificate changes.

The resulting setup is fairly simple:

* **Certbot** issues and renews certificates using Cloudflare DNS-01.
* **Syncthing** distributes the resulting certificate files to the hosts that need them.
* **Traefik** reads the certificates directly from the filesystem.
* A small **cert-watcher** container detects certificate changes and triggers a Traefik file-provider reload.
* **Infisical** is used for application secrets, but not for the bootstrap credential required to obtain the certificates.

The important part is that renewal does not require rebuilding images, copying certificates manually, or restarting Traefik.

## The certificate flow

The certificate lifecycle looks like this:

```text
                    Cloudflare DNS
                         ▲
                         │ DNS-01
                         │
                    ┌────┴─────┐
                    │  Certbot │
                    └────┬─────┘
                         │
                 /certs/domain/*
                         │
                    ┌────▼─────┐
                    │ Syncthing│
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │                     │
        Traefik host 1        Traefik host 2
              │                     │
              └──────────┬──────────┘
                         │
                   cert-watcher
                         │
                  Traefik reload
```

Only the certificate consumers receive the private keys.

## 1. Certbot container

I use a small custom container around Certbot rather than configuring every certificate manually through Compose.

The container persists the important Certbot directories:

```yaml
services:
  certbot:
    image: ghcr.io/pjsoni/certbot-cloudflare:${CERTBOT_VERSION:-latest}
    container_name: certbot
    restart: unless-stopped

    env_file:
      - stack.env

    volumes:
      - /docker-data/certbot-data/config:/etc/letsencrypt
      - /docker-data/certbot-data/logs:/var/log/letsencrypt
      - /docker-data/certbot-data/cloudflare.ini:/cloudflare.ini:ro
      - /docker-data/certbot-data/renewal-hooks:/etc/letsencrypt/renewal-hooks
      - /docker-data/certbot-data/var-lib-letsencrypt:/var/lib/letsencrypt

    networks:
      - service-net

networks:
  service-net:
    external: true
```

The persistent `/etc/letsencrypt` directory is particularly important. It contains Certbot's certificate, renewal, and account state. Without it, the container would effectively start from scratch every time it was recreated.

### Certificate configuration

The certificates are configured through an environment variable rather than hard-coded into the script.

For example:

```env
CERTBOT_EMAIL=admin@example.com
CERTBOT_CERTS=example.com,*.example.com|internal.example.com,*.internal.example.com
CERTBOT_CF_CREDENTIALS=/cloudflare.ini
CERTBOT_PROPAGATION_SECONDS=60
CERTBOT_RENEW_INTERVAL=12h
CERTBOT_CREATE_COMBINED_CERT=true
CERTBOT_COMBINED_CERT_OUTPUT=/cert-export
```

The format is:

```text
certificate-primary-name,san-name,san-name|another-certificate,san-name
```

The first hostname becomes the Certbot certificate name.

This makes adding another certificate as simple as changing configuration rather than modifying the container image.

## 2. Cloudflare DNS-01

Certbot uses the Cloudflare DNS plugin to complete the ACME DNS-01 challenge.

The Cloudflare credential is mounted as a file:

```yaml
- /docker-data/certbot-data/cloudflare.ini:/cloudflare.ini:ro
```

and referenced with:

```env
CERTBOT_CF_CREDENTIALS=/cloudflare.ini
```

The file itself is protected on the host:

```bash
chmod 600 /docker-data/certbot-data/cloudflare.ini
```

This is intentional.

I originally considered putting the Cloudflare credential into Infisical, but that creates a bootstrap dependency:

```text
Certbot needs Infisical
        ↓
Infisical needs HTTPS
        ↓
HTTPS needs a certificate
        ↓
Certificate needs Certbot
```

That doesn't provide any useful security improvement; it simply creates a circular dependency.

For bootstrap infrastructure, I therefore keep a small number of credentials outside the secrets manager and protect them directly at the host/filesystem level. Infisical can then handle secrets for applications that start after the core infrastructure is available.

## 3. Certificate reconciliation

The custom Certbot entrypoint periodically reconciles the configured certificates.

For each certificate it:

1. Reads the configured primary hostname and SANs.
2. Checks whether the certificate already exists.
3. Reads the certificate's `Identifiers:` from Certbot.
4. Compares the existing names with the configured names.
5. Issues or updates the certificate if they differ.
6. Runs `certbot renew` for normal renewal processing.
7. Exports any required combined PEM files.

This means changing:

```env
CERTBOT_CERTS=example.com,www.example.com
```

to:

```env
CERTBOT_CERTS=example.com,www.example.com,api.example.com
```

causes the next reconciliation cycle to detect the change and request an updated certificate.

The container stays running and performs the reconciliation periodically:

```env
CERTBOT_RENEW_INTERVAL=12h
```

There is no external cron job involved.

## 4. Keeping Certbot output useful

One thing I changed from a typical Certbot wrapper is logging.

Rather than hiding Certbot output with:

```bash
certbot renew --quiet
```

the wrapper allows Certbot's actual output to reach Docker's logging system.

The wrapper's own messages use structured logfmt:

```text
ts="2026-09-19 12:00:01" level=info component=cert-manager msg="Checking configured certificates."
ts="2026-09-19 12:00:02" level=info component=certbot msg="Certificate renewal completed."
ts="2026-09-19 12:00:03" level=error component=certbot msg="Failed to renew certificate."
```

This works well with Loki because the logs remain human-readable while also being queryable:

```text
{container="certbot"} | logfmt | level="error"
```

Or to isolate Certbot itself:

```text
{container="certbot"} | logfmt | component="certbot" | level="error"
```

This is much more useful than only knowing that the container exited with an error.

## 5. Distributing certificates with Syncthing

Once Certbot has issued a certificate, the files exist under:

```text
/etc/letsencrypt/live/
```

I use Syncthing as the distribution mechanism.

The important distinction is that Syncthing is **not responsible for certificate management**. It only moves already-issued certificate files between trusted machines.

For example:

```text
certbot host
    │
    │ Syncthing
    ▼
/docker-data/syncthing-data/certs/
    │
    ├── traefik-host-1/
    └── traefik-host-2/
```

The synchronization scope is deliberately limited to the certificate directory. I do not synchronize the entire `/etc/letsencrypt` directory.

This prevents unrelated Certbot state, account information, and renewal configuration from being unnecessarily distributed.

Private keys should only be synchronized to machines that actually need them.

## 6. Traefik consumes the files directly

On a Traefik host, the certificate directory is mounted read-only:

```yaml
volumes:
  - /docker-data/syncthing-data/certs:/certs:ro
```

The certificates are then configured through Traefik's file provider.

For example:

```yaml
tls:
  certificates:
    - certFile: /certs/example.com/fullchain.pem
      keyFile: /certs/example.com/privkey.pem

    - certFile: /certs/internal.example.com/fullchain.pem
      keyFile: /certs/internal.example.com/privkey.pem
```

The important detail here is that Traefik does not need access to Certbot itself. It only needs the resulting certificate files.

That makes the two components independently replaceable.

## 7. Reloading Traefik without restarting it

The remaining problem is detecting when Syncthing has delivered a new certificate.

Traefik's file provider can watch a directory:

```yaml
providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true
```

I use a small `cert-watcher` container based on Alpine and `inotify-tools`.

The image is intentionally tiny:

```dockerfile
FROM alpine:3.22

RUN apk add --no-cache inotify-tools

COPY cert-watcher.sh /usr/local/bin/cert-watcher.sh

RUN chmod 0755 /usr/local/bin/cert-watcher.sh

ENTRYPOINT ["/usr/local/bin/cert-watcher.sh"]
```

The container watches the synchronized certificate directory:

```text
/certs
```

and has write access only to the Traefik dynamic configuration directory.

When a certificate changes, it touches a small file:

```text
dynamic/cert-reload.yml
```

The file does not need to contain configuration. Its modification time is enough to cause the file provider to notice a change.

For example:

```yaml
# Certificate reload trigger.
```

The important part is that it remains a valid YAML file. An empty `http:` or `tls:` block should not be used simply as a reload trigger because Traefik can reject an otherwise empty configuration section.

The watcher therefore has no Docker socket and does not restart Traefik.

Its permissions are effectively:

```text
read certificates
        +
write one Traefik dynamic directory
        =
certificate reload trigger
```

That is considerably narrower than giving a helper container access to `/var/run/docker.sock`.

## 8. HTTP to HTTPS

Traefik handles the HTTP-to-HTTPS redirect independently of certificate renewal.

For example, the HTTP entrypoint can redirect to HTTPS:

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"
```

Certificates are then handled by the TLS configuration, while routing and middleware remain separate concerns.

This separation also makes troubleshooting easier.

If HTTP works but HTTPS fails, I can check:

1. certificate issuance,
2. certificate synchronization,
3. certificate files on the Traefik host,
4. Traefik's file-provider configuration,
5. router/TLS configuration.

Each stage can be tested independently.

## 9. Optional combined PEM output

Some applications expect a single PEM file containing both the certificate chain and private key.

For those applications I enable:

```env
CERTBOT_CREATE_COMBINED_CERT=true
CERTBOT_COMBINED_CERT_OUTPUT=/cert-export
```

The generated file is effectively:

```bash
cat fullchain.pem privkey.pem > ssl-cert.pem
```

but the script creates it atomically:

```text
ssl-cert.pem.tmp
       ↓
write complete file
       ↓
chmod 600
       ↓
rename
       ↓
ssl-cert.pem
```

This avoids consumers seeing a partially written certificate.

For example, a generated file can then be synchronized to a service such as Pi-hole without requiring that service to understand Certbot's directory structure.

## 10. The complete renewal path

At this point the operational flow is straightforward:

```text
Certbot
  │
  ├── checks configured certificates
  ├── renews when required
  └── writes certificate files
          │
          ▼
      Syncthing
          │
          ▼
   Traefik certificate directory
          │
          ▼
    cert-watcher detects change
          │
          ▼
    Traefik file-provider reload
```

No container restart is required.

If a certificate expires or renewal fails, the first place to look is Certbot. If Certbot succeeds but Traefik still serves the old certificate, the problem is further down the pipeline.

## 11. Troubleshooting

I generally troubleshoot the system from left to right.

### Check Certbot

```bash
docker logs certbot
```

Look for certificate issuance, renewal, and Cloudflare DNS errors.

### Check the generated certificate

```bash
openssl x509 \
  -in /docker-data/certbot-data/config/live/example.com/fullchain.pem \
  -noout \
  -subject \
  -issuer \
  -dates
```

### Check Syncthing

Verify that the updated files reached the destination host.

```bash
ls -l /docker-data/syncthing-data/certs/example.com/
```

The modification time should correspond to the latest renewal.

### Check Traefik

Look at the Traefik logs and verify that the dynamic configuration was reloaded.

If necessary, manually touch the reload trigger:

```bash
touch dynamic/cert-reload.yml
```

If Traefik reloads correctly after that, the problem is likely in the certificate watcher rather than Traefik itself.

### Check the certificate actually being served

From a client:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates
```

This verifies the certificate being presented by the actual HTTPS endpoint rather than merely checking the file on disk.

## 12. Security considerations

There are a few important boundaries in this design.

**Cloudflare credentials stay on the Certbot host.**

The DNS API credential does not need to be distributed to Traefik hosts.

**Private keys are distributed only where required.**

Syncthing should not synchronize certificate private keys to hosts that do not terminate TLS.

**Traefik does not need the Docker socket for certificate management.**

It may already use the socket for Docker service discovery, but certificate renewal itself does not require additional Docker privileges.

**The certificate watcher does not need the Docker socket.**

It only needs read access to the certificates and write access to its reload trigger.

**Bootstrap secrets are separate from application secrets.**

The Cloudflare credential is required to establish TLS, so placing it behind a service that itself depends on TLS creates an unnecessary dependency cycle.

After the TLS layer is operational, application secrets can be delivered through a secrets-management system such as Infisical.

## Conclusion

The useful part of this setup is not any individual component. Certbot, Cloudflare DNS, Syncthing, and Traefik are all well-established tools.

The practical benefit comes from giving each component a small, clearly defined responsibility:

```text
Certbot      → issue and renew
Syncthing    → distribute
Traefik      → serve
cert-watcher → trigger reload
```

The result is a certificate lifecycle that is automatic but still easy to troubleshoot.

It also establishes a useful bootstrap boundary for the rest of the infrastructure:

```text
Bootstrap
    ↓
TLS
    ↓
HTTPS
    ↓
Secrets Management
    ↓
Applications
```

That ordering avoids making the infrastructure responsible for bootstrapping the very service it needs in order to start.

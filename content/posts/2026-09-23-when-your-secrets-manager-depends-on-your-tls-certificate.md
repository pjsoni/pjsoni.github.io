---
title: "When Your Secrets Manager Depends on Your TLS Certificates"
date: 2026-09-23
slug: "when-your-secrets-manager-depends-on-your-tls-certificates"
summary: "Why bootstrap-critical services should not depend on a secrets manager that itself requires the services they are responsible for securing."
categories: [Infrastructure, Security]
tags: [homelab, containers, secrets-management, tls]
jumbotron:
  meta: true
---

## The Problem

I've been gradually moving secrets in my homelab to {{< external-link >}}[Infisical](https://infisical.com/){{</ external-link >}}, with the goal of getting application credentials out of Docker Compose files and environment variables.

That works well for normal applications.

But I recently ran into an architectural problem with my certificate management stack.

My custom Certbot container is responsible for obtaining and renewing the TLS certificates used by Traefik. Traefik, in turn, provides HTTPS access to the services in my environment—including Infisical itself.

That creates a dependency that is easy to overlook:

```text
Certbot
   │
   │ obtains TLS certificates
   ▼
Traefik
   │
   │ provides HTTPS
   ▼
Infisical
   │
   │ provides secrets
   ▼
Applications
```

At first glance, moving Certbot's Cloudflare API credential into Infisical seems like the obvious thing to do.

But doing that changes the dependency chain to:

```text
Certbot
   │
   ▼
Infisical
   │
   ▼
Traefik
   │
   ▼
TLS certificate
   │
   └──────────────► Certbot
```

Now I have a circular dependency.

## The Bootstrap Problem

Certbot needs a Cloudflare credential because it uses the Cloudflare DNS API to obtain certificates.

Those certificates are needed by Traefik.

Traefik provides the HTTPS endpoint through which Infisical is accessed.

Infisical provides the secret that Certbot would need.

So the system effectively becomes:

```text
Certbot
  requires Infisical
      requires HTTPS
          requires Traefik
              requires Certbot
```

There is no clean starting point.

This is a classic bootstrap problem: **the component responsible for establishing the secure control plane cannot depend on that control plane to start.**

## Not Every Secret Belongs in the Secrets Manager

The solution was not to find a more complicated way to make Infisical work earlier in the boot process.

Instead, I changed the boundary.

Some credentials are inherently **bootstrap secrets**.

In my case, the Cloudflare API credential used by Certbot is one of them.

The important distinction is:

| Secret                            | Storage                | Reason                                              |
| --------------------------------- | ---------------------- | --------------------------------------------------- |
| Cloudflare credential for Certbot | Local bootstrap secret | Required to establish TLS                           |
| Infisical bootstrap credentials   | Local bootstrap secret | Required to establish access to Infisical           |
| Application credentials           | Infisical              | Available after the secure control plane is running |

This results in a much cleaner architecture:

```text
                    BOOTSTRAP
                       │
        ┌──────────────┴──────────────┐
        │                             │
 Cloudflare credential       Infisical bootstrap
        │                             │
        ▼                             │
     Certbot                          │
        │                             │
        ▼                             │
     TLS certs                        │
        │                             │
        ▼                             │
     Traefik ◄────────────────────────┘
        │
        ▼
      HTTPS
        │
        ▼
    Infisical
        │
        ▼
 Application secrets
```

The bootstrap layer is deliberately small.

Everything after that can use centralized secret management.

## Keeping Certbot's Credential as a File

My custom Certbot image already supports a credentials-file configuration:

```sh
CF_CREDS="${CERTBOT_CF_CREDENTIALS:-/cloudflare.ini}"
```

and passes that file to Certbot:

```sh
--dns-cloudflare-credentials "$CF_CREDS"
```

The Compose configuration mounts the credential read-only:

```yaml
volumes:
  - /docker-data/certbot-data/config:/etc/letsencrypt
  - /docker-data/certbot-data/logs:/var/log/letsencrypt
  - /docker-data/certbot-data/cloudflare.ini:/cloudflare.ini:ro
  - /docker-data/certbot-data/renewal-hooks:/etc/letsencrypt/renewal-hooks
  - /docker-data/certbot-data/var-lib-letsencrypt:/var/lib/letsencrypt
```

This has an additional advantage over putting the Cloudflare token into an environment variable.

The credential isn't part of the container's environment:

```text
docker inspect
    └── Config.Env
          └── Cloudflare token
```

Instead, Certbot gets access to the file:

```text
/cloudflare.ini
```

as a read-only mount.

The host-side file is protected with restrictive permissions as well:

```bash
chmod 600 /docker-data/certbot-data/cloudflare.ini
```

The resulting Cloudflare credentials file contains only what Certbot needs:

```ini
dns_cloudflare_api_token = <token>
```

## Why I Didn't Use Infisical Anyway

It would certainly be possible to build a more elaborate solution around this.

For example:

```text
Infisical Agent
      │
      ▼
cloudflare.ini
      │
      ▼
Certbot
```

But that still leaves the fundamental bootstrap question:

**How does the Infisical Agent authenticate and communicate with Infisical before Traefik and TLS are available?**

You can solve that with additional bootstrap credentials, alternative network paths, certificates, HTTP endpoints, or a separate management network.

At that point, however, the architecture is becoming considerably more complicated just to eliminate one small local secret.

That's a trade-off I don't think is worthwhile here.

The purpose of a secrets manager is to reduce complexity and exposure—not to create a dependency graph that makes the infrastructure harder to bootstrap.

## Bootstrap Secrets Should Be Deliberate

This doesn't mean that bootstrap secrets should simply be ignored.

They should be treated differently.

For my environment, that means:

* Keep the number of bootstrap secrets very small.
* Keep them outside application Compose files.
* Don't put them into Git.
* Don't inject them into the general container environment unless necessary.
* Restrict filesystem permissions.
* Mount credential files read-only where possible.
* Document exactly why each bootstrap secret exists.
* Use centralized secret management for everything that doesn't participate in the bootstrap process.
* Rotate bootstrap credentials independently.

The goal isn't:

> "Every secret must be stored in Infisical."

The better goal is:

> **"Every secret should have an appropriate storage and delivery mechanism for the dependency level at which it is required."**

## The Bigger Lesson

Centralized secret management is extremely useful, but it doesn't eliminate the need for a bootstrap layer.

Every infrastructure system has a root of trust.

Something has to exist before the rest of the infrastructure can securely start.

In my case, the dependency chain is:

```text
Bootstrap credentials
        │
        ▼
     Certbot
        │
        ▼
   TLS certificates
        │
        ▼
     Traefik
        │
        ▼
    Infisical
        │
        ▼
 Applications
```

Trying to move every credential into the layer below it can create a circular dependency.

Instead, I've chosen to keep the Cloudflare credential required by Certbot as a deliberately managed bootstrap secret. Once Traefik and HTTPS are operational, Infisical becomes the source of truth for the secrets used by the services further up the stack.

That gives me a much simpler startup path:

**Bootstrap → TLS → HTTPS → Secrets Manager → Applications.**

And, importantly, the system still has a clear starting point when everything is completely down.

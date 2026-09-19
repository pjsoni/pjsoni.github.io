---

layout: post
title: "Automating TLS Certificates with Certbot, Cloudflare DNS, and Traefik"
date: 2026-09-19
slug: automated-tls-certificates-certbot-cloudflare-traefik
categories: [homelab,docker, security]
tags: [certbot,letsencrypt,cloudflare,tls,traefik,docker,syncthing]

---

Managing TLS certificates becomes surprisingly complicated once an environment grows beyond a single server.

A simple setup can use Traefik's built-in ACME support and let Traefik handle everything. But in a larger environment, the certificate issuer, reverse proxies, and services consuming those certificates may live on different hosts.

For my environment, I wanted a design with a few specific properties:

* Certificates should be issued independently of Traefik.
* DNS-01 should be used so internal services do not need to be publicly reachable.
* Certificate renewal should be completely automated.
* Multiple Traefik instances should be able to consume the same certificates.
* Private keys should never be baked into container images.
* Traefik should reload certificates without being restarted.
* The certificate system should work even when higher-level infrastructure such as a secret-management platform is unavailable.

The resulting architecture separates certificate **issuance**, **distribution**, and **consumption**.

## Architecture

At a high level, the flow looks like this:

```text
                         Internet
                            │
                            │ DNS-01
                            ▼
                    ┌─────────────────┐
                    │    Cloudflare   │
                    │      DNS API    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │     Certbot     │
                    │                 │
                    │ Issue / Renew   │
                    └────────┬────────┘
                             │
                      Certificate files
                             │
                             ▼
                    ┌─────────────────┐
                    │ Distribution    │
                    │     Layer       │
                    │   Syncthing     │
                    └───────┬─────────┘
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │ Traefik  │   │ Traefik  │   │ Traefik  │
       │ Host A   │   │ Host B   │   │ Host C   │
       └────┬─────┘   └────┬─────┘   └────┬─────┘
            │              │              │
            ▼              ▼              ▼
         Services       Services       Services
```

This separation is important.

Certbot is responsible for obtaining certificates.
Syncthing is responsible for transporting them.
Traefik is responsible for serving them.

Each component has a relatively small and well-defined responsibility.

---

## Why DNS-01?

There are two common ACME challenge mechanisms:

* HTTP-01
* DNS-01

HTTP-01 requires the ACME server to reach a web server over HTTP. That is inconvenient for infrastructure where many services are intentionally private.

DNS-01 solves that problem by proving control of the domain through a DNS record.

That means certificates can be issued for services that are:

* internal-only,
* behind a firewall,
* not exposed to the Internet,
* hosted on private VLANs, or
* otherwise unreachable from the public Internet.

For a homelab or private infrastructure, DNS-01 is therefore a natural fit.

The Certbot container uses the Cloudflare DNS API to create the temporary challenge records required by Let's Encrypt.

---

# Using a Custom Certbot Container

Instead of treating Certbot as a one-shot command, I use a small custom container around Certbot.

For example:

```yaml
services:
  certbot:
    image: ghcr.io/example-org/certbot-cloudflare:v5.7.0
    restart: unless-stopped
```

The important part is not the image itself. The important part is that the container provides a predictable lifecycle around the standard Certbot tooling.

The wrapper handles tasks such as:

1. Initial certificate provisioning.
2. Reconciliation of the desired certificate configuration.
3. Persistent Certbot state.
4. Periodic renewal checks.
5. Renewal hooks.
6. Deployment/distribution after successful renewal.

Certbot itself still performs the actual certificate issuance and renewal logic.

This distinction is useful because the wrapper does not need to reimplement ACME or certificate-renewal logic.

---

## Why Not Simply Run `certbot renew` Manually?

A certificate-management container should be self-contained.

Rather than depending on a host cron job, the container can periodically execute:

```text
certbot renew
```

The renewal loop can run approximately every 12 hours.

That does **not** mean a certificate is renewed every 12 hours.

Certbot examines the existing certificate and its renewal configuration and determines whether renewal is actually necessary.

The container therefore acts more like a small certificate-management service than a one-time command runner.

---

# Persisting Certbot State

Certbot maintains important state outside the certificate files themselves.

The persistent storage should include the relevant Certbot directories, including:

```text
/etc/letsencrypt
/var/lib/letsencrypt
/var/log/letsencrypt
```

The exact Docker volume layout is an implementation detail, but the principle is important:

> Never make certificate state dependent on the lifecycle of the container.

If the container is recreated, Certbot should still know which certificates exist, which renewal configurations are active, and where its certificates are stored.

This also makes upgrades much safer.

---

# Cloudflare Credentials

The Cloudflare API credential is supplied to Certbot at runtime rather than being embedded in the image.

A typical configuration uses a credentials file with restrictive permissions:

```text
dns_cloudflare_api_token = <token>
```

The token should have the minimum permissions necessary to modify DNS records for the required zone.

The important security principles are:

* Do not bake credentials into the image.
* Do not commit credentials to Git.
* Do not expose the credentials unnecessarily through container configuration.
* Prefer a narrowly scoped Cloudflare API token.
* Mount the credentials read-only where possible.

The certificate container should have access to the credential, but the credential should not become part of the application image.

---

# Certificate Naming and Reconciliation

One useful feature of the custom wrapper is treating the desired certificate configuration as declarative.

For example, a certificate may represent:

```text
example.com
*.example.com
```

Using stable Certbot `--cert-name` semantics makes it possible to reconcile the desired certificate configuration instead of blindly creating new certificate lineages every time the container starts.

This is particularly important for automated environments.

The container should be safe to restart without producing a new certificate lineage on every startup.

---

# Automated Renewal

The renewal lifecycle is intentionally simple:

```text
Container starts
      │
      ▼
Load persistent Certbot state
      │
      ▼
Ensure desired certificates exist
      │
      ▼
Periodic renewal check
      │
      ▼
   certbot renew
      │
      ├── Nothing to renew
      │
      └── Certificate renewed
                  │
                  ▼
             Deploy hook
                  │
                  ▼
          Distribute new files
```

The important part is the **deploy hook**.

A successful renewal should trigger the next stage of the system.

This prevents downstream systems from reacting to every renewal check. They only need to react when Certbot actually produces a new certificate.

---

# Certificate Distribution

The certificate issuer and the reverse proxy do not have to live on the same host.

That is one of the main reasons to separate certificate management from Traefik.

For example:

```text
Certificate host
      │
      │ renewed certificate
      ▼
Certificate distribution
      │
      ├──────────► Traefik A
      │
      ├──────────► Traefik B
      │
      └──────────► Traefik C
```

This also makes it possible to add another reverse-proxy host without changing the certificate-issuance mechanism.

The certificate simply becomes another artifact that needs to be distributed to trusted consumers.

---

# Why Syncthing?

Syncthing is used here as the **certificate transport and distribution mechanism**.

This is a useful pattern whenever certificate issuance and certificate consumption happen on different machines.

Certificates are already filesystem-based artifacts, so a secure file synchronization mechanism is a natural way to move them between trusted hosts.

The architecture becomes:

```text
Certbot
   │
   │ certificate files
   ▼
Certificate store
   │
   ▼
Syncthing publisher
   │
   ├──────────────► Syncthing receiver ──► Traefik A
   │
   ├──────────────► Syncthing receiver ──► Traefik B
   │
   └──────────────► Syncthing receiver ──► Traefik C
```

This is more than simply reusing an existing service. Syncthing is a reasonable fit for this particular problem because it provides:

* peer-to-peer file synchronization,
* encrypted transport,
* automatic synchronization,
* support for multiple hosts,
* filesystem-based operation,
* no certificate-specific API integration,
* receive-only destinations.

A receive-only destination is especially useful for certificate consumers. The Traefik host needs to receive certificates, but it should not be able to modify the source certificate repository.

## A General Pattern

The broader architectural pattern is:

```text
Certificate Issuer
        │
        ▼
Certificate Store
        │
        ▼
Distribution Layer
        │
   ┌────┼────┐
   ▼    ▼    ▼
 Proxy Proxy Proxy
```

Syncthing is one implementation of the distribution layer.

Other environments could use:

* object storage,
* configuration management,
* deployment automation,
* secure file transfer,
* orchestration tooling.

The important architectural decision is separating **issuance from distribution**.

---

## Keep the Synchronization Scope Small

There is an important security consideration here.

Do not synchronize the entire:

```text
/etc/letsencrypt
```

tree unless there is a specific reason to do so.

The receiving systems generally need only the certificate and private key required by Traefik, such as:

```text
fullchain.pem
privkey.pem
```

The exact filenames depend on the certificate layout.

The smaller the synchronized dataset, the smaller the blast radius.

However, distributing `privkey.pem` means every receiving system becomes part of the TLS trust boundary.

A compromised Traefik host therefore represents a potential compromise of the corresponding private key.

Syncthing makes distribution convenient; it does not eliminate that security responsibility.

---

# Traefik as the Certificate Consumer

Traefik does not issue the certificates in this design.

Instead, it reads certificates from the filesystem.

This creates a clean separation:

```text
Certbot
  └── "I manage certificates."

Syncthing
  └── "I distribute certificates."

Traefik
  └── "I serve certificates."
```

Traefik's certificate configuration can then reference the synchronized files.

For example:

```yaml
tls:
  certificates:
    - certFile: /certs/example.com/fullchain.pem
      keyFile: /certs/example.com/privkey.pem
```

The certificate directory should be mounted read-only into Traefik.

Traefik has no reason to modify the certificate store.

---

# Reloading Certificates Without Restarting Traefik

Synchronizing a new certificate onto disk does not necessarily mean the running Traefik process immediately starts serving it.

The second problem is therefore:

> How do we tell Traefik that the certificate files changed?

Rather than restarting Traefik, I use a small certificate watcher.

The watcher monitors the certificate directory using filesystem events such as:

```text
close_write
moved_to
```

When a relevant change is detected, the watcher waits briefly to debounce multiple filesystem events and then updates a small trigger file such as:

```text
cert-reload.yml
```

The Traefik file provider notices the change and reloads the dynamic configuration.

The flow becomes:

```text
Syncthing
    │
    ▼
Certificate file changes
    │
    ▼
cert-watcher
    │
    │ debounce
    ▼
Touch trigger file
    │
    ▼
Traefik file provider
    │
    ▼
Dynamic configuration reload
    │
    ▼
New certificate served
```

This avoids restarting Traefik and also avoids giving the watcher access to the Docker socket.

That last point is important: a certificate watcher should not need `/var/run/docker.sock`.

---

# Dynamic Traefik Configuration

Keeping Traefik's dynamic configuration separate from the static configuration makes the system easier to understand and maintain.

A structure such as this works well:

```text
dynamic/
├── tls.yml
├── middlewares.yml
├── routers.yml
└── cert-reload.yml
```

The file provider watches the dynamic configuration directory.

A small provider throttle can also help avoid unnecessary reloads when several filesystem changes happen close together:

```yaml
providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true

providersThrottleDuration: 10s
```

The exact value is not particularly important. The goal is to prevent a burst of filesystem events from producing a burst of configuration reloads.

---

# HTTP to HTTPS

The public-facing entry points are configured so that HTTP redirects to HTTPS.

Conceptually:

```text
HTTP :80
   │
   ▼
301/308 redirect
   │
   ▼
HTTPS :443
```

This keeps the user-facing behavior simple while allowing all normal application traffic to use TLS.

---

# Security Middleware

TLS is only one part of the reverse-proxy security model.

The Traefik configuration also uses middleware for things such as:

* security headers,
* compression,
* network access control.

A typical middleware chain might look like:

```text
internal-chain
    │
    ├── security-headers
    ├── compression
    └── internal-allowlist
```

However, security middleware should not be blindly applied to every application.

One lesson from this setup was that globally applied headers can conflict with application-specific behavior.

For example, an application may require different handling for:

```text
Referrer-Policy
X-XSS-Protection
X-Frame-Options
```

The correct configuration depends on the application.

Security middleware should therefore be treated as a policy layer that may require application-specific exceptions.

---

# Network-Level Access Control

The reverse proxy also acts as a network boundary.

For example, a management-facing Traefik instance can allow access from the normal client network as well as a restricted management network, while other Traefik instances remain limited to their intended network.

The important design principle is:

> Do not use one broad allowlist simply because all Traefik instances share the same configuration style.

Different entry points can have different trust boundaries.

Keep the actual network ranges environment-specific rather than copying them into a reusable configuration.

---

# Pin Infrastructure Versions

One of the more valuable operational lessons was the importance of version pinning.

A major Traefik upgrade was tested and caused compatibility problems with the existing stack. The environment was subsequently returned to the known-good Traefik 2.x release.

The lesson is not that upgrades should be avoided.

The lesson is:

> Foundational infrastructure should be upgraded deliberately rather than implicitly.

For production-like infrastructure, avoid relying on:

```yaml
image: traefik:latest
```

Prefer an explicit version:

```yaml
image: traefik:v2.11.56
```

The same principle applies to custom infrastructure images.

Explicit versions make troubleshooting much easier because the environment is reproducible.

---

# Bootstrap and Dependency Ordering

There is another important architectural consideration when certificate management is part of the core infrastructure.

The certificate system must not depend on services that themselves require the certificates.

For example:

```text
Certbot
   │
   ▼
Certificates
   │
   ▼
Traefik
   │
   ▼
HTTPS services
   │
   ▼
Secret management / applications
```

This prevents a circular dependency such as:

```text
Infisical requires HTTPS
       │
       ▼
Traefik requires certificate
       │
       ▼
Certificate service requires Infisical
       │
       └────── circular dependency
```

The certificate-management component should therefore be capable of bootstrapping independently of higher-level services.

This is one reason the certificate credentials and initial configuration should be available to the core infrastructure without depending on the secret-management system that is being protected by that same infrastructure.

---

# The Complete Renewal Lifecycle

Putting everything together:

```text
                    ┌──────────────┐
                    │    Certbot   │
                    └──────┬───────┘
                           │
                    DNS-01 challenge
                           │
                           ▼
                    ┌──────────────┐
                    │   Cloudflare │
                    └──────┬───────┘
                           │
                           ▼
                    Certificate issued
                           │
                           ▼
                    Deploy hook runs
                           │
                           ▼
                    ┌──────────────┐
                    │  Syncthing   │
                    └──────┬───────┘
                           │
                 certificate synchronized
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       ┌─────────────┐           ┌─────────────┐
       │   Traefik   │           │   Traefik   │
       │    Host A   │           │    Host B   │
       └──────┬──────┘           └──────┬──────┘
              │                         │
              ▼                         ▼
       cert-watcher                cert-watcher
              │                         │
              ▼                         ▼
       trigger file                 trigger file
              │                         │
              └────────────┬────────────┘
                           ▼
                  Traefik reloads
                           │
                           ▼
                New certificate served
```

The important thing about this design is that every stage can be inspected independently.

---

# Troubleshooting the Certificate Pipeline

When certificate renewal appears broken, avoid looking at the system as a single component.

There are several independent stages:

```text
Certificate issued?
       │
       ▼
Certificate distributed?
       │
       ▼
Certificate watcher triggered?
       │
       ▼
Traefik reloaded?
       │
       ▼
Traefik serving new certificate?
```

Check each stage separately.

### 1. Check Certbot

Look at the renewal configuration and logs.

Verify:

* the certificate exists,
* renewal configuration is valid,
* Cloudflare credentials work,
* DNS-01 challenges succeed.

### 2. Check the source certificate

Verify the actual certificate files on the certificate-management host.

The certificate's expiration date should reflect the successful renewal.

### 3. Check Syncthing

Confirm that the updated files reached the receiving Traefik host.

### 4. Check the watcher

Verify that filesystem changes were detected and the reload trigger was updated.

### 5. Check Traefik

Verify that the file provider detected the dynamic configuration change.

### 6. Check the certificate presented to clients

Finally, inspect the certificate actually being served by the endpoint.

This distinction is critical:

> A renewed certificate on disk is not necessarily the certificate loaded by Traefik, and the certificate loaded by Traefik is not necessarily the certificate a client is currently receiving.

Breaking the problem into those stages makes troubleshooting much faster.

---

# Strengths of This Architecture

## Clear separation of responsibilities

Each component has a narrow role:

* Certbot → certificate lifecycle
* Cloudflare → DNS challenge
* Syncthing → certificate transport
* Traefik → TLS termination and routing
* cert-watcher → reload notification

That makes the architecture easier to reason about.

## Works with private services

DNS-01 eliminates the requirement for the ACME server to reach internal applications.

## Supports multiple reverse proxies

A single certificate-management system can distribute certificates to several Traefik instances.

## Fully automated renewal

Once configured, certificate renewal does not require manual intervention.

## No Traefik restart required

Certificates can be reloaded dynamically after synchronization.

## No Docker socket required for certificate watching

The watcher can operate entirely through filesystem events.

## Good bootstrap characteristics

The certificate infrastructure can remain independent of higher-level applications and secret-management services.

---

# Weaknesses and Tradeoffs

No architecture is free of tradeoffs.

## Private-key distribution

The biggest concern is that private keys must reach every Traefik host.

Those systems therefore become part of the certificate trust boundary.

## More components

This design introduces:

* Certbot,
* Cloudflare DNS,
* Syncthing,
* cert-watcher,
* Traefik file-provider configuration.

Traefik's built-in ACME functionality is considerably simpler for smaller environments.

## Custom code

The Certbot wrapper and certificate watcher are additional software that must be maintained and upgraded.

The more custom components there are, the more responsibility the operator has for testing them.

## Indirect reload mechanism

The certificate reload path involves several steps:

```text
file change
 → watcher
 → debounce
 → trigger file
 → Traefik provider
 → certificate reload
```

It works well, but it is more complex than a proxy that manages the entire certificate lifecycle itself.

## Distribution failures are independent of renewal

Certbot may successfully renew a certificate while Syncthing is temporarily unable to distribute it.

This means monitoring should not stop at certificate issuance.

The health of the entire pipeline matters.

## Infrastructure upgrades can have a wide blast radius

Traefik and other foundational components should be version-pinned and upgraded deliberately.

---

# Security Considerations

A few practices are particularly important for this architecture.

### Use least-privilege Cloudflare credentials

The DNS API token should only have the permissions required for the relevant DNS zone.

### Keep private keys out of Git

Certificates and especially private keys should never become source-controlled configuration.

### Restrict certificate synchronization

Only synchronize the files that consumers actually need.

### Use read-only mounts

Traefik should consume certificates through read-only filesystem mounts.

### Use receive-only destinations where appropriate

A Traefik host should generally receive certificates rather than being allowed to modify the source repository.

### Keep the certificate infrastructure isolated

The certificate-management host is part of the security infrastructure and should be treated accordingly.

### Pin versions

Do not allow an unexpected image update to change certificate-management or reverse-proxy behavior.

---

# Lessons Learned

The most important lesson from this architecture is that TLS management becomes easier to reason about when it is divided into three distinct problems:

```text
1. How do I obtain the certificate?
2. How do I distribute the certificate?
3. How do I make consumers reload it?
```

Those problems do not necessarily need to be solved by the same application.

For this architecture:

```text
Obtaining       → Certbot + Cloudflare DNS
Distributing    → Syncthing
Reloading       → cert-watcher + Traefik file provider
Serving         → Traefik
```

That separation makes it possible to evolve one part without redesigning everything else.

For example, Syncthing could eventually be replaced with another secure distribution mechanism without changing how Certbot obtains certificates or how Traefik consumes them.

Likewise, additional Traefik hosts can be added without creating additional ACME clients.

---

# Conclusion

A reverse proxy does not have to be responsible for every aspect of TLS management.

For a small deployment, Traefik's built-in ACME support may be all that is necessary. But when certificate issuance, reverse proxies, and infrastructure services span multiple hosts, separating those responsibilities can provide a more flexible architecture.

The resulting design is straightforward:

```text
                 ┌────────────────────┐
                 │      Certbot        │
                 │  Certificate Mgmt   │
                 └─────────┬──────────┘
                           │
                           ▼
                 ┌────────────────────┐
                 │     Syncthing      │
                 │ Certificate Transport│
                 └─────────┬──────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Traefik A    Traefik B    Traefik C
              │            │            │
              └────────────┼────────────┘
                           ▼
                       Services
```

The key architectural idea is not any particular container or tool.

It is the separation of **certificate lifecycle, certificate distribution, and certificate consumption**.

Once those responsibilities are separated, automated TLS becomes easier to scale, troubleshoot, and adapt as the infrastructure grows.

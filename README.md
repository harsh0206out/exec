# Multi-Account Multi-Server Setup Plan (Updated)

### Accounts & Servers Overview

| Account (Country) | Server 1     | IP Type | Server 2 | IP Type |
| ----------------- | ------------ | ------- | -------- | ------- |
| Nestle (HK)       | KitKat\*     | Public  | BarOne   | Private |
| Cadbury (IN)      | DairyMilk\*  | Public  | FiveStar | Private |
| Hersheys (AT)     | ExoticDark\* | Public  | Kisses   | Private |

\*Servers with public IPv4. All paired servers are in the same subnet and VNet per account.

---

## Step 0: Planning Networking & Access

* Ensure SSH access to all servers (private key-based).
* Keep VNet/subnet consistent per account.
* Non-public servers (BarOne, FiveStar, Kisses) reachable only via Cloudflare Tunnel.

---

## Step 1: Cloudflare Tunnel Setup

* Install `cloudflared` on BarOne, FiveStar, Kisses.
* Create per-server tunnels with unique hostnames/subdomains:

  * `barone.cf.mydomain.com`
  * `fivestar.cf.mydomain.com`
  * `kisses.cf.mydomain.com`
* Connect tunnels to respective backend ports (PHP-FPM/Nginx).
* Ensure tunnels are persistent via systemd.

---

## Step 2: Base Server Setup

* Update Ubuntu packages.
* Install Nginx/Apache, firewall rules, fail2ban, monitoring.
* Configure PHP-FPM backend.
* Public servers reachable directly; private servers only via tunnels.

---

## Step 3: PHP-FPM & Application Setup

* Install PHP 8.3 + FPM + required extensions.
* Configure PHP-FPM pools for app user.
* Deploy PHP application code uniformly.
* Configure Nginx reverse proxy to PHP-FPM.
* Enable opcache, caching, and performance tuning.

---

## Step 4: SSL / Origin Certificates (Updated)

**Current:** CF origin certificates on all servers.

**Better Plan:**

1. **Centralized Certificate Management:**

   * Use Vault or Cloudflare API to manage origin certificates centrally.
   * Push certificates to all servers automatically.
2. **Mutual TLS Option:**

   * Enable mTLS between Cloudflare Tunnel and backend for added security.
3. **Auto-Renewal & Verification:**

   * Automate certificate renewal and verification to prevent downtime.
   * Optionally use scripts to validate SSL post-renewal.
4. **Deployment:**

   * Apply origin certificates to all servers (public & tunnel).
   * Configure Nginx/Apache HTTPS listener.
   * Redirect HTTP → HTTPS.

---

## Step 5: DNS & Domain Mapping (Updated)

**Strategy:**

* Public IP Servers → A records:

  * `kitkat.mydomain.com` → Public IPv4
  * `dairymilk.mydomain.com` → Public IPv4
  * `exoticdark.mydomain.com` → Public IPv4

* Tunnel Servers → CNAME → Cloudflare Tunnel hostname:

  * `barone.mydomain.com` → `barone.cf.mydomain.com`
  * `fivestar.mydomain.com` → `fivestar.cf.mydomain.com`
  * `kisses.mydomain.com` → `kisses.cf.mydomain.com`

* Cloudflare Proxy:

  * Enable orange-cloud for tunnel servers.
  * Optionally proxy public servers via CF for caching/SSL.

* Future-proofing:

  * Keep DNS mapping modular for adding new servers or regions.
  * Maintain consistent naming convention for easier Worker logic integration.

---

## Step 6: Monitoring & Maintenance

* Centralized logging/monitoring (CloudWatch/Prometheus + Grafana).
* Backup plan for application and database.
* Update PHP, system packages, `cloudflared` regularly.
* Maintain firewall rules and Cloudflare policies consistently.

---

## Step 7: Scaling/Extensibility

* For future accounts:

  * 2 servers per account (public + private).
  * Tunnel setup for private servers.
  * DNS + SSL + PHP-FPM uniform configuration.
* Optionally automate deployments via Ansible/Terraform.

---

## Step 8: Cloudflare Worker for Geo & Load Routing

**Objective:** Route users to nearest region and distribute load between two servers per region while keeping domain same.

**Worker Logic:**

1. **Country Detection:**

   * Use `request.cf.country` to detect user region.

2. **Region Mapping:**

   * `HK`: SE Asia, East Asia, Oceania (Mongolia → Japan → Australia etc.)
   * `AT`: Europe + North Africa (NA → Turkey → Norway → GB etc.)
   * `IN`: South Asia, Middle East, East Africa, and remaining world for now

3. **Round-Robin Selection:**

   * Each region has 2 servers (PIP + Tunnel).
   * Randomly choose one server per request for load distribution.

4. **Proxy Request:**

   * Use `fetch()` in Worker to forward request to selected server.
   * Preserve SSL, headers, cookies so browser sees `mydomain.com`.

5. **Response Handling:**

   * Worker returns the server response directly to client.

**Benefits:**

* Geo-routing with country-level accuracy.
* Round-robin load balancing within region.
* Users always see consistent domain (`mydomain.com`).
* Fully compatible with Cloudflare Free Tier.

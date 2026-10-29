# DOLE Regional Office VII — HTML Monitoring & Geotagging Prototype

**Files:** `index.html` is the complete standalone front-end. The `deploy/` folder contains an **optional** Apache VirtualHost example for TCP port **83**.

## What works

- Dashboard with counts of activities, beneficiaries, completed, and ongoing activity statuses.
- Map with program-colored markers and popup information, using Leaflet + OpenStreetMap.
- Filters by field office and program, including a searchable records table.
- Add, edit, delete, and geotag activity records.
- Manually triggered place lookup through OpenStreetMap Nominatim; map click performs reverse lookup.
- Location search results **must verify Cebu or Bohol** and match the selected office: CPFO (Cebu excluding Tri-City), TCFO (Cebu City, Mandaue City, Lapu-Lapu City), BFO (Bohol).
- Saves changes to browser **localStorage**. CSV export and JSON backup/restore are provided.
- Starts with **10 clearly marked sample activities**. These are invented examples, *not actual DOLE program records*.

## Important limitations

This HTML version is a **client-side prototype**. It has no PHP/MySQL backend, user authentication, audit log, multi-user synchronization, or secure storage. Data is kept in the local browser and may be erased when site storage is cleared. This prototype is **not appropriate for real beneficiary details or confidential DOLE records**. Use simulated data and move to the PHP/MySQL version before production rollout.

Maps, icons and Bootstrap load from CDNs, and map/location search require internet access. For reliable Nominatim search, serve `index.html` through Apache over HTTP rather than `file://` (browsers may not send a valid HTTP Referer from a file). Public Nominatim usage policy: https://operations.osmfoundation.org/policies/nominatim/ — searches are initiated by explicit clicks/Enter only, not autocomplete; client code spaces requests, and caches search results. A production deployment should use an agency-controlled geocoder/proxy/cache and approved geospatial boundary data; a response's textual administrative address is **not a legal GIS boundary guarantee**.

## Quick test (Windows)

Open `index.html` in a browser for the dashboard and sample records. For the geocoding search, prefer a local web server or Apache because opening a `file://` page may prevent a proper HTTP Referer.

## Ubuntu + Apache, dedicated port 83

**The steps below use a separate path and do not modify `/var/www/html` or your SSH port 8022.** They configure a dedicated Apache site on port 83. If another Apache VirtualHost already serves port 83 (such as `dole-monitor-83.conf`), **do not enable both without reviewing `sudo apache2ctl -S`**. Either disable the unused test site or adjust the existing VirtualHost to point at the HTML directory.

### 1. Upload and extract

From Windows PowerShell (if SSH is `itunit` on port `8022`):

```powershell
scp -P 8022 "$env:USERPROFILE\Downloads\dole-ro7-monitor-html.zip" itunit@49.157.74.165:/home/itunit/
```

On Ubuntu:

```bash
sudo apt install -y unzip
mkdir -p /tmp/dole-html-extract
unzip -o /home/itunit/dole-ro7-monitor-html.zip -d /tmp/dole-html-extract
sudo mkdir -p /var/www/dole-ro7-html
sudo cp /tmp/dole-html-extract/dole-ro7-monitor-html/index.html /var/www/dole-ro7-html/index.html
sudo chown -R root:www-data /var/www/dole-ro7-html
sudo chmod 755 /var/www/dole-ro7-html
sudo chmod 644 /var/www/dole-ro7-html/index.html
```

### 2. Check Apache port 83

```bash
sudo ss -tulpn | grep ':83 ' || true
sudo grep -n 'Listen 83' /etc/apache2/ports.conf || true
sudo apache2ctl -S
```

If and only if `Listen 83` is missing, add it *once* to `/etc/apache2/ports.conf` alongside your existing `Listen 80` (do not replace existing entries):

```bash
sudo nano /etc/apache2/ports.conf
```

### 3. Add the HTML site only when port 83 is not assigned to another active application

```bash
sudo cp /tmp/dole-html-extract/dole-ro7-monitor-html/deploy/dole-ro7-html-83.conf /etc/apache2/sites-available/dole-ro7-html-83.conf
sudo a2ensite dole-ro7-html-83.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

If an *unused test site* such as `dole-monitor-83.conf` is enabled on port 83, first review it with `sudo apache2ctl -S`. Disable it only if you know it is not hosting a live app:

```bash
# Only when the old site is confirmed unused:
sudo a2dissite dole-monitor-83.conf
```

Then repeat `sudo apache2ctl configtest` and `sudo systemctl reload apache2`.

### 4. Confirm connectivity

```bash
curl -I http://127.0.0.1:83/
sudo ss -tulpn | grep ':83 '
```

If external network access is approved and UFW is active:

```bash
sudo ufw allow 83/tcp
```

If this server uses a router or upstream firewall, that device must also allow/forward TCP 83.

**Demo URL:** `http://49.157.74.165:83/`

> Port 83 over plain HTTP is not secure for real credentials or program data. Set up an HTTPS endpoint (or use your organization's VPN) before a live deployment.

## Transition to PHP/MySQL

The record shape mirrors the requested schema (`field_office_id`, `program_code`, `activity_title`, `barangay`, `municipality_city`, `province`, `latitude`, `longitude`, `status`, `beneficiaries_count`, `date_conducted`, `remarks`). PHP PDO + MySQL APIs can replace the localStorage reads/writes while keeping the front-end design. Protect API writes with authentication, authorization, CSRF checks, server-side field-office validation, and audit logging.

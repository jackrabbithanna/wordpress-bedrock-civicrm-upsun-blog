---
title: "CiviCRM with WordPress Bedrock on Upsun: Installation Guide"
author: Mark Hanna (jackrabbithanna)
date: 2026-05-06
description: A production-ready setup for deploying CiviCRM on top of WordPress Bedrock on Upsun, using a public Composer template that bundles the Upsun config, build hooks, mounts, nginx rules, and cron jobs needed for a real CiviCRM site.
---

# CiviCRM with WordPress Bedrock on Upsun

Recently I published the [CiviCRM with Drupal 11 on Upsun guide](https://developer.upsun.com/posts/hands-on/civicrm-drupal-11-on-upsun), and a common follow-up question I got was: *"Does the same approach work for WordPress?"* The answer is yes — and arguably the WordPress side is even tidier when you start from [Roots Bedrock](https://roots.io/bedrock/) instead of stock WordPress.

This guide walks through deploying CiviCRM on top of a Bedrock-based WordPress install on Upsun, using a public Composer template I developed at [github.com/Skvare/upsun-wordpress-bedrock-civicrm-template](https://github.com/Skvare/upsun-wordpress-bedrock-civicrm-template). The template encodes all the small but important decisions — read-only filesystem layout, separate CiviCRM database, mounted persistent directories, nginx rules to block direct PHP execution in upload paths, and crons for both WordPress and CiviCRM — so you don't have to rediscover them.

If you've read the Drupal version, the shape will look familiar. The hosting platform's strengths are the same: Upsun's PHP inage, separate persistent mounts, MariaDB with multiple schemas, and source operations for auto-update all map cleanly onto how a healthy CiviCRM install actually wants to live.

## Why Bedrock?

Stock WordPress puts everything — core, plugins, uploads, `wp-config.php`, and your application code — in one big bucket at the web root. That works on shared hosting but fights you on a PaaS, where you want a clear split between immutable build artifacts (deployed via Git) and writable persistent data (mounts).

Bedrock fixes this by:

- Moving the WordPress core into `web/wp/` so it can be managed entirely by Composer.
- Putting plugins, themes, and uploads under `web/app/` (the equivalent of `wp-content/`).
- Loading config from environment variables via `vlucas/phpdotenv`, which Upsun fills in for you.
- Generating salts and DB credentials at runtime instead of committing them.

Drop CiviCRM on top of that, and you get a layout where the immutable parts (core, plugins, civi-core, civi extensions installed via Composer) deploy cleanly, and the writable parts (uploads, cache, civi templates_c, civi custom dirs) live on Upsun mounts.

## Prerequisites

Before pushing anything, you'll want:

- An [Upsun](https://upsun.com) account and a project created in the console.
- The [Upsun CLI](https://docs.upsun.com/administration/cli.html) installed locally.
- PHP 8.1+ and Composer 2.x for local development.
- Four CiviCRM environment variables generated and ready to set in the Upsun console:
    - `CIVICRM_SITE_KEY`
    - `CIVICRM_CRED_KEYS`
    - `CIVICRM_SIGN_KEYS`
    - `CIVICRM_DEPLOY_ID`

Use the [CiviCRM documentation on encryption keys](https://docs.civicrm.org/sysadmin/en/latest/setup/encryption/) to generate `CRED_KEYS` and `SIGN_KEYS`. `SITE_KEY` is a long random string of your own choosing. `DEPLOY_ID` is used to invalidate caches and asset URLs across deploys — set it to anything stable per environment (a UUID works well).

Set these in the Upsun console under **Variables**, marked *Visible at runtime*. The application config declares them as empty placeholders so Upsun knows to inject them:

```yaml
variables:
  env:
    CIVICRM_CRED_KEYS: ''
    CIVICRM_SIGN_KEYS: ''
    CIVICRM_SITE_KEY: ''
    CIVICRM_DEPLOY_ID: ''
```

## Quick start

```bash
git clone https://github.com/Skvare/upsun-wordpress-bedrock-civicrm-template.git my-civicrm-site
cd my-civicrm-site
upsun project:set-remote <your-project-id>
git push upsun main
```

Once the build finishes, open the project URL, run through the standard WordPress installer, and then activate the CiviCRM plugin. The rest of this article explains *why* the template is structured the way it is, so you can adapt it to your own needs.

## Local development

For local work, the template uses a conventional `.env` file:

```bash
composer install
cp .env.example .env
# edit .env — set DB credentials, WP_HOME, AUTH_KEY/SALT pairs, and CIVICRM_* keys
```

`.env` is only read when not on Upsun; in production, the `.environment` shell file (covered below) builds the same variables from Upsun's relationship and route data. Commit Bedrock's `.env.example` so collaborators have a starting point, but never commit `.env` itself.

## The Upsun configuration

Everything Upsun needs to build and run the site lives in a single file: `.upsun/config.yaml`. Let's walk through it section by section.

### Application stack

```yaml
applications:
  wordpress:
    type: 'php:8.3'
    dependencies:
      php:
        composer/composer: '^2.1'
    runtime:
      extensions:
        - opcache
        - redis
        - sodium
        - apcu
        - blackfire
        - imagick
        - mbstring
```

PHP 8.3 with the extensions CiviCRM actually uses: `opcache` and `apcu` for performance, `redis` if you choose to add a Redis service for caching, `sodium` for modern crypto, `imagick` for receipt/contact photo handling, `mbstring` for multi-byte string operations, and `blackfire` for production profiling when you need it.

If you don't have a Blackfire subscription, drop that line — listing an extension that isn't licensed will fail the build.

### Database: one service, two schemas

I recommend installing CiviCRM in its own database. On Upsun you don't need to provision two separate services — MariaDB supports multiple schemas in one instance, with separate endpoints per schema:

```yaml
services:
  db:
    type: mariadb:11.8
    configuration:
      properties:
        max_allowed_packet: 64
        max_heap_table_size: 32
        table_definition_cache: 2000
        table_open_cache: 2000
        default_charset: utf8mb4
        default_collation: utf8mb4_unicode_ci
        innodb_adaptive_hash_index: 0
        optimizer_use_condition_selectivity: 4
      schemas:
        - main
        - civicrm
      endpoints:
        mysql:
          default_schema: main
          privileges:
            main: admin
            civicrm: admin
        civicrm:
          default_schema: civicrm
          privileges:
            main: admin
            civicrm: admin
```

A few things worth flagging:

- `max_allowed_packet: 64` (MB) gives CiviCRM enough headroom for bulk imports and large mailings; the default is too small.
- `default_charset: utf8mb4` and the matching collation are non-negotiable for international data and emoji.
- `innodb_adaptive_hash_index: 0` is a CiviCRM-specific tuning hint — the AHI tends to hurt more than help on CiviCRM's query patterns, especially under contention.
- Two endpoints (`mysql` and `civicrm`) let you give each connection a different default schema while still allowing CiviCRM to reach into the WordPress (`main`) schema for user lookups via the UF (User Framework) tables.

The relationship is wired in the application block:

```yaml
relationships:
  database: 'db:mysql'
```

### Mounts: where writable data lives

Bedrock plus CiviCRM has four directories that absolutely need to be writable at runtime, and they need to *not* be in your Git repo:

```yaml
mounts:
  '/web/app/uploads':
    source: storage
    source_path: 'uploads'
  '/web/app/wp-content/cache':
    source: storage
    source_path: 'cache'
  '/tmp':
    source: tmp
    source_path: 'tmp'
```

`web/app/uploads` holds both WordPress media library files and CiviCRM's `persist`, `custom`, and `upload` directories — the latter three are subdirectories created by CiviCRM on first run. Putting them all under a single mount keeps backups simple and avoids cross-mount permission surprises.

`web/app/wp-content/cache` is a separate mount because page cache plugins churn it constantly; isolating it from media uploads keeps backup snapshots small.

`/tmp` uses the `tmp` source rather than `storage`, so it doesn't count against your persistent disk and is safe to wipe between containers.

Note what's *not* mounted: `web/app/civicrm-ext/`, where Composer-managed extensions get installed. That directory is part of the build artifact, so extensions are versioned in `composer.json` and not editable through the CiviCRM UI in production. This is the right tradeoff for a PaaS — predictable deploys, no drift.

### Build hook

```yaml
build:
  flavor: composer
hooks:
  build: |
    set -e
    echo "Installing Upsun CLI"
    curl -fsSL https://raw.githubusercontent.com/platformsh/cli/main/installer.sh | VENDOR=upsun bash
```

The `composer` flavor handles `composer install` for you. The added CLI install gives you the Upsun CLI inside the container, which is occasionally useful for source operations and scheduled tasks that need to talk back to the Upsun API.

### Deploy hook

```yaml
deploy: |
  set -e
  sleep 45 || sleep 60 || sleep 90
  wp cache flush
  wp core update-db
```

The leading `sleep` looks weird, and it is. It exists because on a fresh deploy CiviCRM's WP-CLI integration occasionally races the readiness of the writable mount; giving the container 45 seconds to settle before flushing caches eliminates a class of intermittent first-deploy errors. The chained `|| sleep 60 || sleep 90` is belt-and-suspenders and effectively no-ops on success — feel free to simplify if it bothers you.

`wp cache flush` clears any stale object cache between releases. `wp core update-db` runs WordPress's own DB migrations on core upgrades — it's idempotent and cheap when there's nothing to do.

### Web locations: serve statically, block PHP everywhere it shouldn't run

This is the longest section of the config and the most security-critical. The high-level rule is: PHP only executes from places we explicitly trust. Everything in `uploads`, `cache`, and the various civi data dirs serves static assets but blocks `*.php` execution.

The root location handles WordPress requests:

```yaml
"/":
  root: "web"
  passthru: "/wp/index.php"
  index:
    - "index.php"
  expires: 300s
  scripts: true
  allow: true
  rules:
    '\.': { allow: false }
    ^/composer\.json:
      allow: false
    ^/license\.txt$:
      allow: false
    ^/readme\.html$:
      allow: false
```

Bedrock's Composer-managed layout lives at `web/wp/`, so `passthru` points there. The rules block hidden files, the Composer manifest, and the standard WordPress info pages that leak version data.

Uploads are explicitly hardened:

```yaml
"/wp/wp-content/uploads":
  root: "web/app/uploads"
  scripts: false
  allow: false
  rules:
    '\.(jpe?g|gif|png|svg|...|3g[p2])$':
      allow: true
      expires: 1w
    '\.(?i:php|phtml|php3|php4|php5|php7|php8|phar|pl|cgi|asp|aspx|jsp|cfm|shtml|shtm|htaccess|sh|bash|exe|dll)$':
      allow: false
    '/\.':
      allow: false
```

This pattern repeats for `/app/civicrm-ext`, `/app/uploads/civicrm/persist`, `/app/uploads/civicrm/custom`, and `/app/uploads/civicrm/upload`. Every path that accepts user-uploaded data has the same allowlist of safe extensions and an explicit denylist of every PHP-ish extension I've ever seen exploited. If you add a new mount that takes uploads, copy this block.

The one exception that legitimately needs PHP execution under uploads is KCFinder, the file browser bundled with CiviCRM:

```yaml
"/app/plugins/civicrm/civicrm/packages/kcfinder":
  root: "web/app/plugins/civicrm/civicrm/packages/kcfinder"
  allow: true
  scripts: true
  expires: 15m
  passthru: true
```

Locking that down further is on the roadmap but works as-is for now.

### Routes and cache cookies

```yaml
routes:
  "https://{default}/":
    type: upstream
    upstream: "wordpress:http"
    cache:
      enabled: true
      default_ttl: 300
      cookies:
        - '/^wordpress_logged_in_/'
        - '/^wordpress_sec_/'
        - 'wordpress_test_cookie'
        - '/^wp-settings-/'
        - '/^wp-postpass/'
        - '/^wp-resetpass-/'
        - '/^PHPSESSID$/'
        - '/^simple_wp_membership_sec_/'
```

Two things matter here. First, `upstream` *must* match the application name in the `applications:` block exactly — `wordpress:http` here. Mismatched names produce a deploy that succeeds but serves only 502s.

Second, the `cookies` allowlist is the difference between Upsun's edge cache being useful and being a foot-gun. By naming only the cookies that actually distinguish users, anonymous visitors get cached responses while logged-in users (and members, in the case of `simple_wp_membership_sec_`) bypass the cache cleanly. CiviCRM contribution and event pages are mostly anonymous traffic; this configuration lets them be served from cache, which matters during big campaigns.

### Crons

Two scheduled jobs cover both WordPress and CiviCRM:

```yaml
crons:
  wp:
    spec: '*/30 * * * *'
    commands:
      start: |
        wp cron event run --due-now
  civi-cron:
    spec: '*/30 * * * *'
    commands:
      start: |
        wp --user=cronuser civicrm api job.execute auth=0
```

The `wp` cron handles WordPress core scheduled events. The `civi-cron` job runs CiviCRM scheduled jobs through the WP-CLI integration, executing as a dedicated `cronuser` account that you'll create after the initial install (covered below).

For sites that send a lot of mail, you may want to split out CiviCRM mailing processing onto a faster cadence (e.g. every 5 minutes) by adding a third cron that runs `wp --user=cronuser civicrm api job.process_mailing auth=0` — the same separation we use on the Drupal side.

### Source operations: auto-update

```yaml
source:
  operations:
    auto-update:
      command: |
        curl -fsS https://raw.githubusercontent.com/platformsh/source-operations/main/setup.sh | { bash /dev/fd/3 sop-autoupdate; } 3<&0
```

This wires up Platform.sh / Upsun source operations so you can run `upsun source-operation:run auto-update` (or schedule it) and have Upsun open a PR / branch with `composer update` results applied. For CiviCRM sites, this is how you keep up with security releases without doing a manual `composer update` push every time.

## The Composer template

The `composer.json` is where the WordPress + CiviCRM marriage actually happens. The interesting parts:

### Repositories and packages

```json
"repositories": [
  { "type": "vcs", "name": "skvare/wordpress-bedrock-civicrm-settings",
    "url": "https://github.com/Skvare/wordpress-bedrock-civicrm-settings.git" },
  { "type": "vcs", "name": "civicrm/civicrm-extension-plugin",
    "url": "https://github.com/Skvare/civicrm-extension-plugin" },
  { "type": "vcs", "name": "civicrm/civicrm-wordpress",
    "url": "https://github.com/civicrm/civicrm-wordpress.git" },
  { "type": "composer", "url": "https://wpackagist.org",
    "only": ["wpackagist-plugin/*", "wpackagist-theme/*"] }
]
```

`wpackagist.org` provides Composer-flavored access to the WordPress.org plugin and theme directories. The CiviCRM repos point to specific upstream sources, including a Skvare fork of `civicrm-extension-plugin` (more on that below).

The `require` block pins CiviCRM at `6.13.*` and pulls in `civicrm-asset-plugin`, `civicrm-extension-plugin`, `civicrm-core`, `civicrm-packages`, and `civicrm-wordpress` together. It also pulls in `skvare/wordpress-bedrock-civicrm-settings` — a small package that ships a pre-configured `civicrm.settings.php` tailored to the Bedrock + Upsun layout. Doing this as a Composer package instead of an in-repo file makes upgrading the settings template across multiple sites a single bump.

### CiviCRM extensions via Composer

Out of the box, CiviCRM extensions are installed and updated through the CiviCRM admin UI — which on a read-only filesystem like Upsun is a non-starter. The `civicrm/civicrm-extension-plugin` (Skvare fork) lets you declare extensions in `composer.json` instead:

```json
"civicrm": {
  "cms_type": "wordpress",
  "extensions_install_path": "./web/app/civicrm-ext/contrib",
  "extensions": {
    "uk.co.vedaconsulting.mosaico": {
      "url": "https://download.civicrm.org/extension/uk.co.vedaconsulting.mosaico/3.12.1773937815/uk.co.vedaconsulting.mosaico-3.12.1773937815.zip"
    },
    "org.civicrm.cividiscount": {
      "url": "https://lab.civicrm.org/extensions/cividiscount/-/archive/3.10.2/cividiscount-3.10.2.zip"
    },
    "org.civicrm.contactlayout": {
      "url": "https://lab.civicrm.org/extensions/contactlayout/-/archive/3.0.4/contactlayout-3.0.4.zip"
    }
  }
}
```

Each extension is fetched from a URL (typically the CiviCRM extension directory or a GitLab archive), unpacked into `web/app/civicrm-ext/contrib`, and made available to CiviCRM. Pinning the URL means deploys are reproducible — the same Git SHA always produces the same extension set.

To add an extension, find its `.zip` URL on civicrm.org, add it to this list, run `composer update`, and push.

### Patches

```json
"patches": {
  "civicrm/civicrm-core": {
    "WIP composer support": "https://patch-diff.githubusercontent.com/raw/civicrm/civicrm-core/pull/32945.patch"
  },
  "civicrm/civicrm-wordpress": {
    "Composer and PaaS support": "https://patch-diff.githubusercontent.com/raw/civicrm/civicrm-wordpress/pull/362.patch"
  },
  "civicrm/civicrm-extension-plugin": {
    "WP location of settings file in config file": "./patches/civi-wp-composer-config.patch"
  }
}
```

Two upstream PRs and one local patch. The civi-core and civi-wordpress patches make necessary changes for discovering CiviCRM Core in composer's vendor directory, and relax assumptions about being able to write `civicrm.settings.php` at runtime — exactly the assumption a read-only PaaS filesystem violates. These will go away once the upstream PRs merge; in the meantime, `cweagans/composer-patches` applies them at install time.

### Post-install scripts

```json
"post-install-cmd": [
  "cp web/app/plugins/civicrm/wordpress-bedrock-civicrm-settings/civicrm.settings.php web/app/plugins/civicrm/",
  "composer civicrm:publish"
]
```

The first line copies the pre-configured `civicrm.settings.php` from the Skvare settings package into the location CiviCRM looks for it. The second line runs `civicrm:publish`, which copies CiviCRM's static assets into the location declared earlier (`web/app/plugins/civicrm/civicrm`). Both run during build, so the artifact that ships to Upsun has settings and assets already in place.

## The `.environment` shell file

Bedrock expects WordPress configuration to come from environment variables. On Upsun, you don't get to write a literal `.env` file in production — instead, `.environment` runs at container start and exports the same names from Upsun's built-in variables:

```bash
# Database environment variables — populated from PLATFORM_RELATIONSHIPS
export DB_HOST="$DATABASE_HOST"
export DB_PORT="$DATABASE_PORT"
export DB_PATH="$DATABASE_PATH"
export DB_DATABASE="$DB_PATH"
export DB_USERNAME="$DATABASE_USERNAME"
export DB_PASSWORD="$DATABASE_PASSWORD"
export DB_SCHEME="$DATABASE_SCHEME"
export DATABASE_URL="${DB_SCHEME}://${DB_USERNAME}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_PATH}"

# Routes, URLs, primary domain — derived from PLATFORM_ROUTES
export SITE_ROUTES="$(echo "$PLATFORM_ROUTES" | base64 --decode)"
export DOMAIN_CURRENT_SITE="$(
  echo "$SITE_ROUTES" | jq -r --arg app "$PLATFORM_APPLICATION_NAME" \
    'map_values(select(.primary == true and .type == "upstream" and .upstream == $app))
      | keys | .[0]
      | if (.[-1:] == "/") then (.[0:-1]) else . end'
)"

export WP_HOME="${DOMAIN_CURRENT_SITE}"
export WP_SITEURL="${WP_HOME}/wp"
export WP_DEBUG_LOG="/var/log/app.log"

# Salts derived from PLATFORM_PROJECT_ENTROPY — stable per project, never committed
export AUTH_KEY="${PLATFORM_PROJECT_ENTROPY}AUTH_KEY"
export SECURE_AUTH_KEY="${PLATFORM_PROJECT_ENTROPY}SECURE_AUTH_KEY"
# ...one line per Bedrock salt
```

The clever bit is the `PLATFORM_ROUTES` parse: rather than hard-coding `WP_HOME` per environment, we ask Upsun what the primary route for *this* application is and trim the trailing slash. Branch environments automatically get the right URL; production gets the production URL. Bedrock reads `WP_HOME` and `WP_SITEURL` from these exports through `phpdotenv`-style loading.

`PLATFORM_PROJECT_ENTROPY` is per-project secret entropy that Upsun guarantees is stable and unique. Concatenating it with each salt name gives you eight different, deterministic, project-scoped values — no need to commit secrets, no need to rotate them on every deploy.

## First-time WordPress + CiviCRM install

After `git push upsun main` succeeds, follow this order:

### 1. Run the WordPress installer

Open the project URL. You'll see the standard WordPress 5-minute install. Set the site title, admin user, and admin password. The DB connection is already wired up via `.environment`, so you won't see a database step.

### 2. Activate the CiviCRM plugin

Go to **Plugins** in the WordPress admin. CiviCRM is already installed as a Composer plugin under `web/app/plugins/civicrm/`. Click **Activate**.

The plugin will detect the pre-shipped `civicrm.settings.php` and the configured `civicrm` database endpoint, run its install routine, and finish without prompting for DB credentials.

Visit `/wp-admin/admin.php?page=CiviCRM` to confirm CiviCRM loaded correctly.

### 3. Create the cron user

CiviCRM's API runs as a WordPress user. The `civi-cron` job runs `wp --user=cronuser ...`, so create a user with that exact username:

- **Users → Add New**
- Username: `cronuser`
- Role: Administrator
- Set a strong password (you'll never log in as this user — the cron runs as it locally)

Once that user exists, the cron job at `*/30 * * * *` starts succeeding silently. Tail the cron log via `upsun log cron` if you want to confirm.

### 4. Activate any extensions you declared

Extensions you listed in `composer.json` are *installed* (their files are on disk) but not *enabled*. Go to **CiviCRM → Administer → Extensions** and click Enable on each one. This is the only ongoing UI step CiviCRM extensions require — once enabled, the state lives in the database, so subsequent deploys keep them on.

## Things to know once you're running

**The CiviCRM admin UI will warn about extension installs being unavailable.** That's expected — you're managing extensions through Composer, not the UI. Document this in your runbook so future-you (or a colleague) doesn't think it's a bug.

**Mailings.** If you do significant outbound email, add a dedicated mailing cron at `*/5 * * * *` running `wp --user=cronuser civicrm api job.process_mailing auth=0` and let the half-hourly cron handle everything else. Don't try to send mailings synchronously from web requests.

**Caching.** CiviCRM defaults to a database-backed cache. For larger sites, add a Redis service to the project, expose it as a relationship, and set `CIVICRM_CACHING_TYPE=Redis` as an Upsun variable. The PHP `redis` extension is already in the runtime list.

**Backups.** Upsun's automatic snapshots cover both the database and persistent mounts, so the `uploads` mount and CiviCRM's `persist` directory are included in restores. You don't need a separate backup strategy unless you want offsite copies — in which case `duplicity` or `restic` driven from a manual cron writing to S3-compatible storage works well.

**Branch environments.** Because `WP_HOME` and `WP_SITEURL` are derived from `PLATFORM_ROUTES`, every branch environment gets the right URL automatically. CiviCRM's `CIVICRM_DEPLOY_ID` is a single env var across environments, though — if you want per-environment deploy IDs to invalidate asset caches independently, set the variable per environment in the Upsun console rather than at the project level.

## Wrapping up

The combination of Bedrock's Composer-first layout and Upsun's PaaS primitives gives CiviCRM a much better home than traditional shared hosting: predictable, reproducible deploys; sensible filesystem boundaries between code and data; and a database service that scales without you having to think about it.

The template is open source and MIT-licensed. If you find a rough edge or want to add a configuration option (Redis, multisite, a particular extension), pull requests are welcome at [github.com/Skvare/upsun-wordpress-bedrock-civicrm-template](https://github.com/Skvare/upsun-wordpress-bedrock-civicrm-template).

If your organization needs help architecting a CiviCRM install — WordPress, Drupal, or Joomla — [Skvare](https://skvare.com/contact) does this professionally. Thirteen years and counting of CiviCRM in production.

---

*Mark Hanna is CTO of [Skvare](https://skvare.com), a Drupal and CiviCRM consultancy. He's the author of the Drupal 11 + CiviCRM on Upsun guide and maintains several CiviCRM-adjacent open source modules including CiviCRM Entity, CiviCRM Form Builder Blocks, CiviCRM Drush, and CiviCRM Reroute Mail.*

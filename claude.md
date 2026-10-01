
# CLAUDE.md

Context and rules for working on **WATHI Gala Billetterie**, a WordPress plugin. Read this file and `README.md` before making changes. `README.md` is the functional specification (in French); this file explains how to build it.

## Project overview

WATHI (wathi.org) is a West African citizen think tank based in Dakar, Senegal. This plugin runs on the production WordPress site and handles the annual WATHI Gala:

- A presentation page for the gala
- Ticket sales through **Wave** mobile money payment links (Senegal, amounts in FCFA / XOF)
- A **stock of pre-made ticket images** supplied by WATHI's graphic designer, one image per ticket, each with a unique visible number
- Automatic assignment of one available ticket image per validated sale, sent by e-mail as an attachment
- Each sent ticket is marked as sent and permanently removed from the available stock
- A protected front-end **team page** where a colleague validates Wave payments and sends tickets
- **Excel export** (.xlsx) of successful sales, delivered tickets, stock and summary

The plugin does **not** generate ticket designs, PDFs or QR codes. It only stores, assigns and sends the designer's files.

## Hard constraints

**Server versions (cannot be upgraded for now):**
- WordPress **6.8.10**
- PHP **7.4.33**

All PHP code must run on PHP 7.4 and stay compatible with PHP 8.x. Never use:
- `match` expressions
- Named arguments
- Union types, `mixed`, `static` return type, `never`
- Nullsafe operator `?->`
- Constructor property promotion
- `str_contains()`, `str_starts_with()`, `str_ends_with()`, `array_is_list()`
- Enums, `readonly`, first-class callable syntax, attributes `#[...]`
- `throw` as an expression

Typed properties, arrow functions (`fn`), `??=` and spread in arrays are allowed (PHP 7.4).

**Dependencies:**
- Only `phpoffice/phpspreadsheet` pinned to `^1.29` (2.x requires PHP 8.1)
- `composer.json` must keep `"config": { "platform": { "php": "7.4.33" } }`
- Do not add dependencies without asking. Prefer WordPress core APIs.

**Language:**
- Code, identifiers, comments and commit messages: English
- Every user-facing string (front-end, admin, e-mails, Excel headers, error messages): **French**, wrapped in translation functions with the text domain `wathi-gala-billetterie`
- Database status values are lowercase French slugs without accents (see below)

**Style of produced documents and UI:** sober, clean, premium. No em dashes (—) in user-facing text or docs. No decorative horizontal rules under headings.

## Naming conventions

| Item | Value |
|---|---|
| Plugin slug / folder | `wathi-gala-billetterie` |
| Main file | `wathi-gala-billetterie.php` |
| Text domain | `wathi-gala-billetterie` |
| PHP namespace | `Wathi\GalaBilletterie` |
| Function / hook / option prefix | `wgb_` |
| Constants prefix | `WGB_` |
| Table prefix | `{$wpdb->prefix}wathi_gala_` |
| REST namespace | `wathi-gala/v1` |
| Custom role | `wgb_gestionnaire` (label "Gestionnaire billetterie") |
| Private storage folder | `wp-content/uploads/wathi-gala-prive/` (or outside web root if configured) |
| Sale reference format | `GALA{YY}-{NNNN}`, e.g. `GALA26-0042` |
| Ticket number format | `{PREFIX}-{NNN}`, e.g. `VIP-014`, taken from the designer's file name |

Follow the WordPress Coding Standards (WPCS). One class per file in `includes/`, file named `class-{name}.php`.

## Architecture

```
wathi-gala-billetterie.php     Bootstrap, constants, autoload, hooks registration
includes/
  class-activator.php          Tables (dbDelta), role and capabilities, private folder + .htaccess
  class-settings.php           Settings pages (Settings API)
  class-sales.php              Sale creation, status transitions, expiry (cron)
  class-ticket-stock.php       Import, validation, atomic assignment, invalidation
  class-private-storage.php    Writing, reading and streaming private files
  class-wave.php               Wave links (manual mode) and Checkout API (automatic mode)
  class-webhook.php            REST endpoint for Wave notifications
  class-mailer.php             E-mails via wp_mail() with attachments
  class-export.php             XLSX generation with PhpSpreadsheet
  class-team-page.php          Front-end team page logic and AJAX/POST handlers
  class-logger.php             Append-only action journal
  class-shortcodes.php
templates/                     Overridable from theme in wathi-gala/
uninstall.php
```

Shortcodes: `[wathi_gala_presentation]`, `[wathi_gala_billets]`, `[wathi_gala_confirmation]`, `[wathi_gala_envoi]`.

## Domain model

**Sales** table `wathi_gala_ventes`, statuses:

| Slug | Meaning |
|---|---|
| `en_attente` | Form submitted, payment not confirmed |
| `payee` | Payment confirmed, ticket not sent yet |
| `livree` | Ticket(s) assigned and e-mail accepted by the mail server |
| `echec_envoi` | Ticket(s) assigned, e-mail failed |
| `expiree` | No payment within the configured delay (default 48 h) |
| `annulee` | Cancelled by the team (reason required) |

A "successful" sale is `payee`, `livree` or `echec_envoi`.

**Tickets** table `wathi_gala_billets`, statuses:

| Slug | Meaning |
|---|---|
| `disponible` | In stock, can be assigned |
| `reserve` | Assigned to a sale, sending in progress or failed |
| `envoye` | Sent, permanently out of stock |
| `invalide` | Sale cancelled after sending, must be refused at the entrance |
| `retire` | Faulty file removed before any sending |

**Journal** table `wathi_gala_journal`: append-only (sale or ticket id, user id, action, IP, date, details JSON).

Full column lists are in `README.md` section 15. Use `dbDelta()` and store a schema version in option `wgb_db_version` for migrations.

## Ticket stock invariants (never break these)

1. A ticket is assigned to **at most one sale, ever**. `vente_id` is set once and never changed to another sale.
2. Allowed transitions only:
   - `disponible` → `reserve` (assignment) or `retire` (admin removal with reason)
   - `reserve` → `envoye` (mail accepted)
   - `envoye` → `invalide` (sale cancelled after sending)
   - `reserve` → `invalide` (sale cancelled before successful sending)
   - Nothing ever goes back to `disponible`.
3. A failed e-mail keeps the ticket in `reserve` on the same sale. Resending reuses the same ticket and increments `nombre_envois`. It never consumes a new ticket.
4. Sent tickets and their rows cannot be deleted through the plugin.
5. A Wave transaction ID can be used only once (unique index).
6. Ticket `numero` and file `empreinte` (SHA-256) are unique.
7. Available places shown to buyers = count of `disponible` tickets for the category, minus pending reservations if implemented. Never a hard-coded quota.

**Atomic assignment.** Two colleagues may validate at the same time. Never do "SELECT an available ticket then UPDATE it". Use a single conditional UPDATE and check affected rows:

```php
$token = wp_generate_uuid4();
$updated = $wpdb->query( $wpdb->prepare(
    "UPDATE {$table} SET statut = 'reserve', vente_id = %d, jeton = %s, date_attribution = %s
     WHERE statut = 'disponible' AND categorie = %s
     ORDER BY id ASC LIMIT 1",
    $sale_id, $token, current_time( 'mysql', true ), $category
) );
if ( 1 !== $updated ) {
    // Out of stock: stop, do not mark the sale as delivered.
}
$ticket = $wpdb->get_row( $wpdb->prepare(
    "SELECT * FROM {$table} WHERE jeton = %s", $token
) );
```

For multi-ticket orders, repeat per ticket inside a transaction (`START TRANSACTION` / `COMMIT` / `ROLLBACK`, tables in InnoDB) so an order is either fully assigned or not at all.

Before sending, recompute the file's SHA-256 and compare with `empreinte`. Refuse to send on mismatch and log it.

## Wave payments

- **Manual mode (default and fallback):** each category has a Wave payment link set in settings. Plain Wave links do not notify the site. The buyer is shown the sale reference and asked to put it in the payment note. A colleague checks Wave Business, then enters the transaction ID and amount on the team page.
- **Automatic mode (optional):** Wave Business Checkout API with a webhook at `/wp-json/wathi-gala/v1/wave-webhook`. Always verify the webhook signature before touching a sale. Make the handler idempotent (same event received twice must not send twice).
- **Do not invent Wave API endpoints, payload fields or signature schemes.** Check the official Wave Business API documentation or ask the user before implementing automatic mode.
- Amounts are integers in FCFA (XOF has no decimals). Warn when the entered amount differs from the expected amount.

## Security rules

This plugin handles payments and personal data on a PHP 7.4 server that no longer gets security patches. Treat security as a requirement, not an extra.

- **Capabilities:** check `current_user_can()` on every action. Custom caps: `wgb_view_sales`, `wgb_send_tickets`, `wgb_cancel_sales`, `wgb_export`, `wgb_manage_stock`, `wgb_manage_settings`, `wgb_view_journal`. Administrators get all; `wgb_gestionnaire` gets view, send, cancel, export only.
- **Nonces:** every form and AJAX/REST action (except the signed Wave webhook) uses a nonce.
- **Input:** sanitize on input (`sanitize_text_field`, `sanitize_email`, `absint`...), validate server-side, escape on output (`esc_html`, `esc_attr`, `esc_url`, `wp_kses_post`).
- **SQL:** always `$wpdb->prepare()`. No string concatenation of user input.
- **Uploads:**
  - Check the real MIME type with `finfo` / `wp_check_filetype_and_ext()`, not the extension
  - Allow only `image/png`, `image/jpeg`, `application/pdf`
  - Max 5 MB per file
  - Extract ZIP archives to a temp folder, validate each entry, reject paths with `..` or absolute paths (zip slip), reject nested archives
  - Rename stored files with random names (`bin2hex( random_bytes( 16 ) )` + extension)
- **Private storage:** ticket files never go to the Media Library and never get a public URL. The folder has `.htaccess` with `Require all denied` plus an empty `index.php`. For Nginx, document the `deny all` rule (see README section 13). Admin preview streams the file through an authenticated handler with `nocache_headers()`.
- **Team members never download available stock files.** They only see counts and the ticket number assigned to a sale.
- **Excel exports** are streamed directly to the browser (`php://output`), never written to a public folder. Neutralize spreadsheet formula injection: prefix cell values starting with `=`, `+`, `-`, `@` with a single quote.
- **Rate limiting** on the public purchase form (honeypot field + transient-based limit per IP).
- **Caching:** send `nocache_headers()` on plugin pages and document excluding `/gala/envoi-billets`, `/gala/confirmation` and `/wp-json/wathi-gala/` from page cache and Cloudflare cache.
- **Journal** every sensitive action: import, assignment, sending, resending, e-mail correction, cancellation, removal, export, settings change.
- Never log or display full personal data in error messages. Never put personal data in URLs.
- Recommend 2FA for team accounts (handled by a separate plugin, not by this one).
- Data retention: provide a purge function after the gala (Senegal personal data law n° 2008-12).

## E-mails

- Send with `wp_mail()`. The site is expected to use an SMTP plugin; do not implement SMTP in this plugin.
- Attach the ticket file(s) by absolute path from private storage.
- Mark the ticket `envoye` and the sale `livree` **only if `wp_mail()` returns true**. Otherwise set the sale to `echec_envoi`, keep the ticket `reserve`, log it and notify the team alert address.
- HTML e-mail templates in `templates/emails/`, in French, simple and clean.

## Environment

- Production: wathi.org, WordPress behind **Cloudflare**.
- The user (WATHI tech and communications) maintains the site. Assume shared-hosting style constraints: no root access, no long-running processes. Use WP-Cron for expiry and scheduled exports.
- Timezone: Africa/Dakar (UTC+0). Store dates in UTC, display in the site timezone.

## Local development

Use a local environment matching production versions:

```bash
# wp-env (Docker) with matching versions, in .wp-env.json:
# { "core": "WordPress/WordPress#6.8.10", "phpVersion": "7.4", "plugins": [ "." ] }
npx @wordpress/env start

composer install
composer exec phpcs     # WPCS + PHPCompatibilityWP
```

Code quality setup (dev dependencies):
- `wp-coding-standards/wpcs`
- `phpcompatibility/phpcompatibility-wp` with `testVersion` set to `7.4-` in `phpcs.xml.dist`

Run PHPCS before considering any task done. A PHP 8-only construct is a bug.

## Testing checklist for any change touching sales or stock

- Two simultaneous validations of different sales in the same category get different tickets
- Validation with empty stock is blocked and nothing is marked as delivered
- Resend uses the same ticket and does not reduce the stock
- Cancelling a delivered sale sets its tickets to `invalide`, stock unchanged
- Reusing a Wave transaction ID is refused
- Uploading a renamed `.php` file as `.png` is refused
- Ticket files are not reachable by direct URL
- A `gestionnaire` user cannot import, remove stock, change settings or read the journal
- Export opens correctly in Excel and LibreOffice, French accents intact

## Working with the user

- The user is technical (software engineering degree, maintains WATHI's WordPress and Cloudflare). Be direct and concrete.
- Ask before: adding a dependency, changing the database schema after it ships, implementing the Wave API, or anything that cannot be undone on production.
- Keep `README.md` in sync when behavior changes.

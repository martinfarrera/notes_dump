# Notes Dump

Notes Dump is a dependency-free note workspace with password-protected synchronization between devices. The browser always saves a local copy first; an authenticated PHP API then synchronizes the workspace with a single MySQL row.

## What it does

- Editable, resizable note tables in a responsive masonry layout, with independent colors and typography.
- Keyboard and drag-and-drop ordering for tables and activities.
- Automatic removal of completed activities after three hours.
- Immediate local saves plus debounced server synchronization.
- A single password-protected workspace with no registration or public accounts.
- An absolute three-day authenticated session on each device.
- Optimistic concurrency: a stale device cannot silently overwrite a newer server revision.

## Deploy to Hostinger

This project requires a Hostinger **Custom PHP/HTML** website, PHP with PDO MySQL, HTTPS, and one MySQL database. It does not work on Hostinger Website Builder.

### 1. Create the database

1. In hPanel, open **Websites → Dashboard → Databases → Management**.
2. Create a MySQL database and user. Record the full database name, username, and password; Hostinger prefixes the first two with the account identifier.
3. Open phpMyAdmin for that database, choose **Import**, and import [`schema.sql`](schema.sql).

Hostinger reference: [Create a MySQL database](https://www.hostinger.com/support/1583542-how-to-create-a-new-mysql-database-in-hostinger/).

### 2. Generate the login password hash

Do not put the plain-text application password in any file. On a computer with PHP, run:

```bash
read -s PASSWORD
export PASSWORD
php -r 'echo password_hash(getenv("PASSWORD"), PASSWORD_DEFAULT), PHP_EOL;'
unset PASSWORD
```

The first command waits silently for the password and may show no prompt. Copy the resulting `$2y$...` hash.

If local PHP is unavailable, create a temporary PHP CLI script in Hostinger's terminal that calls `password_hash`, run it once, and delete it immediately. Do not use an online hash generator.

### 3. Create the private configuration

Copy `config.example.php` to `config.php`, then replace every placeholder:

```php
<?php

declare(strict_types=1);

return [
    'database' => [
        'dsn' => 'mysql:host=localhost;dbname=FULL_DATABASE_NAME;charset=utf8mb4',
        'username' => 'FULL_DATABASE_USERNAME',
        'password' => 'DATABASE_PASSWORD',
    ],
    'password_hash' => 'PASTE_THE_PASSWORD_HASH_HERE',
    'session_name' => 'notes_dump_session',
    'session_lifetime_seconds' => 259200,
    'cookie_secure' => true,
    'max_workspace_bytes' => 1048576,
];
```

`config.php` is ignored by Git and excluded from the deployment ZIP. `session_lifetime_seconds` controls the absolute authenticated-session window; `259200` is three days. Keep `cookie_secure` set to `true` in production and ensure HTTPS is active before signing in.

### 4. Upload the application

1. Back up the current site before replacing files.
2. Upload and extract `notes-dump-hostinger.zip` in the domain's `public_html` folder.
3. Upload the completed `config.php` separately to `public_html/config.php`.
4. Confirm these runtime paths exist:

```text
public_html/
├── api/
│   ├── auth.php
│   ├── bootstrap.php
│   └── workspace.php
├── app.js
├── config.php
├── favicon.svg
├── index.html
├── logic.js
└── styles.css
```

Hostinger reference: [Upload a custom PHP/HTML website to `public_html`](https://www.hostinger.com/support/1583289-how-to-manually-transfer-a-website-to-hostinger/).

### 5. Verify migration and synchronization

1. Open the deployed site on the Mac that already contains the notes.
2. Open Configuration with the gear icon (`⚙`), then sign in from the **Synchronization** section. Because the server is empty and the Mac has local tables, Notes Dump uploads the Mac workspace as revision 1.
3. Wait until the control reports **Sincronizado**.
4. Open the same HTTPS address on the phone, sign in, and confirm the server workspace loads.
5. Edit one activity on the phone, wait for synchronization, focus the Mac window, and confirm the change appears.
6. Open the gear icon (`⚙`), switch between light and dark mode, and use **Recargar** to verify that local data survives a page reload.

Do this first upload from the Mac. An empty phone does not initialize or overwrite an empty server, but using the Mac first makes the intended migration explicit and easy to verify.

## Data and conflict semantics

| Situation | Result |
| --- | --- |
| Browser edit | Saved immediately to `localStorage`, then sent to the server after a short debounce. |
| Empty server + nonempty local workspace | Local workspace initializes the server. |
| Existing server + empty new device | Server workspace replaces the empty local workspace. |
| Only one side changed since the last synchronized snapshot | The changed side wins automatically. |
| Both sides changed | Synchronization stops and asks which copy to keep. |
| Network or server error | Local data remains intact and the gear icon reports the synchronization problem. |
| Stale PUT request | API returns HTTP `409` with the current server revision; the client does not overwrite it. |

The server stores one workspace in `notes_dump_workspace`. Every accepted write increments `revision`. A conflict choice replaces the whole workspace; it does not merge individual tables.

Browser data remains local to each browser profile. Clearing browser storage removes that device's offline copy and synchronization metadata, but does not delete the server workspace. The application currently has no server-side delete endpoint or account recovery flow.

## Security model

- The login password is verified with PHP `password_verify`; only its hash is stored in `config.php`.
- Authentication uses a server-side session with `HttpOnly`, `Secure`, and `SameSite=Strict` cookies. The cookie, PHP garbage-collection window, and an explicit server-side expiration timestamp enforce an absolute three-day lifetime by default.
- State-changing requests require a per-session CSRF token and reject cross-site origins.
- Database access uses PDO native prepared statements.
- The API validates the JSON structure, caps a workspace at 1 MiB by default, and limits table/activity counts and string sizes.
- API responses disable caching and never return database credentials or the password hash.

This is intentionally a single-user system. Anyone who knows the application password can read and replace the complete workspace. Use a unique password and protect the Hostinger account with two-factor authentication.

## Run and verify locally

The UI remains usable with local-only persistence from a static server, but remote synchronization requires PHP, MySQL, `config.php`, and the schema.

Run the dependency-free JavaScript tests and syntax checks:

```bash
node --test tests/*.test.js
node --check logic.js
node --check app.js
```

If PHP CLI is installed, lint the API:

```bash
php -l api/bootstrap.php
php -l api/auth.php
php -l api/workspace.php
php -l config.example.php
```

## Deployment archive contents

`notes-dump-hostinger.zip` contains only the web runtime plus `config.example.php`. It intentionally excludes `config.php`, tests, development metadata, `schema.sql`, and this README. Import `schema.sql` through phpMyAdmin before uploading the archive.

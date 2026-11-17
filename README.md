# Laravel unsafe-upload lab (local only)

**Current state: VULNERABLE.** Phase 1 is live; Phase 2 (the fix) is next. Do
not expose this lab to any network — the upload endpoint intentionally permits
arbitrary file writes and PHP execution from `public/uploads/`.

## Run

```bash
cd unsafe-upload-lab
./start.sh                  # http://127.0.0.1:8085/upload
# in another terminal:
python3 exploit.py
./stop.sh                   # when finished
```

Bundled PHP 8.3 and PHP's built-in server are used directly with the Laravel
framework router. One process bound to `127.0.0.1:8085`, PID in
`unsafe-upload-lab/.server.pid`. No Docker, no containers, no services, no
restart policy. `start.sh` prints the URL and the exploit command.

Prerequisites: Python 3 (standard library only) for `exploit.py`; nothing else
needs installing because `app/vendor/` and `runtime/` are bundled.

## Layout

| Path | Purpose |
|---|---|
| `app/app/Http/Controllers/UploadLabController.php` | Vulnerable upload handler |
| `app/resources/views/upload-lab.blade.php` | Upload form (includes `@csrf`) |
| `app/routes/web.php` | `GET`/`POST /upload` route |
| `app/public/uploads/` | Writable, web-executable destination (deliberate) |
| `flag.txt` | Secret **outside** `public/`, copied by the webshell command |
| `exploit.py` | Multipart PHP webshell upload + command execution |
| `ATTACK.md` | Phase 1 walkthrough and Phase 2 fix preview |
| `start.sh` / `stop.sh` | Lab controls (PID file, no restart) |

## AGENTS.md note

The copied Laravel app's `AGENTS.md` conventions were followed: the controller
and view were generated with `php artisan make:controller` and
`php artisan make:view`; relevant `.agents/skills/laravel-best-practices` rules
(security, validation, routing, blade views, style) were read; Artisan
`route:list` was used; and Pint was run after PHP edits. There is no
`.ai/rules` directory in this app.

Allowing unvalidated uploads with client-supplied filenames into a
web-executable directory is a **deliberate, documented deviation** from those
security rules — it is the lab target. `@csrf` is kept, and `exploit.py` fetches
a legitimate token first, demonstrating that CSRF tokens do not prevent a user
from uploading whatever they choose.

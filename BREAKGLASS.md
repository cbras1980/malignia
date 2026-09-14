# Break-glass SSH access

A recovery path for when you've lost every normal way to authenticate (SSH
keys, password safe, etc.): a dedicated, low-privilege `breakglass` account
that is the *only* account allowed to log in over SSH with a password
(optionally plus a TOTP code), reachable either with a normal SSH client or
through Wetty, a browser-based terminal that just runs `ssh` server-side and
shows it in a web page.

The real authentication happens in sshd/PAM via a `Match User breakglass`
block - the browser page is not custom auth code, it's an SSH client with a
web UI in front of it. This was written and reviewed but **not executed
against a live server** (this session only has git access to this repo, not
a shell on your actual host) - read every step below before running
anything, and rehearse the whole flow once now, while you still have normal
access, rather than waiting until you actually need it.

## What gets set up

| Playbook | What it does |
|---|---|
| `set_breakglass_user.yml` | Creates the `breakglass` OS account with the password hash you supply. |
| `set_breakglass_sshd.yml` | Adds a `Match User breakglass` drop-in that allows password (and, optionally, TOTP) auth *only* for that account; every other account keeps your existing key-only policy. Validates with `sshd -t` before reloading. |
| `set_breakglass_pam_totp.yml` | Optional. Requires a TOTP code in addition to the password, for `breakglass` only. |
| `set_breakglass_sudoers.yml` | Grants `breakglass` sudo access - either scoped to a small recovery script (default) or full NOPASSWD sudo (`breakglass_sudo_mode: full`). |
| `set_breakglass_wetty.yml` | Installs Wetty behind a dedicated `wetty` system user, bound to `127.0.0.1` only. |
| `set_breakglass_alert.yml` | Optional. Sends a Telegram message (via the same `telegram-send` setup as `myexperiments/elite_alert.py`) every time `breakglass` logs in. |
| `set_breakglass_nginx.yml` | Optional, off by default. TLS-terminating reverse proxy in front of Wetty. Assumes the Debian/Ubuntu `sites-available`/`sites-enabled` nginx layout. |
| `set_breakglass_all.yml` | Runs everything above except nginx, in order. |

## 1. Choose your settings

Edit `vars/breakglass.yml` for anything non-secret (sudo mode, TOTP on/off,
Telegram alert on/off, ports, the account whose keys get restored). **Never**
put the password hash, TOTP secret, or real public keys in that file - it's
committed to git.

## 2. Generate a strong passphrase and its hash

Since this password is the sole recovery factor if you skip TOTP, make it
long and memorable rather than short and "complex" - e.g. 6+ random words
(diceware-style), which you actually memorize rather than write down.

```bash
openssl passwd -6 'your long memorized passphrase'
```

## 3. Run the core setup

```bash
ansible-playbook -i inventory.txt set_breakglass_all.yml \
  -e breakglass_password_hash='$6$...' \
  -e breakglass_recover_authorized_keys='["ssh-ed25519 AAAA... you@host"]'
```

If you turned on `breakglass_require_totp`, the PAM playbook will fail on
the first run with instructions to generate a secret
(`sudo -u breakglass google-authenticator`) - **print the resulting QR code
or secret and store it somewhere physical, separate from your password
safe**. That's the entire point of a break-glass factor: it must survive
losing the password safe. Then re-run the command above.

## 4. Test before you trust it

Keep your current SSH session open. In a **second** terminal:

```bash
ssh breakglass@your-server -o PreferredAuthentications=password -o PubkeyAuthentication=no
```

Confirm you're prompted for the password (and TOTP code, if enabled), and
that you land in a shell. Only once this works should you consider closing
your normal session in a real emergency.

If anything is wrong, fix it from your still-open normal session - that's
why you keep it open during testing.

## 5. (Optional) Put Wetty behind HTTPS

Wetty is bound to `127.0.0.1:{{ wetty_port }}` and not reachable from the
network by itself. To reach it from a browser on any computer, put a
TLS-terminating reverse proxy in front of it - `set_breakglass_nginx.yml` is
provided for that if you're already running nginx with the
`sites-available`/`sites-enabled` layout, once you have a domain and a
certificate (e.g. from certbot):

```bash
ansible-playbook -i inventory.txt set_breakglass_nginx.yml \
  -e breakglass_manage_nginx=true \
  -e breakglass_domain=recover.yourdomain.tld \
  -e breakglass_tls_cert=/etc/letsencrypt/live/recover.yourdomain.tld/fullchain.pem \
  -e breakglass_tls_key=/etc/letsencrypt/live/recover.yourdomain.tld/privkey.pem
```

If you use a different proxy or nginx layout, use this template as a
reference and set it up by hand instead.

The Wetty CLI flags in `templates/wetty.service.j2` haven't been verified
against a live install - after running `set_breakglass_wetty.yml`, check
`wetty --help` for the installed version and confirm the service actually
starts and connects (`systemctl status wetty`, then load the page) before
relying on it.

## 6. Rotate the password periodically

```bash
ansible-playbook -i inventory.txt set_breakglass_user.yml \
  -e breakglass_password_hash="$(openssl passwd -6 'new passphrase')"
```

## If you get locked out anyway

Use your hosting provider's out-of-band console (serial/VNC console access -
most VPS and dedicated server providers offer this independently of SSH) to
fix `sshd_config`, `sudoers`, or the account directly.

## Threat model notes

- Only `breakglass` accepts password auth; every other account is unaffected
  and stays key-only.
- `MaxAuthTries 3` and `LoginGraceTime 20` in the sshd drop-in limit brute-force
  attempts per connection; make sure fail2ban (or an equivalent) is watching
  sshd's auth log to block repeat offenders by IP - this repo doesn't set
  that up, verify it separately.
- Default sudo scope is `scoped`: `breakglass` can only run
  `breakglass-recover.sh` (restore your main key, reload sshd, check status)
  as root, not arbitrary commands. Widen to `full` only if you hit a real
  recovery scenario the script doesn't cover.
- Deleting this setup later: disable/remove the `breakglass` user, delete
  `/etc/ssh/sshd_config.d/99-breakglass.conf`, `/etc/sudoers.d/breakglass`,
  and the `wetty` systemd service, then reload sshd.

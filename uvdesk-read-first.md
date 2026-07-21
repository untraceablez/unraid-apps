# UVdesk on Unraid — Read First

This image is a straight repackage of the official
[`uvdesk/community-skeleton`](https://github.com/uvdesk/community-skeleton)
release, built unmodified. Because upstream bakes the application into the image
and expects you to finish setup through a **web installer**, there are a couple of
Unraid-specific things to know before you start. Read this once and you'll avoid
the two problems everyone hits.

---

## 1. The container must be given a foreground command

The upstream image starts Apache and MySQL as **background services** and then
runs `/bin/bash` in the foreground. Run detached (as Unraid does), that shell has
no TTY, hits end-of-input immediately, exits — and the container stops a second
after it starts.

This is by upstream design (the image is meant to be run interactively or
`exec`'d into), and we do **not** modify upstream. Instead we give the container
something that stays in the foreground. The template already does this via **Post
Arguments**:

```
tail -f /dev/null
```

Apache and MySQL keep serving; `tail` holds the container open. If you ever build
your own `docker run`, the equivalent is appending `tail -f /dev/null` after the
image name (or running it with `-dit`).

> If your container starts and immediately stops, this is almost always the
> reason — confirm Post Arguments is set.

---

## 2. Pick a database mode — external or built-in

The entrypoint only touches the **bundled** MySQL, and only when
`MYSQL_DATABASE` **and** `MYSQL_USER` **and** `MYSQL_PASSWORD` are all set. That
one rule decides your mode:

### External DB (recommended)

Point UVdesk at a separate MySQL/MariaDB (its own Unraid container, or any server
on your network).

- **Leave all *Built-in DB* fields blank** — especially `MYSQL_DATABASE`.
- On start you'll see:
  `Notice: Skipping configuration of local database - one or more mysql environment variables are not defined.`
  **This is expected and correct** on the external path — it is a yellow notice,
  not an error.
- The *External DB Host* / *External DB Port* template fields are for **your
  reference only** — the container does not read them. You enter the real host,
  port, database, user, and password in the **web installer** (see below).
- On your external server, create the database and a user with rights to it
  *before* running the installer.

### Built-in DB (quick tests only)

Let the container run its own MySQL.

- Set *Built-in DB: Database*, *User*, *Password*, and *Root Password*.
- Add the `/var/lib/mysql` volume.
- In the web installer, use host **`127.0.0.1`**, port `3306`, and the same
  credentials.
- **Caveat:** the image initializes `/var/lib/mysql` at build time. Bind-mounting
  an *empty* appdata folder over it hands mysqld an uninitialized data directory
  and it won't start. For anything you care about, use the external DB path.

---

## 3. Finish setup in the web installer

> ### ⚠️ Run the installer in an Incognito / private window
>
> **Do the whole web wizard in an Incognito/private window (or a browser profile
> with extensions disabled).** A browser **password manager** (1Password was the
> confirmed culprit here, but others behave the same) hooks the installer's form
> fields and interferes with the **Website configuration** step: the *Member /
> Customer Panel Prefix* fields don't register properly, the **Proceed** button
> greys out the instant you touch the Customer field, and you can't advance — even
> after typing valid values. Disabling the manager *for the site* is **not enough**
> (its content script is still injected); a private window with extensions off is
> the reliable fix. In Incognito the wizard completes normally.
>
> If you already have a half-broken install from this, see section 6.

After the container is up, browse to `http://<UNRAID-IP>:<PORT>` and complete the
UVdesk web wizard. The Database step is where you type your real connection
details:

- **External DB:** your server's host/IP, port, database, user, password.
- **Built-in DB:** host `127.0.0.1`, port `3306`, and the `MYSQL_*` values you set.

The wizard also asks for an admin account, and (in some releases) app
secret / environment / currency. Those are installer inputs, **not** container
environment variables — you won't find template fields for them, and that's
correct.

---

## 4. Permissions & user IDs (Unraid 99:100 vs the image's 1000)

Two things bite everyone here:

- **`PUID`/`PGID` do nothing.** This is upstream UVdesk's own image, not a
  LinuxServer.io image — there's no s6/`usermod` layer, so `PUID`/`PGID`
  environment variables are silently ignored. Don't add them.
- **The app runs as UID/GID 1000.** The Dockerfile creates the `uvdesk` user with
  a plain `adduser`, fixing it at **1000:1000**. Unraid, by contrast, owns appdata
  as **`nobody:users` = 99:100**. That mismatch means any host folder you mount
  into the container is *unwritable* by the app until you fix its ownership.

So **every volume you bind-mount must be chowned to 1000** on the Unraid host:

```sh
chown -R 1000:1000 /mnt/user/appdata/uvdesk/var
```

Symptom if you skip this: the Web UI shows a Symfony error like
`Unable to create the "cache" directory (/var/www/uvdesk/var/cache/dev)`.
Confirm the app's UID any time with:

```sh
docker exec uvdesk id uvdesk   # -> uid=1000(uvdesk) gid=1000(uvdesk)
```

If you'd rather not deal with it, simply **don't mount `/var/www/uvdesk/var`** —
it's cache/logs/sessions, and the in-image copy is already correctly owned and
writable.

---

## 5. Persisting your install (important)

Upstream bakes the app into `/var/www/uvdesk` inside the image, which makes
persistence awkward:

- You **cannot** bind-mount the whole `/var/www/uvdesk` from an empty appdata
  folder — an empty host directory shadows the installed application and you'll
  get a broken container.
- The template maps `/var/www/uvdesk/var` (cache, logs, sessions). But the
  installer's output — `.env` (which holds your `DATABASE_URL` and `APP_SECRET`)
  and `config/` — lives **outside** `var`. So if you **recreate** the container
  (an update, an "Apply" that rebuilds it), you lose your DB wiring and have to
  run the installer again.

### If you want your install to survive container recreation

Seed a populated app directory into appdata once, then bind-mount it back:

```sh
# 1. Create/start the container once from the template (let it come up).

# 2. Copy the image's app directory out to appdata (run on the Unraid host):
docker cp uvdesk:/var/www/uvdesk /mnt/user/appdata/uvdesk/app

# 3. Fix ownership so the in-container 'uvdesk' user can write to it:
#    (the uvdesk user/group is created by adduser in the image; UID/GID is
#    typically 1000:1000 on this base — verify with:
#      docker exec uvdesk id uvdesk
#    then chown to that UID:GID)
chown -R 1000:1000 /mnt/user/appdata/uvdesk/app

# 4. Edit the container: add a Path mapping
#      Container: /var/www/uvdesk   ->   Host: /mnt/user/appdata/uvdesk/app   (rw)
#    You can then drop the /var/www/uvdesk/var mapping (it's now inside the app mount).

# 5. Apply. Your .env, config, and uploads now live in appdata and survive
#    recreation and updates.
```

If you don't need installs to survive recreation (e.g. you're evaluating UVdesk),
you can skip this — just know a rebuild means re-running the installer.

---

## 6. Post-install config fixes

The installer writes a couple of values into `config/packages/uvdesk.yaml` that
are commonly wrong or blank after setup. Both are fixed the same way: edit the
file, then rebuild the cache. (These are runtime config the installer itself
manages — you're not modifying the image or upstream source.)

### `/member/login` throws a `member_prefix … null` error

The wizard's **Website configuration** step has a *Member Panel Prefix* and
*Customer Panel Prefix* field. If either is submitted **blank**, the installer
writes `uvdesk_site_path.member_prefix:` (empty → YAML `null`) into
`uvdesk.yaml`. Because the member routes are defined as
`/%uvdesk_site_path.member_prefix%/login`, a null value breaks **every**
`/member/*` route with:

```
A string value must be composed of strings and/or numbers, but found parameter
"uvdesk_site_path.member_prefix" of type "null" inside string value
"/%uvdesk_site_path.member_prefix%".
```

Worse, the finalization step calls the router right after writing the bad file,
so the "Installation" page **hangs forever** on the same null. One blank field,
both symptoms.

**The placeholder trap (this is the sneaky part).** The prefix fields render the
*current* value from `uvdesk.yaml` into their `value=`, with a grey
`placeholder="member"` as a hint. If `uvdesk.yaml` already holds a blank prefix,
the field is **empty** and the grey placeholder shows through — it *looks*
prefilled but submits nothing. So "I saw `member` and clicked Proceed" can still
write `null`. **Rule: if the field text is grey, it's empty. Click in and type a
real value so the text is black.** (Typing may grey out Proceed briefly while the
validator runs — that's normal; it re-enables once both fields hold valid,
distinct, alphanumeric values.)

**Why it won't reset on a "fresh" install.** `config/packages/uvdesk.yaml` lives
in the **image's app layer**, not in a mounted data directory. Wiping the MySQL
datadir or the `var` dir does **not** reset it — so once a blank prefix is saved,
every later wizard run re-reads the blank, re-shows empty placeholder fields, and
you re-submit blank. It only truly resets when the **container is recreated** from
the image (a template edit + Apply usually does this; a plain restart does not),
or when you fix the file directly (below).

**Avoid it:** run the installer in an **Incognito / private window** (section 3) —
a password-manager extension is the usual reason the prefix step won't accept your
input and silently submits blank. Then make sure the two prefix fields show
**black** text (real values like `member` / `customer`), not grey placeholders,
before clicking Proceed.

**Already broken?** Your schema and admin user already exist, so you don't need to
reinstall. Check the current value, then repair in place:

```sh
# See the value. Blank after the colon = the problem:
docker exec uvdesk grep uvdesk_site_path /var/www/uvdesk/config/packages/uvdesk.yaml

# Put valid prefixes back and rebuild the cache:
docker exec uvdesk sed -i \
  -e 's/^\(\s*uvdesk_site_path\.member_prefix:\).*/\1 member/' \
  -e 's/^\(\s*uvdesk_site_path\.knowledgebase_customer_prefix:\).*/\1 customer/' \
  /var/www/uvdesk/config/packages/uvdesk.yaml
docker exec uvdesk php /var/www/uvdesk/bin/console cache:clear --env=prod
```

Then log in at `http://<IP>:<PORT>/member/login` with the admin you created.
(Recreating the container from the image also resets `uvdesk.yaml` to the pristine
`member`/`customer` defaults, if you'd rather start the wizard clean.)

### `site_url` stays at `localhost:8000`

The installer **never sets** `uvdesk.site_url` — it's left at the shipped default
`localhost:8000`. It does **not** affect serving, login, or the wizard, so it's
not urgent. What it does affect: the **host** in absolute URLs UVdesk bakes into
**outbound email** (ticket notifications, invites, password resets) and some
embedded resource links — left default, those point at `http://localhost/…`,
wrong for anyone reaching the helpdesk remotely.

Note the consuming code parses the value and uses the **host only — the port is
dropped**. So a non-standard port like `:6744` can't be represented in generated
links. The clean fix is to front UVdesk with a reverse proxy on 80/443 under a
real domain and set `site_url` to that domain:

```sh
docker exec uvdesk sed -i "s|site_url: .*|site_url: 'helpdesk.example.com'|" \
  /var/www/uvdesk/config/packages/uvdesk.yaml
docker exec uvdesk php /var/www/uvdesk/bin/console cache:clear --env=prod
```

> Reminder: for `cache:clear` to stick, `var/cache` must be writable by the app
> (UID 1000). If you mounted `/var/www/uvdesk/var`, chown it to 1000 (section 4)
> or drop the mount — otherwise the rebuilt cache can fail to write.

---

## Quick reference

| Thing | Value / Field | Notes |
|---|---|---|
| Keep container alive | Post Arguments = `tail -f /dev/null` | The fix for "starts then stops" |
| Web UI | `http://<UNRAID-IP>:<PORT>` | Container port 80 |
| External DB | Leave *Built-in DB* fields blank | Enter DB details in the web installer |
| Built-in DB | Set `MYSQL_DATABASE/USER/PASSWORD/ROOT_PASSWORD` + `/var/lib/mysql` | Installer host = `127.0.0.1`; persistence caveat applies |
| "Skipping…local database" notice | Expected on external path | Yellow notice, not an error |
| `PUID` / `PGID` | Not supported — remove them | No LSIO remap layer; app is fixed at UID 1000 |
| Mounted-volume permissions | `chown -R 1000:1000` the host path | Unraid is 99:100; the app runs as 1000 |
| "Unable to create cache directory" | chown the `var` mount to 1000 (or drop it) | See section 4 |
| Persist install across rebuilds | `docker cp` seed of `/var/www/uvdesk` → bind-mount | See section 5 |
| Installer / prefix step stuck, Proceed greyed | Run the wizard in an **Incognito** window | Password managers (e.g. 1Password) break the prefix fields; see section 3 |
| Panel prefix fields | Type real values (black text, not grey placeholder) | Grey = empty → `member_prefix null` error + hung install; see section 6 |
| `/member/login` `member_prefix … null` | `sed` fix in `uvdesk.yaml` + `cache:clear` | See section 6 |
| `site_url` = `localhost:8000` | Set to your reverse-proxied domain | Installer never sets it; port is dropped in generated links; see section 6 |

---

*This image tracks upstream `uvdesk/community-skeleton` releases and is built
without modification. Application behavior, the installer, and env-var handling
are upstream's; this document only covers running it on Unraid.*

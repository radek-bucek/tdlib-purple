# Pidgin + Telegram (tdlib-purple) on Fedora 43

Procedure for getting Pidgin on Fedora 43 (KDE Plasma) to talk to Telegram
via the `tdlib-purple` plugin. The original upstream has been unmaintained
for years and Telegram refuses old TDLib protocol versions, so this setup
uses the **BenWiederhake/tdlib-purple** fork and a TDLib pinned to a
specific commit (version 1.8.35).

Target environment:
- Fedora 43, Pidgin 2.14.14, libpurple 2.14.14
- GCC 15.2.1
- TDLib commit `8d08b34e22a08e58db8341839c4e18ee06c516c5` (version 1.8.35, August 2024)
- tdlib-purple from `BenWiederhake/tdlib-purple` fork (master)

---

## 1) Build dependencies

```
sudo dnf install -y gcc gcc-c++ cmake make git gperf \
    openssl-devel zlib-devel libpurple-devel pkgconfig pidgin
```

---

## 2) Register an API_ID / API_HASH with Telegram

Telegram requires every client to identify itself with its own `api_id` +
`api_hash` pair. Public "shared" pairs that used to be baked into
open-source clients are progressively blocked.

1. Go to https://my.telegram.org → log in with your phone number and SMS code
2. Click "API development tools"
3. Fill in the form:
   - **App title:** anything (e.g. `pidgin tdlib`)
   - **Short name:** alphanumeric, 5–32 chars (e.g. `PidginTdlib`)
   - **URL:** must be non-empty, e.g. `https://github.com/ars3niy/tdlib-purple`
   - **Platform:** Desktop (must actually select the radio button!)
   - **Description:** a full sentence, not a single word
4. If submission only returns `ERROR` with no detail, see the **Troubleshooting** section.

Output: `api_id` (an integer) + `api_hash` (a 32-character hex string).
The hash is secret — treat it like a password.

---

## 3) Source checkout into /usr/local/src

```
cd /usr/local/src
sudo git clone https://github.com/tdlib/td.git
sudo git clone --depth=1 https://github.com/BenWiederhake/tdlib-purple.git
sudo git config --global --add safe.directory /usr/local/src/td
sudo git config --global --add safe.directory /usr/local/src/tdlib-purple
```

Check out the correct TDLib commit (the fork pins this via its submodule):

```
cd /usr/local/src/td
sudo -E git checkout 8d08b34e22a08e58db8341839c4e18ee06c516c5
grep "project.*VERSION" CMakeLists.txt   # must report 1.8.35
```

---

## 4) Build TDLib (5–10 min)

```
cd /usr/local/src/td
sudo rm -rf build && sudo mkdir build && cd build
sudo cmake -DCMAKE_BUILD_TYPE=Release \
           -DCMAKE_INSTALL_PREFIX:PATH=/usr/local ..
sudo make -j$(nproc)
sudo make install
```

This installs headers into `/usr/local/include/td/`, libraries into
`/usr/local/lib/`, and the CMake export into `/usr/local/lib/cmake/Td/`.

---

## 5) Patches applied to tdlib-purple

Two patches must be applied before building, both shipped in this same
directory:

- `01-runtime-api-credentials.patch` — runtime loading of API_ID / API_HASH
- `02-display-name-fallback.patch`  — fallback for nameless buddies

To apply (the source tree is owned by root after cloning, so take it over first):

```
sudo chown -R $(id -un):$(id -gn) /usr/local/src/tdlib-purple
cd /usr/local/src/tdlib-purple
git apply /home/jumbox/Plocha/tdlib-purple/01-runtime-api-credentials.patch
git apply /home/jumbox/Plocha/tdlib-purple/02-display-name-fallback.patch
git diff --stat   # sanity: 2 changed files, ~74 insertions, ~3 deletions
```

### Patch A — `01-runtime-api-credentials.patch`

Without this patch, the API_ID / API_HASH have to be baked into the binary
at compile time (`-DAPI_ID=... -DAPI_HASH=...`), which is impractical. The
patch adds a fallback chain:

1. Per-account UI fields in Pidgin (this part already exists in the fork)
2. Environment variables `TELEGRAM_API_ID` / `TELEGRAM_API_HASH`
3. File `~/.purple/telegram-tdlib-api.conf` in the form:
   ```
   api_id=12345
   api_hash=0123456789abcdef0123456789abcdef
   ```
4. Compile-time default

**Modifies:** `td-client.cpp` — adds includes, a new function
`loadRuntimeApiCredentials()`, and tweaks `sendTdlibParameters()`.

At login the plugin logs where the credentials came from:

```
telegram-tdlib: Using API_ID=36618871 from telegram-tdlib-api.conf
```

### Patch B — `02-display-name-fallback.patch`

Telegram users that have neither `first_name` nor `last_name` set used to
appear in Pidgin only as `idXXXXXXXX`. The patch adds a fallback chain:

1. `first_name + ' ' + last_name` (original behaviour, when something is set)
2. `@editable_username` (preferred Telegram username)
3. `@active_usernames[0]` (any other alias username)
4. `+phone_number` (when available)
5. `idXXXXXXXX` (final fallback)

**Modifies:** `client-utils.cpp`, function `makeBasicDisplayName()`.

---

## 6) Build and install the plugin

```
cd /usr/local/src/tdlib-purple
sudo rm -rf build && sudo mkdir build && cd build
sudo cmake -DCMAKE_BUILD_TYPE=Release \
           -DTd_DIR=/usr/local/lib/cmake/Td \
           -DNoVoip=True ..
sudo make -j$(nproc)
sudo make install
```

The plugin is installed as:

```
/usr/lib64/purple-2/libtelegram-tdlib.so
```

Roughly 39 MB (TDLib is linked statically — no runtime dependency on
`libtdjson`).

`-DNoVoip=True` disables voice calls, which don't work well over libpurple
anyway and would require the additional `libtgvoip` library.

---

## 7) User-side API configuration

```
mkdir -p ~/.purple
cat > ~/.purple/telegram-tdlib-api.conf <<EOF
api_id=YOUR_API_ID
api_hash=YOUR_API_HASH
EOF
chmod 600 ~/.purple/telegram-tdlib-api.conf
```

Priority: env `TELEGRAM_API_ID/HASH` > this conf > compile-time default.
Changes take effect only after a Pidgin restart (or account
disconnect/reconnect).

---

## 8) Launching Pidgin

```
pidgin &
```

Account: Accounts → Add → Protocol "Telegram (tdlib)" → enter your phone
number with country code (e.g. `+420...`) as the username. Telegram sends
an SMS code; Pidgin will pop up a dialog asking for it.

**Debug log** (useful when troubleshooting):

```
pkill -x pidgin; sleep 2
pidgin -d > /tmp/pidgin.log 2>&1 &
# log in again, then:
grep "telegram-tdlib" /tmp/pidgin.log | head -50
```

In the log, look for `Using API_ID=... from ...` — it confirms where the
plugin picked up its credentials.

---

## 9) Buddy list cleanup

Telegram pushes every account you've ever talked to into Pidgin — typically
dozens, including bots, channels, and anonymous accounts. A helper script
filters out "spam" buddies:

```
~/bin/telegram-buddy-cleanup.py            # dry-run, prints what it would delete
~/bin/telegram-buddy-cleanup.py --apply    # actually deletes
```

Safeguards:
- refuses to run while Pidgin is alive (would otherwise be overwritten)
- automatically backs up `~/.purple/blist.xml.bak-<date>`
- `--keep "<alias>"` (repeatable) — whitelist exceptions

Deletion criteria: empty alias, alias `idXXXXX`, alias `+420...`.

Inside Pidgin itself: menu **Buddies → Show** → uncheck *Offline Buddies*
and *Empty Groups* for clarity.

---

## 10) Building a Fedora RPM

Instead of the manual build out of `/usr/local/src`, the plugin can be
shipped as a relocatable RPM that:

- statically links TDLib 1.8.35 (no runtime dependency on a system TDLib)
- has API_ID / API_HASH baked in (install = one step, no conf file in
  `~/.purple/` required)
- applies both patches automatically during `%build`
- still honours runtime overrides via env / Pidgin UI / `.conf` as a fallback
  (useful if Telegram ever blocks the compiled-in API_ID)

Files in this directory used for that purpose:

| File                                  | Purpose                                          |
|---------------------------------------|--------------------------------------------------|
| `tdlib-purple.spec`                   | RPM spec (commit pins for fork + TDLib)          |
| `make-srpm.sh`                        | wrapper: clones, tars, calls rpmbuild            |
| `api-credentials.conf`                | API_ID / API_HASH (secret, **gitignored**)       |
| `01-runtime-api-credentials.patch`    | applied during `%prep`                           |
| `02-display-name-fallback.patch`      | applied during `%prep`                           |

Dependencies:
```
sudo dnf install -y rpm-build rpmdevtools git
```

Usage:
```
cd ~/Plocha/tdlib-purple
./make-srpm.sh              # SRPM only       → ~/rpmbuild/SRPMS/
./make-srpm.sh --rpm        # SRPM + RPM      → ~/rpmbuild/RPMS/x86_64/
```

On first run, when `api-credentials.conf` does not exist, the script
interactively asks for API_ID and API_HASH and writes them into
`~/Plocha/tdlib-purple/api-credentials.conf` with `chmod 600`. Subsequent
runs read the file silently.

**SRPM safety:** API_ID/HASH are passed to `rpmbuild` only as
`--define "_api_id N" --define "_api_hash X"` for the final binary build.
**The SRPM itself does not contain the credentials** — it is safe to share.
The recipient has to plug in their own pair via:
```
rpmbuild --rebuild --define "_api_id NNN" --define "_api_hash HEX" \
    ~/rpmbuild/SRPMS/tdlib-purple-*.src.rpm
```

Installing the resulting RPM:
```
sudo dnf install ~/rpmbuild/RPMS/x86_64/tdlib-purple-0.8.1-1.ben.fc43.x86_64.rpm
```

Then Pidgin → Accounts → Add → Telegram (tdlib). No `~/.purple/` config
file with credentials is needed — everything is in the binary.

**Removing an RPM-installed copy:**
```
sudo dnf remove tdlib-purple
```

---

## Key paths

| What                              | Where                                                        |
|-----------------------------------|--------------------------------------------------------------|
| TDLib source                      | `/usr/local/src/td`                                          |
| TDLib build artefacts             | `/usr/local/include/td/`, `/usr/local/lib/libtd*`            |
| tdlib-purple source (fork)        | `/usr/local/src/tdlib-purple`                                |
| tdlib-purple source (orig. bak)   | `/usr/local/src/tdlib-purple.ars3niy-bak`                    |
| Installed plugin                  | `/usr/lib64/purple-2/libtelegram-tdlib.so`                   |
| Old-plugin backup                 | `/usr/lib64/purple-2/libtelegram-tdlib.so.bak-<date>`        |
| API credentials                   | `~/.purple/telegram-tdlib-api.conf`                          |
| Telegram session/cache            | `~/.purple/tdlib/<phone>/`                                   |
| Pidgin buddy list                 | `~/.purple/blist.xml` (+ `.bak-<date>`)                      |
| Cleanup script                    | `~/bin/telegram-buddy-cleanup.py`                            |
| Debug log                         | `/tmp/pidgin.log` (only when pidgin is started with `-d`)    |
| RPM build script                  | `~/Plocha/tdlib-purple/make-srpm.sh`                         |
| RPM creds (gitignored)            | `~/Plocha/tdlib-purple/api-credentials.conf`                 |
| rpmbuild tree                     | `~/rpmbuild/{SRPMS,RPMS,SOURCES,SPECS,BUILD,BUILDROOT}`      |
| Source tarball cache              | `/tmp/tdlib-purple-rpmbuild-cache/`                          |

---

## Troubleshooting

### `Authentication error: code 406 (UPDATE_APP_TO_LOGIN)`

Telegram is refusing your client. Most common causes in order of likelihood:

1. **TDLib is too old** — Telegram rejects the MTProto layer. Fix: rebuild
   TDLib from a more recent commit (1.8.35 pinned above was working as of
   May 2026). If it breaks again, try the current `master` and the
   `BenWiederhake` fork's latest commit — the fork sometimes needs patches
   for newer TDLib API breaks.

2. **API_ID is blocked** — the plugin is using the upstream "public"
   default, which Telegram has already cut off. Fix: register your own
   pair (sections 2 + 7).

3. **Wrong API_ID/HASH in the conf file** — typo, nonexistent ID. Verify
   in the debug log that the plugin is using your ID
   (`Using API_ID=... from telegram-tdlib-api.conf`).

After any API change, delete the old session state:
```
rm -rf ~/.purple/tdlib/+420XXXXXXXXX
```

### my.telegram.org registration returns just `ERROR`

Telegram deliberately gives no detail. In practice:

1. **Platform radio button isn't actually selected** — pick Desktop.
2. **Empty URL** or too-short Description (< ~10 chars).
3. **Browser cache / cookies** — incognito mode, different browser, Ctrl+Shift+R.
4. **VPN / Tor / datacentre IP** — turn off, use a residential IP.
5. **New Telegram account** — under ~1-2 months old, my.telegram.org won't
   let you register an app no matter what you fill in.

### TDLib build fails

A full TDLib build needs ~8 GB of RAM. On a smaller machine use `make -j2`
instead of `-j$(nproc)`, or build TDLib through the `tdutils` target in
batches (see `https://github.com/tdlib/td#building`).

### tdlib-purple build fails with `'updateUserChatAction' in namespace 'td::td_api'`

This should not happen with the pinned commit 1.8.35, but if you try a
different TDLib version, stick to exactly what the fork wants — verify with:
```
cd /usr/local/src/tdlib-purple
git submodule status   # prints the TDLib commit hash the fork expects
```

---

## Long-term caveats

- **tdlib-purple is barely maintained.** Telegram periodically raises the
  required MTProto layer and the whole construction stops working. The
  remedy is either to find a newer commit of the fork, or give up and use
  the official Telegram Desktop.
- **The API hash is secret.** If it leaks you can revoke / regenerate it
  at https://my.telegram.org/apps.
- **Backups.** Every modification step backs up `blist.xml` and the
  previous plugin binary. If something breaks, restore the backup.

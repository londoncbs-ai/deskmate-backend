# Running the backend on a server

One Ubuntu server runs the backend as a service. Caddy in front of it gets a free HTTPS certificate and passes requests through. The frontend (on AWS Amplify) reaches it through a single rewrite rule, so the browser only ever talks to the frontend's own address and sign-in cookies keep working.

```
browser ─► https://deskmate.londoncbs.uk  (Amplify)  ── /api/* ──►  https://api.deskmate.londoncbs.uk  (this server: Caddy ─► the backend)
```

## What the server needs
- **Ubuntu 24.04 or newer**, 2 CPUs and about 4 GB of memory (the employees' browser is the hungry part).
- **A disk of 30 GB or more.** The databases, uploaded files and backups live on it.
- **A fixed (Elastic) IP address.** A normal public address changes when the server is stopped and started.
- **Firewall (AWS security group):** inbound **22** (SSH), **80** and **443** (HTTP and HTTPS, Caddy needs both).

## First deploy
From your own computer, in this repo:

```bash
HOST=ubuntu@<server-ip> KEY=~/.ssh/<your-key>.pem \
API_HOST=api.example.com APP_ORIGIN=https://app.example.com \
bash deploy/push.sh
```

It copies the code over SSH, then runs `setup.sh` on the server, which installs everything and starts the service. It is safe to run again; it never overwrites your settings or your data.

Then, two things only you can do:

1. **DNS.** Add an **A record** for the API address (for example `api.deskmate.londoncbs.uk`) pointing at the server's IP. Caddy fetches the HTTPS certificate on the first request once this resolves.
2. **Tell Amplify to send `/api` to the server.** Amplify console → your app → Hosting → *Rewrites and redirects* → *Manage redirects* → add, **above** the existing catch-all rule:

   | Source address | Target address | Type |
   |---|---|---|
   | `/api/<*>` | `https://api.deskmate.londoncbs.uk/api/<*>` | `200 (Rewrite)` |

Check: `curl https://deskmate.londoncbs.uk/api/health` should answer `{"ok":true}`. Until it does, the site says "Can't reach the server".

## Creating companies
Only your team can create companies: the sign-up page needs the **access code**, which is the `SIGNUP_CODE` line in `/etc/deskmate/backend.env` on the server. The public site does not offer "create a workspace"; your team opens `/signup` directly. People you invite into a company never need the code. To open sign-up to everyone later, delete the line and run `sudo systemctl restart deskmate`. To switch it off for everyone, set `SIGNUPS_OPEN=false`.

## Updating
Run `push.sh` again from your computer. It copies the new code and restarts the service.

## Where things are
| | |
|---|---|
| Settings | `/etc/deskmate/backend.env` (then `sudo systemctl restart deskmate`) |
| Data | `/var/lib/deskmate`, including the `.keyfile` that decrypts stored keys. **Back it up together.** |
| Logs | `journalctl -u deskmate -f` and `journalctl -u caddy -f` |
| Service | `sudo systemctl status deskmate` |

## Backups
The app makes its own daily backups under *Settings → Backups* (stored in the data folder, so they do not survive losing the disk). Also snapshot the server's disk regularly: AWS → EC2 → Lifecycle Manager can do this on a schedule.

## When something is wrong
- **The site says "Can't reach the server":** the Amplify rewrite is missing, or the API address does not resolve yet. Try `curl https://api.deskmate.londoncbs.uk/api/health`.
- **The service will not start:** `journalctl -u deskmate -n 80`.
- **No HTTPS certificate:** the DNS record is not pointing at this server yet, or ports 80/443 are closed in the security group.
- **Everyone appears to have the same IP address in the audit log:** Caddy could not read CloudFront's address list; run `sudo API_HOST=<your api address> bash /opt/deskmate/backend/deploy/caddy.sh`.

## When you are done with the SSH key
Remove its public half from `~/.ssh/authorized_keys` on the server (or rotate the key pair) so it can no longer log in.

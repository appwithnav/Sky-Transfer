# SKY Transfer ☁️

**Your files. Your Mac. One link to share.**

SKY Transfer is a self-hosted web app that turns your Mac into a temporary file-sharing server. Send files up to **20 GB each** through a browser, share individual download links or grouped ZIP downloads, and automatically clear expired transfers.

Files stay on your storage Mac. Cloudflare provides internet access and a permanent sharing address without requiring router access or port forwarding.

## Features

- **Large files:** Upload files up to 20 GB each.
- **Parallel uploads:** Transfer up to three files simultaneously.
- **Multiple-file selection:** Browse or drag and drop up to 100 files per selection. Add more while uploads are running.
- **Pause and resume:** Pause active uploads and continue from their saved offsets.
- **Grouped transfers:** Files selected together form one group. Separate selections remain separate transfers.
- **One-click downloads:** Download individual files or an entire completed group as a ZIP.
- **Streamed ZIPs:** Archives are generated during download without storing an extra ZIP on the Mac.
- **Automatic expiry:** Choose 1, 2, or 3 days before files are deleted.
- **Storage control:** A 100 GB limit includes space reserved for unfinished uploads.
- **Permanent sharing address:** A Cloudflare Worker follows the Mac’s changing tunnel URL automatically.
- **Background operation:** macOS login services can start the server, tunnel, and connection updater automatically.
- **Glass-style interface:** Responsive, translucent panels with keyboard focus indicators and reduced-motion support.

## How it works

```text
Sender / recipient browser
          │
          ▼
Cloudflare address
          │
          ▼
Current outbound tunnel
          │
          ▼
SKY Transfer server → Files stored on your Mac
```

The storage Mac publishes its current tunnel address to the connection service every 20 seconds. The service forwards requests to that tunnel, so newly generated sharing links keep the same public address after tunnel changes.

Cloudflare relays the traffic; it is not the permanent file storage location. This setup does not provide end-to-end encryption against the relay provider.

## Requirements

### Storage Mac

- macOS with Python 3.9 or later. Use a currently supported Python release compatible with your macOS when hosting online.
- Enough free disk space for the files you want to receive.
- An internet connection for remote sharing.
- The Mac must remain powered on, awake, and connected while serving transfers.

### Browser users

A modern browser with JavaScript enabled. No desktop app or browser extension is required. Uploading and managing transfers requires the access key; recipients only need the download link.

## Run locally

Open Terminal in the directory containing `server.py` and the `web/` folder, then run:

```sh
python3 server.py
```

Open:

```text
http://127.0.0.1:8787
```

On the first run, the server generates an access key at:

```text
~/Library/Application Support/PrivateCloud/admin-key
```

Open that file locally and enter its contents on the sign-in screen. Keep the key private: it grants management access to all transfers, including deletion.

To prevent idle sleep while the server is running:

```sh
caffeinate -i python3 server.py
```

This does not prevent shutdown, manual sleep, or all laptop-lid sleep behavior.

### Same-network access

The default server listens only on localhost. To allow devices on a trusted local network:

```sh
CLOUD_BIND=0.0.0.0 caffeinate -i python3 server.py
```

Open `http://YOUR-MAC-LAN-IP:8787` on another device. Allow incoming connections if the macOS firewall prompts you. Use HTTPS for a permanent deployment; local HTTP does not encrypt traffic.

### Configuration

| Environment variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `8787` | Server listening port |
| `CLOUD_BIND` | `127.0.0.1` | Listening address |
| `CLOUD_STORAGE` | `~/Library/Application Support/PrivateCloud` | Uploads, database, and access-key directory |

Changing `CLOUD_STORAGE` selects a different storage instance; it does not move existing files automatically.

## Share over the internet

### Temporary tunnel

Install the official [Cloudflare Tunnel client](https://developers.cloudflare.com/tunnel/get-started/quick-tunnels/), start the local server, then run:

```sh
cloudflared tunnel --url http://127.0.0.1:8787 --no-autoupdate
```

Open the printed HTTPS address. Quick Tunnel addresses change when the tunnel restarts, and Quick Tunnels have no uptime guarantee.

### Permanent address

For stable sharing links, deploy the accompanying Cloudflare Worker and connection registry using a Cloudflare account. A purchased domain is not required; the Worker can use a `workers.dev` address.

The connection setup includes:

1. A Worker that forwards requests to the current tunnel.
2. A Durable Object that stores the current destination.
3. A private publishing key that authenticates updates from the storage Mac.
4. A background publisher on the Mac that sends connection updates automatically.

If you are using the supplied cloud setup package, run **Setup SKY Cloud.command** on the storage Mac and complete the Cloudflare login in your browser. The setup requires Node.js and npm, deploys the service, and configures the host updater.

The public address may be shared. The owner’s `cloud-connection.json` contains a private publishing key and must not be committed to GitHub or distributed to recipients. Cloudflare’s [free-tier limits](https://developers.cloudflare.com/workers/platform/limits/) still apply.

Use the permanent address when creating new share links. Previously shared `trycloudflare.com` links cannot be made permanent after their original tunnel ends.

## Upload and share

1. Open SKY Transfer in your browser and sign in with your access key.
2. Choose an expiry of **1, 2, or 3 days**.
3. Click **Browse files** or drag files into the upload area.
4. Select several files with **Ctrl / ⌘** or **Shift**.
5. After uploading, use **Download**, **Download ZIP**, or the corresponding copy-link button.

One selected file stays a single transfer. Multiple files selected together form one group with a shared ZIP link. Files selected later form a separate transfer or group, even if their names match.

A group ZIP becomes available once every file in that group finishes uploading. Individual completed files inside the group can also be downloaded.

## Storage and deletion

Uploads are stored in:

```text
~/Library/Application Support/PrivateCloud
```

To open the folder in Finder:

```sh
open "$HOME/Library/Application Support/PrivateCloud"
```

Files use generated disk names. Their original names and transfer details are stored in SQLite and displayed in the interface. Do not rename stored files or manually edit the database.

- Single-file expiry starts when its upload completes.
- A completed group shares an expiry that starts when its last file finishes.
- Unfinished uploads and incomplete groups are cleared after 24 hours from creation.
- Expired links stop working immediately. Physical cleanup runs approximately every 30 seconds while the server runs, and also at startup.
- **Delete** removes a single transfer, or all files in the selected group, and invalidates its links.
- A download already underway can finish from an open file handle. Its disk space may remain occupied until that download ends.
- Deletion is ordinary filesystem deletion, not secure erasure. Backups and filesystem snapshots may retain copies.

The `admin-key` and `files.sqlite` files are application data and remain after uploads are deleted.

## Background startup

The supplied Mac host launcher installs the server and tunnel as user login services. The permanent-address setup adds the connection publisher.

These services start **when you log in**, not before login. Closing the browser does not stop them. Restarting the Mac interrupts active transfers; the services resume after login.

## Limitations

- The current authentication model uses one shared management key; there are no separate user accounts or per-user permissions.
- Upload resume matches files by name and size. Only resume with the exact same unchanged original; full-file checksum verification is not implemented in this web release.
- The in-memory upload queue survives pause/resume while the tab remains open. After a reload, reselect unfinished originals to resume.
- Streamed ZIP downloads do not support archive-level resume. Individual file downloads support HTTP byte ranges.
- A tunnel change preserves the permanent address, but an active transfer can still be interrupted. Uploads retry briefly; longer interruptions may require **Resume**.
- Transfer speed depends on both users’ connections, the storage Mac’s bandwidth and disk, and relay conditions.
- A full 20 GB end-to-end internet transfer has not yet been validated.

## Built with

- Python’s standard-library HTTP server
- SQLite for transfer metadata
- HTML, CSS, and vanilla JavaScript
- Cloudflare Tunnel for outbound internet connectivity
- Cloudflare Workers and Durable Objects for permanent connection routing

The file server requires no third-party Python packages. Cloud deployment uses Cloudflare’s Wrangler CLI.

## Project status

SKY Transfer is a personal, small-group file-sharing project under active development. Local checks cover authentication, concurrent uploads, resume offsets, file downloads and ranges, ZIP contents, storage limits, grouping, deletion, and expiry cleanup. A live connection test verified that the same download link worked before and after switching between two real tunnels.

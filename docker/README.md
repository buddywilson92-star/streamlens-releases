# StreamLens in Docker: any computer (beta)

StreamLens also comes as a container image. The same image runs on any
computer with Docker: plain Linux, a NAS (Synology, Unraid, TrueNAS, QNAP), and
Docker Desktop on Windows and macOS. It works on x86-64 (Intel/AMD) and ARM64
(Raspberry Pi 4/5, Apple silicon, ARM NAS boxes). It watches your Plex,
Jellyfin or Emby server and serves the same dashboard as the Windows version,
in any browser.

On a Windows computer that runs your media server, the
[Windows installer](https://getstreamlens.com) is still the simplest choice.
Docker is for everything else, and for people who already keep their apps in
Docker.

The image is `docker.io/tntdevteam/streamlens` on Docker Hub (a search for
`tntdevteam/streamlens` in a NAS's container app finds it). The tag `:1`
follows every 1.x release and is what the files below use; `:beta` has the
beta versions ([Updating](#updating)).

## Which file do I use?

All the files are in [StreamLens's Docker folder](https://github.com/buddywilson92-star/streamlens-releases/tree/main/docker),
where this guide is published too. Each section below has a command that
downloads the file it uses.

| You have | Use |
|---|---|
| A media server already running, on this computer or another one | **Standard**: [docker-compose.yml](docker-compose.yml) ([below](#standard-setup-any-computer)) |
| A Linux box or NAS, and StreamLens on your Windows PCs should find it by itself | **Host networking**: [docker-compose.host.yml](docker-compose.host.yml) ([below](#host-networking-linux-only)) |
| No media server yet, or you want it in Docker next to StreamLens | **All in one**: [docker-compose.plex.yml](docker-compose.plex.yml), [docker-compose.jellyfin.yml](docker-compose.jellyfin.yml) or [docker-compose.emby.yml](docker-compose.emby.yml) ([below](#all-in-one-streamlens-and-a-media-server)) |
| Portainer | Any of the files as a stack ([below](#portainer)) |
| Synology, QNAP or TrueNAS | The standard file in the NAS's own container app ([Synology](#synology-dsm-72-or-later), [QNAP](#qnap-container-station-3), [TrueNAS](#truenas-community-edition-docker-based-apps)) |
| Unraid | The template [streamlens.xml](unraid/streamlens.xml) ([below](#unraid)) |
| No compose at all | One `docker run` command ([below](#one-command-instead-of-a-compose-file)) |

## Before you start

- Docker: Docker Engine with the compose plugin on Linux, Docker Desktop on
  Windows 10/11 or macOS, or your NAS's container app. 64-bit only; 32-bit ARM
  is not supported.
- Your media server and an administrator account on it.
- A folder for StreamLens's data (settings, history, logs). Keep it private:
  it holds the key StreamLens uses to talk to your media server.

## What works differently from the Windows version

| | In Docker |
|---|---|
| Updates | You pull the new image ([Updating](#updating)). A container never updates itself. **Settings** → **Updates** tells you when a new version is out, with the commands. |
| Restart | **Restart StreamLens** in Settings works: StreamLens stops, and the container's restart policy (`restart: unless-stopped`) starts it again. Keep that line. |
| Finding it from Windows PCs | With the standard file, StreamLens on your Windows PCs can't find it by itself: type its address in **Connect** (`http://<its address>:3737`). With host networking on Linux they find it. With Docker Desktop they never do ([Docker Desktop](#docker-desktop-windows-and-macos)). |
| Server health tiles | CPU, memory and disks work. On Linux and a NAS they are the computer's own (disks and RAID volumes included). In Docker Desktop they are Docker's Linux virtual machine's, and the tiles say "Docker VM". Graphics tiles work for NVIDIA, AMD and Intel, including the graphics built into NAS processors, where the system reports load ([Graphics](#graphics-nvidia-amd-and-intel)). The media server's memory and conversion count need the computer's process list, which a container doesn't see: that tile says "Not visible from inside a container". Anything StreamLens can't measure says why instead of showing zeros. |
| Plex's own logs | Optional: mount Plex's Logs folder read-only for the Dolby Vision and CPU-fallback checks. Without it, **Settings** → **System** and the health panel say "Plex's log folder isn't available to StreamLens". Jellyfin and Emby need no folder: StreamLens reads their conversion logs over their own API. |
| Firewall | StreamLens doesn't change firewall rules in a container. If your NAS or Linux firewall is on, allow TCP port 3737 from your local network. Docker Desktop: allow Docker Desktop when Windows or macOS asks. |

## Graphics: NVIDIA, AMD and Intel

The graphics tiles carry your graphics chip's own name ("Graphics · Intel UHD
Graphics 600") and show a graph for everything the system reports. Nothing
unknown is shown as 0: a value the system doesn't report reads "Not reported",
with the reason. Under the tiles, a line says whether each conversion runs on
the graphics chip or on the processor, as your media server reports it
("Hardware encoding in use: Intel Quick Sync (Plex reports it)"). That line
works on every system, also where the load itself can't be measured.

| Graphics | What StreamLens needs | What you see |
|---|---|---|
| **AMD** (Radeon cards, Ryzen APUs) | nothing: it reads the load from `/sys` | 3D and video engine load (VCN), memory, temperature; a resting card shows "resting" |
| **Intel** (UHD/Iris/Arc built into the processor, Arc cards) | nothing: the name, the clock, "Graphics active" (how much of the time the whole chip works) and your media server's own report of which conversions use Quick Sync. That is the recommended setup. Only for each engine's load: `pid: host` with the media server running as the same user as StreamLens, which lets StreamLens into the media server's processes ([the trade-off](#pid-host-a-trade-off)) | "Graphics active" and the hardware line; with `pid: host`: also "Video encode + decode (Quick Sync)" and "Video processing", captioned "used by conversions" (Linux 5.19 or later; Arc B-series 6.11) |
| **NVIDIA** | the NVIDIA Container Toolkit on the computer, and a reservation in the compose file with `capabilities: [gpu, utility]` (the `deploy:` block in docker-compose.yml; Unraid: the Nvidia Driver plugin and `--runtime=nvidia` in **Extra Parameters**) | 3D, NVENC and NVDEC load, encoder sessions, memory, temperature |
| Arm NAS models (Realtek, Rockchip) | — | no graph (the chips don't report load); the hardware line still says what each conversion uses |
| No graphics chip (AMD Ryzen Embedded V1500B, R1600) | — | "This NAS's processor has no graphics chip, so conversions run on the processor" |

- **StreamLens never needs `/dev/dri`.** That device is for your media
  server's container, so it can convert with the chip (Plex Pass, Jellyfin,
  Emby Premiere): add `devices: ["/dev/dri:/dev/dri"]` to the media server
  (the all-in-one files have the lines, commented). StreamLens reads the load
  from `/sys`, which every container sees.
- **`pid: host`** lets StreamLens see the computer's processes: the media
  server's memory and conversion count, and how much of the graphics its
  conversions use. It stays optional and off in every compose file; read the
  trade-off below before you turn it on.
- **Docker Desktop** (Windows, Mac): its Linux virtual machine can't see AMD or
  Intel graphics. An NVIDIA card works with `--gpus all` (or the `deploy:`
  block). StreamLens for Windows shows every chip.

### pid: host: a trade-off

With `pid: host` the container sees every process of the computer, including
their command lines. The per-conversion graphics figures also need the media
server to run as the same user as StreamLens (user 1000, or your PUID); if it
doesn't, the tile says so. That same-user access is what lets StreamLens read
the conversions' graphics use (`/proc/<pid>/fdinfo`), and Linux gives it more
than that: the media server's files as the media server sees them
(`/proc/<pid>/root`, including Plex's `Preferences.xml` with Plex's own key),
its environment variables, and the right to stop its processes. Dropping every
privilege, the read-only system and `no-new-privileges` don't prevent it:
they limit what StreamLens can become, not what its own user may do, and
Docker's default AppArmor profile allows it between containers. So if
StreamLens were ever compromised, it could read the media server's key and
stop it.

- **Recommended: leave it off.** Intel graphics still shows its name, clock
  and "Graphics active" for the whole chip, and the line under the tiles says
  which conversions use Quick Sync, as the media server reports it. NVIDIA
  cards and AMD graphics show their own load without it (on a Linux older than
  6.10, or an older AMD chip, AMD's video engine row then says it needs the
  computer's processes).
- **Process memory and conversion count only:** `pid: host` with the media
  server running as a different user. StreamLens can then list the processes
  but not read the media server's files or stop it; the per-conversion
  graphics tiles say "The media server runs as a different user".
- **Everything:** `pid: host` and the same user, if you accept the above.

## Standard setup (any computer)

### Linux or a NAS, over SSH

1. Make a folder, give its `data` folder to the user StreamLens runs as (1000
   by default), and download the compose file into it:

   ```bash
   mkdir -p ~/streamlens/data && sudo chown 1000:1000 ~/streamlens/data
   cd ~/streamlens && curl -fLO https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.yml
   ```

   No `curl`? `wget https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.yml`
   does the same.
2. Optional: set your time zone in it (the `TZ:` line, for example
   `America/Chicago`) with a text editor, such as `nano docker-compose.yml`.
   Then start it:

   ```bash
   docker compose up -d
   ```

3. Open `http://<this-machine's-address>:3737` in a browser.
4. StreamLens asks for a **setup code**. The code proves you own the
   machine. Find it in the container log:

   ```bash
   docker logs streamlens
   ```

   Look for the line `use setup code XXXX-XXXX`. It is also in the file
   `data/setup-code.txt`.
5. Create your admin account and connect your media server (Plex: sign in;
   Jellyfin or Emby: its address and an API key). Done.

A media server **on the same computer** is at `http://host.docker.internal:<port>`
from inside the container (`http://host.docker.internal:32400` for Plex,
`:8096` for Jellyfin and Emby). The standard file gives the container that
name on Linux too. If a Linux firewall such as ufw blocks it, allow the
media server's port from Docker's networks. Plex's sign-in during setup
usually finds the right address by itself.

### Docker Desktop (Windows and macOS)

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   and start it. In its **Settings** → **General**, turn on **Start Docker
   Desktop when you sign in**. Docker Desktop is an app, not a service: after
   the computer restarts, StreamLens runs again only once someone signs in to
   the computer. (The Windows installer runs StreamLens as a service instead,
   which starts without anyone signing in.)
2. Make a folder inside your user folder and download the compose file into
   it. Windows, in **PowerShell**:

   ```powershell
   mkdir -Force $HOME\StreamLens; cd $HOME\StreamLens
   Invoke-WebRequest -UseBasicParsing https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.yml -OutFile docker-compose.yml
   ```

   Mac, in **Terminal**:

   ```bash
   mkdir -p ~/StreamLens && cd ~/StreamLens
   curl -fLO https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.yml
   ```

   On Windows only your account can open a folder inside your user folder,
   which keeps StreamLens's data private. Nothing else to prepare: Docker
   makes the `data` folder.
3. Optional: set your time zone in `docker-compose.yml` (the `TZ:` line, for
   example `America/Chicago`), with Notepad on Windows (`notepad docker-compose.yml`)
   or TextEdit on a Mac (`open -e docker-compose.yml`).
4. Start it, in the same window (still in that folder):

   ```bash
   docker compose up -d
   ```

   Later, in a new window, go to the folder first: `cd $HOME\StreamLens` in
   PowerShell, `cd ~/StreamLens` in Terminal on a Mac.
5. Open `http://localhost:3737` on this computer. The setup code is in
   `docker logs streamlens` (or Docker Desktop → **Containers** →
   **streamlens** → **Logs**).
6. Plex, Jellyfin or Emby installed **on this same computer**: use
   `http://host.docker.internal:32400` (Plex) or `http://host.docker.internal:8096`
   (Jellyfin, Emby) as its address. For Plex, the sign-in during setup usually
   finds it by itself; if it doesn't, uncomment `PLEX_URL` in the compose file
   (it already says `http://host.docker.internal:32400`) and run
   `docker compose up -d` again.
7. Other devices open `http://<this computer's address>:3737`. Find the
   address in Windows **Settings** → **Network & internet** → your connection's
   **Properties** (IPv4 address), or on a Mac in **System Settings** →
   **Network** → your connection → **Details**. When Windows asks whether
   Docker Desktop may accept connections, allow it on **private** networks.
   In StreamLens, **Settings** → **Computers & people** → **Connect another
   computer** shows the same advice.

What to know about Docker Desktop:

- **Health figures.** Docker Desktop runs containers in its own small Linux
  virtual machine. The CPU, memory and disk tiles show that virtual machine, not
  your Windows PC or Mac, and they say so ("Docker VM"; **Settings** →
  **System** explains it). They are your media server's figures only when the
  media server also runs in Docker Desktop. How much of the computer the virtual
  machine gets is set in Docker Desktop's **Settings** → **Resources** (with
  WSL 2 on Windows: the `.wslconfig` file).
- **Finding it.** Your other computers never find StreamLens in Docker Desktop
  by themselves: type the address (step 7) in **Connect** on them. Docker
  Desktop's host networking (**Settings** → **Resources** → **Network**) is not
  the same as on Linux: "host" there is the virtual machine, so it doesn't help.
  Don't use docker-compose.host.yml on Docker Desktop.
- **Which device asked.** Docker Desktop passes connections on to the container
  from its own address, so every device reaches StreamLens from Docker's
  address (such as `172.18.0.1`), and a device asking to connect shows that
  address instead of its own. Check the device name before you click **Allow**.
  For the same reason StreamLens can't tell a device at home from one on the
  internet there: never forward port 3737 on your router to a Docker Desktop
  computer.
- **Wrong passwords count for every device together.** StreamLens can't tell
  your devices apart in Docker Desktop, so wrong passwords from any of them add
  up: after five wrong tries, every device (the one at this computer too) has
  to wait before the next try. In Docker Desktop that wait is never longer than
  a minute. A name that keeps getting wrong passwords also waits on its own,
  up to 15 minutes, from whichever device. **Settings** → **System** says so too.
- **The data folder.** A folder shared from Windows shows up in the container as
  open to everyone, and StreamLens can't change that from inside, so its log
  says the folder's privacy is up to Windows. That is why step 2 puts it inside
  your user folder. Or keep the data in a Docker volume instead of a folder:
  replace `- ./data:/data` with `- streamlens-data:/data` and add these two
  lines at the end of the file, starting at the left edge:

  ```yaml
  volumes:
    streamlens-data:
  ```

### One command instead of a compose file

The same standard setup as one `docker run`. Linux and macOS (make the data
folder first on Linux, as above):

```bash
docker run -d --name streamlens --hostname streamlens --restart unless-stopped --init --user 1000:1000 -p 3737:3737 --add-host host.docker.internal:host-gateway -e TZ=Etc/UTC -v "$PWD/data:/data" --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges:true --sysctl "net.ipv4.ping_group_range=0 2147483647" docker.io/tntdevteam/streamlens:1
```

Windows PowerShell, in your StreamLens folder:

```powershell
docker run -d --name streamlens --hostname streamlens --restart unless-stopped --init --user 1000:1000 -p 3737:3737 -e TZ=Etc/UTC -v "${PWD}\data:/data" --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges:true --sysctl "net.ipv4.ping_group_range=0 2147483647" docker.io/tntdevteam/streamlens:1
```

Add `-e PLEX_URL=http://host.docker.internal:32400` and the like for the
settings in [Settings from the container](#settings-from-the-container-optional).
To update: `docker pull docker.io/tntdevteam/streamlens:1`, then
`docker rm -f streamlens` and the same `docker run` again (the data folder
keeps everything).

## Keys and Docker secrets

Connecting the media server in the browser is the easiest way: StreamLens
keeps the key in its data folder. When you set up StreamLens from a file
instead, give it the key **as a file** (a Docker secret), never as text in the
compose file: a key in `environment:` is shown by `docker inspect` to anyone
who can use Docker on that computer.

| Media server | Address | Key from a file (recommended) | Key as text |
|---|---|---|---|
| Plex | `PLEX_URL` | `PLEX_TOKEN_FILE` | `PLEX_TOKEN` |
| Jellyfin | `JELLYFIN_URL` | `JELLYFIN_API_KEY_FILE` | `JELLYFIN_API_KEY` |
| Emby | `EMBY_URL` | `EMBY_API_KEY_FILE` | `EMBY_API_KEY` |

1. Get the key. Jellyfin: signed in as an administrator, **Dashboard** →
   **API Keys** → **New API Key**, type StreamLens as the app name, **Create**.
   Emby: the gear icon → **Api Keys** (under Advanced) → **New Api Key**. Plex: signing in during setup is much easier; a Plex key
   (token) only when you know you need one.
2. Save it, alone on one line, in `secrets/jellyfin_api_key.txt` next to the
   compose file (`plex_token.txt`, `emby_api_key.txt` for the others).

   Linux or a NAS, over SSH: let only StreamLens's user read it. That is user
   1000, or your PUID with the [PUID and PGID start](#starting-as-root-with-puid-and-pgid)
   (then use your PUID and PGID instead of `1000:1000`):

   ```bash
   mkdir -p secrets && chmod 700 secrets
   nano secrets/jellyfin_api_key.txt      # paste the key, save
   sudo chown 1000:1000 secrets/jellyfin_api_key.txt && sudo chmod 400 secrets/jellyfin_api_key.txt
   ```

   Windows, in PowerShell in your StreamLens folder: Notepad asks whether to
   create the file (**Yes**); paste the key, save, close Notepad. The folder is
   inside your user folder, so only your account can open it.

   ```powershell
   mkdir -Force secrets
   notepad secrets\jellyfin_api_key.txt
   ```

   Mac, in Terminal in your StreamLens folder: copy the key first, then (this
   writes what you copied into the file):

   ```bash
   mkdir -p secrets && pbpaste > secrets/jellyfin_api_key.txt
   ```

3. In the compose file, uncomment the `JELLYFIN_API_KEY_FILE` line, the
   `secrets:` lines under the service and the `secrets:` block at the end:

   ```yaml
       environment:
         JELLYFIN_URL: "http://jellyfin:8096"
         JELLYFIN_API_KEY_FILE: /run/secrets/jellyfin_api_key
       secrets:
         - jellyfin_api_key
   # (at the end, starting at the left edge)
   secrets:
     jellyfin_api_key:
       file: ./secrets/jellyfin_api_key.txt
   ```

4. `docker compose up -d`. The log says
   `JELLYFIN_API_KEY_FILE supplies the Jellyfin connection for this run`.

What StreamLens does with it: it reads the file once at start, uses the key
only for the address in `JELLYFIN_URL`, never writes it to its settings, never
prints it in its log, and **Settings** shows the connection as "set by the
container". Docker mounts the file at start, so the key is never part of the
image, and `docker inspect` shows only the file's name. To change the key,
replace the file and run `docker compose up -d --force-recreate`. A key that
can't be read (wrong owner, empty file) is reported in the log without the
key, and the connection saved in Settings is used instead.

## Starting as root with PUID and PGID

Many NAS guides start containers as root with two numbers, PUID and PGID, for
the user and group that should own the data (Unraid's template does this).
StreamLens can start that way: it is root only long enough to give the data
folder to that user, then runs as that user with every privilege dropped. In
the compose file:

1. Change `user: "1000:1000"` to `user: "0:0"`.
2. Under `environment:`, add `PUID: "1026"` and `PGID: "100"`, with your own
   account's numbers: over SSH, `id <your user name>` shows them
   (`uid=1026(...) gid=100(users)`).
3. Replace the two lines `cap_drop:` and `- ALL` with these two (four spaces
   in, like the lines around them):

   ```yaml
       cap_drop: [ALL]
       cap_add: [CHOWN, FOWNER, DAC_OVERRIDE, SETUID, SETGID, SETPCAP]
   ```

   Without them the container stops at once, and its log names this line.

The same with `docker run`: `--user 0:0 -e PUID=1026 -e PGID=100 --cap-drop ALL
--cap-add CHOWN --cap-add FOWNER --cap-add DAC_OVERRIDE --cap-add SETUID
--cap-add SETGID --cap-add SETPCAP`. A key file ([Keys and Docker secrets](#keys-and-docker-secrets))
must then be readable by your PUID, not by 1000.

## Host networking (Linux only)

Use [docker-compose.host.yml](docker-compose.host.yml)
instead of the standard file when:

- StreamLens on your Windows PCs should find the NAS by itself (it answers on
  UDP port 3737), or
- your media server runs on the same box and only listens on `127.0.0.1`
  (then `PLEX_URL: "http://127.0.0.1:32400"`).

Download it with
`curl -fL -o docker-compose.yml https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.host.yml`
(saved as `docker-compose.yml`, so `docker compose up -d` finds it).

It works on Linux Docker hosts only (Synology, Unraid, TrueNAS, QNAP, plain
Linux), not on Docker Desktop for Windows or macOS.

## All in one: StreamLens and a media server

Each of these files starts a media server and StreamLens together, in one
compose project. StreamLens reaches the media server by its service name
inside the project's own network: `http://plex:32400`, `http://jellyfin:8096`,
`http://emby:8096`. Both see your media at the same path (`/media`, read-only
for StreamLens), so keyframe analysis needs no path map.

| File | Media server | Evidence StreamLens reads |
|---|---|---|
| [docker-compose.plex.yml](docker-compose.plex.yml) | `lscr.io/linuxserver/plex` | Plex's log folder, mounted read-only at `/plex-logs` (Plex writes its logs to `./plex/logs`) |
| [docker-compose.jellyfin.yml](docker-compose.jellyfin.yml) | `jellyfin/jellyfin` | each conversion's ffmpeg log, over Jellyfin's API (no folder) |
| [docker-compose.emby.yml](docker-compose.emby.yml) | `emby/embyserver` | each conversion's ffmpeg log, over Emby's API (no folder) |

Steps (the top of each file has them too):

1. Make an empty folder, download the file into it as `docker-compose.yml`,
   and make the folders it names. On Linux or a NAS, give them to user 1000.
   For Plex:

   ```bash
   curl -fL -o docker-compose.yml https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.plex.yml
   mkdir -p plex/config plex/logs plex/transcode streamlens/data media
   sudo chown -R 1000:1000 plex streamlens
   ```

   For Jellyfin or Emby, download
   `https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.jellyfin.yml`
   or `https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/docker-compose.emby.yml`
   the same way; the folders to make are at the top of the file. In PowerShell,
   use `Invoke-WebRequest -UseBasicParsing <address> -OutFile docker-compose.yml`.
2. Change `./media` (in both services) to where your films and shows are.
3. **Plex:** for a new server, get a claim code at <https://plex.tv/claim>
   (valid for 4 minutes), put it in `PLEX_CLAIM`, then `docker compose up -d`.
   Open `http://<this computer>:32400/web` to set Plex up, then StreamLens at
   `http://<this computer>:3737` and sign in to Plex there. Then, in Plex,
   **Settings** → **Network** (click **Show Advanced**), set two things and
   **Save Changes**:
   - **LAN Networks**: your home network, for example `192.168.1.0/24`.
     Otherwise Plex counts your own devices as remote and holds them to its
     remote-streaming quality.
   - **Custom server access URLs**: this computer's own address with port
     32400, for example `http://192.168.1.10:32400`. In a Docker network Plex
     only knows its container address (such as `172.18.0.2`), which your TV and
     phone can't reach, so without this they play through Plex's relay, which
     is slow and limited: exactly the buffering StreamLens is there to explain.

   On Linux (not Docker Desktop) you can instead give Plex host networking:
   in the `plex` service, replace the two `ports:` lines with
   `network_mode: host`; in the `streamlens` service, change `PLEX_URL` to
   `http://host.docker.internal:32400` and add the two `extra_hosts:` lines
   from docker-compose.yml. Plex then sees your home network itself, and the
   two settings above aren't needed.
4. **Jellyfin or Emby:** start the media server alone first
   (`docker compose up -d jellyfin`), finish its setup at
   `http://<this computer>:8096`, create an API key, then either paste the key
   during StreamLens's setup (it opens on Jellyfin or Emby, with the address
   filled in), or save it as a Docker secret
   ([Keys and Docker secrets](#keys-and-docker-secrets)). Then
   `docker compose up -d`.

On Docker Desktop these files are good for trying things out. For a real media
server on a Windows PC or Mac, install the media server itself and StreamLens
with the standard file: hardware transcoding is limited or missing in Docker
Desktop (none on a Mac), and the media server sees every device arrive from
Docker's own address.

## Portainer

Any of the files works as a Portainer stack: **Stacks** → **Add stack** →
**Web editor**, paste the file, **Deploy the stack**. (Open the file's link in
[Which file do I use?](#which-file-do-i-use), then use the copy button above
its text.) Change two things first:

- **Folders:** in the web editor, `./data` and other relative paths are made
  inside Portainer's own storage, not next to anything you can find. Use full
  paths, for example `/opt/streamlens/data:/data` (and give that folder to user
  1000), or a Docker volume (`streamlens-data:/data` plus a `volumes:` block,
  as in the Docker Desktop notes).
- **Secrets:** a secret file needs a full path too
  (`file: /opt/streamlens/secrets/jellyfin_api_key.txt`). Don't put a key in
  Portainer's **Environment variables** box: those end up in `docker inspect`.

The setup code is in **Containers** → **streamlens** → **Logs**. To update:
the stack → **Editor** → **Update the stack** with **Re-pull image and
redeploy** turned on.

## Synology (DSM 7.2 or later)

1. **Package Center** → install **Container Manager**.
2. **File Station** → in the `docker` shared folder, create
   `streamlens/data`.
3. **Container Manager** → **Project** → **Create**:
   - Project name: `streamlens`
   - Path: `/docker/streamlens`
   - Source: **Create docker-compose.yml**, and paste the text of
     [docker-compose.yml](docker-compose.yml) (open the
     link, then use the copy button above the text).
4. Folder owner. Pick one of these:
   - **Simplest:** the [PUID and PGID start](#starting-as-root-with-puid-and-pgid),
     in the pasted text: `user: "0:0"`, your own account's numbers as `PUID`
     and `PGID` (over SSH, `id <your DSM user name>` shows them; the first DSM
     account is often 1026 and group 100), and the `cap_add:` line.
   - **Or:** keep `user: "1000:1000"` and give user 1000 the folder over SSH:
     `sudo chown 1000:1000 /volume1/docker/streamlens/data`.
5. Optional, for Plex's own evidence: uncomment the `/plex-logs` line and point
   it at `/volume1/PlexMediaServer/AppData/Plex Media Server/Logs`.
6. **Next** → **Done**. Container Manager builds and starts the project.
7. Open `http://<nas-ip>:3737`. The setup code is in **Container Manager** →
   **Container** → **streamlens** → **Log**.
8. If DSM's firewall is on (**Control Panel** → **Security** → **Firewall**),
   allow TCP port 3737 from your local network.

Graphics on a Synology: models with Intel graphics (for example DS218+,
DS720+, DS920+, DS224+, DS423+) show the chip's name and "Graphics active".
DSM's Linux (4.4 or 5.10) doesn't report the video engine's load, so the
line under the tiles, from your media server, says whether conversions use
Quick Sync. Models with an AMD Ryzen V1500B or R1600 (DS923+, DS1621+ and
others) have no graphics chip, and the tile says so. On the DS225+ and DS425+
DSM may load no graphics driver at all; the tile then says so too
([Graphics](#graphics-nvidia-amd-and-intel)).

## QNAP (Container Station 3)

1. **App Center** → install **Container Station**.
2. **File Station** → in the `Container` shared folder, create
   `streamlens/data`.
3. Folder owner. Pick one of these:
   - Give user 1000 the folder over SSH (turn SSH on in **Control Panel** →
     **Network & File Services** → **Telnet / SSH**):
     `chown 1000:1000 /share/Container/streamlens/data`.
   - Or the [PUID and PGID start](#starting-as-root-with-puid-and-pgid) with
     your own account's numbers (over SSH: `id <your user name>`).
4. **Container Station** → **Applications** → **Create**: name it
   `streamlens`, paste the text of
   [docker-compose.yml](docker-compose.yml) and change
   `./data` to the full path, `/share/Container/streamlens/data` (a relative
   path would end up inside Container Station's own storage). **Validate**,
   then **Create**.
5. Open `http://<nas-ip>:3737`. The setup code is in **Container Station** →
   **Containers** → **streamlens** → **Logs**.
6. If QuFirewall is on, allow TCP port 3737 from your local network.

Graphics on a QNAP: Intel graphics (Celeron J and N models) show the chip's
name and "Graphics active"; QTS's Linux (5.10) doesn't report the video
engine's load, so the line under the tiles, from your media server, says
whether conversions use Quick Sync ([Graphics](#graphics-nvidia-amd-and-intel)).

## Unraid

1. Get the template: in Unraid's **Terminal** (the `>_` icon at the top), run

   ```bash
   curl -fL -o /boot/config/plugins/dockerMan/templates-user/my-StreamLens.xml https://raw.githubusercontent.com/buddywilson92-star/streamlens-releases/main/docker/unraid/streamlens.xml
   ```

   Then **Docker** → **Add Container** → **Template**: choose **StreamLens**
   under the user templates. A Community Applications listing is planned.
2. Keep the defaults:
   - Data: `/mnt/user/appdata/streamlens`
   - PUID 99 / PGID 100 (Unraid's usual `nobody:users`)
   - Port 3737
3. Optional: set **Plex logs** to your Plex container's Logs folder, for
   example `/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Logs`
   for linuxserver/plex.
4. **Apply**. Then open the WebUI from the container's menu. The setup code
   is in the container's **Logs**.

Graphics on Unraid: Intel and AMD graphics show without extra settings. For
how much of the graphics the conversions use, you can add `--pid=host` to
**Extra Parameters** (your media server running as 99:100, like StreamLens),
but read [the trade-off](#pid-host-a-trade-off) first. An NVIDIA
card needs the **Nvidia Driver** plugin: add `--runtime=nvidia` to **Extra
Parameters** and the variables `NVIDIA_VISIBLE_DEVICES` (`all`) and
`NVIDIA_DRIVER_CAPABILITIES` (`utility`), as the plugin describes
([Graphics](#graphics-nvidia-amd-and-intel)).

## TrueNAS (Community Edition, Docker-based apps)

1. Create a dataset for the data, for example `/mnt/tank/apps/streamlens`,
   with a `data` folder inside.
2. **Apps** → **Discover Apps** → **⋮** → **Install via YAML**.
3. Paste the text of [docker-compose.yml](docker-compose.yml)
   (open the link, then use the copy button above the text) and change
   `./data` to the full path, for example `/mnt/tank/apps/streamlens/data`.
4. Set `user:` to the owner of that folder. TrueNAS's apps user is 568:568,
   so `user: "568:568"`. Or use the [PUID and PGID start](#starting-as-root-with-puid-and-pgid).
5. Save. The setup code is in the app's logs.

TrueNAS checks only that the YAML is well-formed, so copy it exactly.

Graphics on TrueNAS: Intel and AMD graphics show without extra settings. For
how much of the graphics the conversions use, you can uncomment `pid: host`
(your media app running as the same user, for example 568), but read
[the trade-off](#pid-host-a-trade-off) first. An NVIDIA card needs TrueNAS's
NVIDIA driver and the `deploy:` block of the compose file
([Graphics](#graphics-nvidia-amd-and-intel)).

## Settings from the container (optional)

Everything can be set in the browser. A few settings can also come from the
container's `environment:`, which helps when you set up several boxes the same
way. While a variable is set it wins: **Settings** shows that field as "set by
the container" and doesn't let you change it there. Nothing is written back to
StreamLens's settings file, so removing the variable brings back what Settings
said.

| Variable | What it sets |
|---|---|
| `TZ` | Your time zone, for times in the log, e.g. `America/Chicago` (default `Etc/UTC`) |
| `STREAMLENS_NAME` | The name your other computers see (default: your media server's name) |
| `STREAMLENS_ADDRESSES` | The addresses your phones use to reach StreamLens, e.g. `192.168.1.10` (several: separated by commas). In a container StreamLens proves who it is only at those addresses, with any compose file, because from inside a container it can't tell which addresses are really yours. Without it, phones won't connect to it from a new address |
| `PLEX_URL` | Plex's address, e.g. `http://192.168.1.10:32400`, `http://host.docker.internal:32400` (Plex on this same computer) or `http://plex:32400` (all in one). The Plex sign-in in setup usually finds it by itself |
| `PLEX_TOKEN_FILE` or `PLEX_TOKEN` | Plex's access key, from a Docker secret file (preferred) or the variable itself ([Keys and Docker secrets](#keys-and-docker-secrets)). Signing in to Plex in setup is easier. |
| `JELLYFIN_URL` | Jellyfin's address, e.g. `http://192.168.1.10:8096` or `http://jellyfin:8096` (all in one). While nothing is connected yet, setup then opens on Jellyfin with this address. Jellyfin support is new: beta |
| `JELLYFIN_API_KEY_FILE` or `JELLYFIN_API_KEY` | An access key Jellyfin made for StreamLens (in Jellyfin, signed in as an administrator: **Dashboard**, **API Keys**, **New API Key**, **Create**). With it, and no Plex access key, StreamLens watches Jellyfin. Connecting Jellyfin in setup or Settings is easier. |
| `EMBY_URL` | Emby's address, e.g. `http://192.168.1.10:8096` or `http://emby:8096` (all in one). While nothing is connected yet, setup then opens on Emby with this address. Emby support is new: beta |
| `EMBY_API_KEY_FILE` or `EMBY_API_KEY` | An access key Emby made for StreamLens (in Emby, signed in as an administrator: the gear icon at the top right, **Api Keys** under Advanced, **New Api Key**; in a narrow window, your user picture and **Manage Emby Server** first). With it, and no Plex or Jellyfin access key, StreamLens watches Emby. Connecting Emby in setup or Settings is easier. |
| `PLEX_LOG_DIR` | Plex's Logs folder inside the container, if you mounted it somewhere other than `/plex-logs` |
| `STREAMLENS_PATH_MAP` | Where the media files your media server names are inside this container (below) |
| `STREAMLENS_PORT` | The dashboard port inside the container (default 3737; change the `ports:` line to match) |
| `STREAMLENS_BIND` | Which address it listens on inside the container (default all) |
| `STREAMLENS_DISCOVERY` | `on` or `off`: whether other computers may find StreamLens by themselves |
| `PUID`, `PGID` | Only when the container starts as root (`user: "0:0"`, Unraid): the user and group that get the data folder and run StreamLens (never 0; [PUID and PGID](#starting-as-root-with-puid-and-pgid)) |

The container's health check (what Docker, Unraid, Portainer and autoheal
show as healthy or unhealthy) asks StreamLens at the port and address it
really listens on: StreamLens writes them to `/tmp/streamlens-health` when
it starts. So a port changed with `STREAMLENS_PORT` or in **Settings** →
**Network access**, or a `STREAMLENS_BIND` of one address (macvlan, host
networking), never makes a healthy container look unhealthy. (Without a
writable `/tmp` it falls back to `STREAMLENS_PORT` and `STREAMLENS_BIND`.)

**Keyframe analysis** reads the media file being played. The media server
names the file by its own path (for example `/volume1/video/Movies/x.mkv`, or
`D:\Movies\x.mkv` when Plex runs on Windows). Mount your media read-only and
tell StreamLens where the media server's folders are:

```yaml
    environment:
      STREAMLENS_PATH_MAP: "/volume1/video=/media"
    volumes:
      - "/volume1/video:/media:ro"
```

Several folders are separated with `;`: `"/movies=/media/movies;/tv=/media/tv"`.
The longest matching folder wins; other paths are used as the media server
names them. **Settings** → **System** lists the folders in use. The all-in-one
files mount the media at the same path in both containers, so they need no
map.

## Updating

StreamLens shows when a new version is out. To update:

```bash
cd ~/streamlens && docker compose pull && docker compose up -d
```

Docker Desktop: the same two commands in PowerShell or Terminal, in your
StreamLens folder. On Synology: **Container Manager** → **Project** →
**streamlens** → **Action** → **Build** (it pulls the new image). On QNAP: in
**Container Station** → **Applications**, open streamlens's menu and recreate
it so it pulls the image again. On Unraid: **Docker** → **Check for Updates** →
**Apply Update**. Portainer: see above. Your settings and history stay in the
data folder.

The compose files ask for the image tagged `:1`, which follows every 1.x
release. **Beta versions** come under the tag `:beta`: to try them, choose the
beta channel in **Settings** → **Updates**, change the end of the `image:` line
from `:1` to `:beta` (Unraid: the **Repository** field), then pull as above.
Choosing the beta channel alone changes nothing in a container: the tag
decides what you get. To leave, choose the stable channel and change the tag
back to `:1`, but only once a stable version at least as new as the beta you
run is out (Settings → Updates shows the newest one): an older version can't
read the history a newer one has upgraded.

**Settings** → **Updates** shows the new image's name and its digest (a
fingerprint like `sha256:…`). To check you got exactly that image:
`docker image inspect --format "{{index .RepoDigests 0}}" <image>:<tag>`.
When a version's notes mention a database change, copy the data folder first
(new versions upgrade the history database, and older versions can't read it
back).

## If something goes wrong

| Symptom | Fix |
|---|---|
| The log says "StreamLens can't write to its data folder" | The data folder belongs to another user. Run the `chown` shown in the log, or use the [PUID and PGID start](#starting-as-root-with-puid-and-pgid). |
| The container stops at once, and the log says it "is missing privileges it needs" | The PUID and PGID start needs the `cap_add:` line ([PUID and PGID](#starting-as-root-with-puid-and-pgid)). |
| The browser can't reach port 3737 | Check the firewall (NAS or Linux: allow TCP 3737 on the local network; Docker Desktop: allow Docker Desktop on private networks), and that nothing else uses port 3737 (then change the first number of the `ports:` line, for example `"3738:3737"`). |
| The media server shows as not reachable | Sign in again, or check its address in Settings. From inside a container, `127.0.0.1` is the container itself: use the computer's address, `host.docker.internal` (same computer) or the service name (all in one). |
| Plex plays through its relay, or TVs and phones say the server is remote (all in one) | Set **Custom server access URLs** and **LAN Networks** in Plex ([All in one](#all-in-one-streamlens-and-a-media-server), step 3). |
| A graphics tile says "Not reported" | Its reason is under the tiles. "StreamLens needs to see the computer's processes": that figure needs `pid: host`, which is optional ([the trade-off](#pid-host-a-trade-off)); without it "Graphics active" and the hardware line still tell. "The media server runs as a different user": the safer setup; the per-conversion figures need both as the same user (StreamLens as the media server's, with the PUID and PGID start), with [the trade-off](#pid-host-a-trade-off). "This system doesn't report how busy the video engine is": the system's Linux is too old for it (Synology, QNAP); the hardware line under the tiles still tells ([Graphics](#graphics-nvidia-amd-and-intel)). |
| "An NVIDIA card needs the NVIDIA Container Toolkit" | Install the toolkit on the computer and uncomment the `deploy:` block (`capabilities: [gpu, utility]`); on Unraid, the Nvidia Driver plugin and `--runtime=nvidia` ([Graphics](#graphics-nvidia-amd-and-intel)). |
| "ping blocked" for every viewer | On host networking: `sudo sysctl -w net.ipv4.ping_group_range="0 2147483647"` on the NAS. |
| "Too many failed attempts" on every device at once (Docker Desktop) | Every device counts as one there ([Docker Desktop](#docker-desktop-windows-and-macos)): wait the time it names (at most a minute), then sign in with the right password. |
| The log warns "can be read by other users of this computer" | StreamLens keeps its data folder private (only its own user may read it) and fixes that by itself when it owns the files. The warning names the file and the `chmod` to run when another user owns it. |
| The log says the data folder is one "Windows shares into Docker" | Docker Desktop: the folder's privacy is up to Windows. Keep it inside your user folder, or use a Docker volume ([Docker Desktop](#docker-desktop-windows-and-macos)). |
| "…_FILE names a file StreamLens can't read" | The key file must be readable by StreamLens's user: `sudo chown 1000:1000 secrets/<file>` (with the PUID and PGID start: your PUID and PGID instead of `1000:1000`). |
| Forgot the admin password | `docker exec -it -u 1000 streamlens node /app/tools/reset-password.js` (with the PUID and PGID start, use your PUID instead of 1000) |

## Security

The container runs as an ordinary user with every Linux privilege dropped. Its
system files are read-only, and it can't gain new privileges. It needs no
Docker socket and never writes outside its data folder. Inside that folder,
its settings, history, keys and logs are readable by StreamLens's own user
only (folders 0700, files 0600). Keys come from files mounted at start, never
from the image. Mount the media server's folders read-only, and never Plex's
whole configuration folder (its `Preferences.xml` holds Plex's own key): the
Logs folder is all StreamLens needs. Don't forward port 3737 to the internet;
remote access is its own feature.

`pid: host` is off in every compose file. Turned on with the media server
running as the same user, it undoes part of the above: StreamLens could then
read the media server's files through its processes (Plex's `Preferences.xml`
and its key included) and its environment, and stop its processes. Keep it off
unless you want the per-conversion graphics figures and accept that
([the trade-off](#pid-host-a-trade-off)).

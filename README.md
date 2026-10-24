# Wingman

Wingman is a private chat assistant for a team of about 10 to 50 people. You install it on a computer you control. The models run on that computer. Staff sign in with Microsoft Entra ID, on-premises Active Directory, or a local emergency account. Each directory group gets a shared folder that only that group can search.

The chat page is [Open WebUI](https://github.com/open-webui/open-webui). Wingman adds the installer, sign-in, group folders, document tools, backups, and the admin console around it.

This page is the install and operate guide for **1.0.0**. This repository holds only the installer download and this guide. After install, the full docs are on the server under `/opt/wingman/docs/`: `USER.md` (staff), `ADMIN.md` (day-to-day administration), `HANDOFF.md` (hardware notes and directory administrator work), and `requirements-matrix.md` (what is implemented, partial, or missing).

## What you get

These are in the 1.0.0 package. The wizard turns some of them on only when you ask.

**Chat and sign-in.** People chat in the browser. A password sign-in, including the emergency admin and Active Directory, also asks for a 6-digit authenticator code. Entra sign-in is kept only when Microsoft reports that MFA was used. The default session is a workday of 8 hours (10 or 12 are choices). An idle browser is signed out after 60 minutes (30 or 120 are choices). Every 15 minutes the server re-checks directory groups in the background. If the directory cannot be reached, the last known groups stay for a grace window (default 15 minutes) and that delay is written to the audit log. After the grace window, chat pauses and the person stays signed in.

**Models.** The installer reads CPU cores, memory, free disk, and any NVIDIA GPUs, then offers catalog models that fit. Ollama runs GGUF 4-bit weights and is the engine when there is no NVIDIA GPU. vLLM runs AWQ or GPTQ 4-bit weights when an NVIDIA GPU has compute capability 7.5 or newer. You can keep one engine or both, and switch later without deleting chats. The catalog is `qwen2.5` at 0.5B, 1.5B, 3B, 7B, 14B, 32B, and 72B, `qwen3:8b`, `llama3.1` at 8B and 70B, and the vision model `qwen2.5vl:7b`. `qwen3:8b` is Ollama only. `qwen2.5:0.5b` is the small model the automated installer test uses. Embeddings use `all-minilm` when the machine has under 12 GB of RAM and no GPU, and `nomic-embed-text` otherwise. `sudo wingman models test` checks a native tool call, a JSON reply, a markdown report, and a cited answer. An empty tool response fails the check. The check does not remove the model.

**Shared folders and knowledge sources.** Each group named in setup gets `/srv/wingman/shared/<group>`. Files dropped there are indexed for that group. The admin console can also sync a local folder, an SMB share, an NFS export, a SharePoint library, work or school OneDrive, WebDAV or Nextcloud, an S3-compatible bucket, or Google Drive. Credentials are encrypted. The sync is read-only. Removing a source deletes that source's index entries after you type its name. `sudo wingman sources sync` runs sources that are due.

**Office files, meetings, mail, and spreadsheets.** On a fresh install, office files, meeting notes, and spreadsheet analysis are turned on for each group chosen in the wizard. Mail stays off until an administrator enables it. An upgrade keeps a copilot policy that is already saved. Office files can be a Word, Excel, PowerPoint, or PDF document built from text the person's groups may read. Meeting notes take a recording you upload or a transcript you type. Speaker names are not detected. Spreadsheet questions run on this server. A comparison adds each group first (North 480 against South 460 in the office example, and month totals for the month example). A question that asks for an average uses the mean. Mail help can save a draft in the signed-in person's own work or school mailbox, or show a draft on this server. Wingman does not send the message.

**Skills and SkillSpector.** A skill is a saved way of asking. A person's own skill stays private until they submit it. SkillSpector 2.13.0 is installed with the product and scans the skill offline before anyone can approve it. There is no separate scanner install. A high-risk result is blocked. A medium result needs an administrator. Sharing with the whole organization, or a skill that uses tools, the code runner, web search, or an outside connection, needs an administrator. Community sharing and user-published tools and models stay off.

**Code runner.** The code runner is included. It runs under gVisor (`runsc`, platform `systrap`) on an internal Docker network with no outbound network. It is not given `NET_ADMIN` or `NET_RAW`. Only an administrator can use it until you grant a group:

```bash
sudo wingman grant Admissions code
```

**Connectors.** Microsoft 365 is optional and is off until you enable it. It is work or school accounts only, read-only unless you also opt in to drafts, and each person connects their own account. Personal Outlook.com is not supported. The admin console can also approve catalog connections for Monday.com, Atlassian, Slack, HubSpot, Salesforce (your own `salesforce.com` or `force.com` host), and Google Workspace products that are marked as a developer preview (Gmail, Drive, Docs, Sheets, Slides, Calendar, Chat, People, Classroom). A free-form server, `mcpo`, and a floating `@latest` address are refused. Tokens stay encrypted on this server. A connector is scanned before it can be approved.

**Documents.** Docling converts office documents for the index. A machine with a working NVIDIA GPU uses the larger CUDA image. A machine without one uses the CPU image.

**Backup and restore.** `sudo wingman backup` stores an encrypted restic repository at `/srv/wingman/restic` and deletes the temporary database dump. The password is `RESTIC_PASSWORD` in `/etc/wingman/env` and is not printed. A nightly backup runs at 02:30 and keeps 7 days, 4 weeks, and 12 months. Restore is the **Backups** page in the admin console.

**Admin console.** Administrators in `Wingman-Admins` open `http://<server>/admin` or `https://<your-domain>/admin`. The page asks for the password and the authenticator code. Buttons run through a small host agent. The page has no shell and no Docker socket. `Wingman-Auditors` and `Wingman-Helpdesk` are narrower roles. Older `LocalDesk-*` names still match. From the console or the command line an administrator can sign one person out, sign everyone out, or disable sign-in on this server without changing Active Directory or Entra. **Updates & Components** lists pinned components. Update and revert take a backup first and restore the previous pin if the health check fails. ClamAV signatures are the only update that runs by itself. Those update and revert actions have not been run against a real newer version than the pin recorded on the server. See [Testing](#testing).

**Antivirus, audit, and the rest.** ClamAV from the Ubuntu packages scans files before indexing, and scans office, meeting, and analysis uploads. Signature updates are automatic. The audit log is a hash-chained daily file. Alerts can go to the dashboard, an SMTP relay, or syslog. Messages cover a virus hit, failed sign-ins, a full disk, a service down, a certificate close to expiry, and a backup failure. Uptime Kuma is installed and listens on `127.0.0.1:3001`. Caddy serves chat on ports 80 and 443. Postgres stores the chat database. SearXNG (private web search), n8n (workflows), and Laya are off unless you enable them. n8n's license is for internal use. Laya is experimental and cannot attach a knowledge base outside the signed-in person's groups.

**Not in this version.** Meeting notes do not label speakers. Personal Outlook.com is refused. This page is not a compliance sign-off. The matrix lists the rest.

## System requirements

- **OS.** Ubuntu 24.04, or a derivative whose `ID_LIKE` contains `ubuntu` and whose Ubuntu codename is `noble`. Linux Mint 22 is that derivative. The check refuses anything else unless `WINGMAN_ALLOW_OTHER_OS=1` is set in `/etc/wingman/setup.env` before setup starts.
- **CPU architecture.** The package is `amd64`. The pinned gVisor archive is the x86_64 build.
- **Package dependencies.** `python3` 3.12 or newer, `ca-certificates`, `curl`, and `gnupg`. `sudo apt install` of the `.deb` pulls those in if they are missing.
- **Installed by setup, not by the `.deb`.** Docker Engine and the Compose plugin, gVisor `runsc` (release `20261012.0`, registered in Docker as `runsc` with `--platform=systrap`), ClamAV, and the firewall. You do not install gVisor yourself.
- **CPU, memory, and GPU.** There is no single minimum. The wizard measures the machine and will not offer a model that does not fit. With no NVIDIA GPU, stay on Ollama. vLLM is not installed, and Docling stays on the CPU image. vLLM needs an NVIDIA GPU of compute capability 7.5 or newer. Under about 12 GB of RAM and no GPU, embeddings use `all-minilm`.
- **Speed, as a planning estimate, not a benchmark.** On CPU only, a short reply from a model around 1.5B often takes several seconds. Around 3B, plan on about a minute. Around 7B to 8B, plan on one to three minutes. Larger than that, a short reply can take several minutes. A GPU is what makes a larger model feel conversational. The first reply after the model has been idle takes about a minute while it loads, including on a GPU.
- **Disk.** The wizard prints a download plan before the large pulls. Those figures are rounded, not measured registry sizes. The base images are about 8 GB, the CPU Docling image about 4 GB, the GPU Docling image about 10 GB, and the vLLM image about 12 GB when you select it. Model files are extra (a 7B Ollama model is about 4.7 GB in the catalog). A model is refused when free disk is less than the weight file plus 2 GB. Leave room beyond that plan for Docker and later downloads.
- **Network.** Root access, and a way to reach the server. Chat uses ports 80 and 443. The setup wizard listens only on `127.0.0.1:8790`.

The installer enables `ufw` unless you skip it. Incoming SSH is allowed only from `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`. Ports 80 and 443 are opened for a public certificate. The wizard port stays closed. If you SSH to this machine from a public address, write this file before you install, or the firewall step can drop your SSH session:

```bash
sudo install -d -m 0750 /etc/wingman
printf 'WINGMAN_SKIP_FIREWALL=1\n' | sudo tee /etc/wingman/setup.env >/dev/null
sudo chmod 0600 /etc/wingman/setup.env
```

## Install 1.0.0

The supported install is the `.deb` on the GitHub Release: [v1.0.0](https://github.com/MontarxDev/wingman-download/releases/tag/v1.0.0). No GitHub login is needed.

Download `wingman_1.0.0_amd64.deb` and `SHA256SUMS` into the same directory:

```bash
curl -LO https://github.com/MontarxDev/wingman-download/releases/download/v1.0.0/wingman_1.0.0_amd64.deb
curl -LO https://github.com/MontarxDev/wingman-download/releases/download/v1.0.0/SHA256SUMS
```

The published checksum is:

```text
3e65c405b0733cb48d6c8f89419337762feb15b52ed83a9f3230a3a87650f101  wingman_1.0.0_amd64.deb
```

Check it, then install. `sha256sum -c` looks for the `.deb` name written in `SHA256SUMS`, in the current directory:

```bash
sha256sum -c SHA256SUMS
sudo apt install ./wingman_1.0.0_amd64.deb
```

`sha256sum -c` prints `wingman_1.0.0_amd64.deb: OK`. If it does not, stop. Do not install that file.

The package does not open a browser. On a new machine it prints the setup URL and a one-time code, which looks like this:

```text
Open http://127.0.0.1:8790/setup
Setup code: <one-time code>
The wizard listens on localhost. From another computer, SSH-forward that port to 127.0.0.1.
The code is also in /etc/wingman/setup-code (root only).
```

The code file is mode 0600. If Wingman is already configured, the package does not start the wizard. It prints `Wingman is already configured. The setup wizard was not started.` and tells you to run `sudo wingman settings`.

From your laptop:

```bash
ssh -L 8790:127.0.0.1:8790 you@the-server
```

Then open `http://127.0.0.1:8790/setup` on the laptop and enter the code. To use another port, set `WINGMAN_SETUP_PORT` in `/etc/wingman/setup.env` before you install, then forward that port instead.

### First-run wizard

The wizard steps, in order, are Welcome, Hardware, Organization, Branding, Sign-in, Groups, Microsoft 365, Add-ons, Security, and Review.

1. **Hardware.** Read the memory, CPU, free disk, and GPU. Choose Ollama, vLLM, or both. A machine with no NVIDIA GPU stays on Ollama, and the vLLM choice is not installed. Pick how many people chat at once, and pick a model from the list that fits. The page shows a short reason for the suggestion.
2. **Organization.** Type the organization name and the emergency admin email. The password is generated and written to `/etc/wingman/admin-credentials`. It is not shown on the page and not printed. Read it on the server with `sudo` when you need it. The authenticator secret is `/etc/wingman/secrets/breakglass.totp`. Do not paste either file into chat, email, or a log.
3. **Branding.** Optional. The chat product keeps the Open WebUI name and logo unless you record a license exception (`under_50`, written permission, or an enterprise license). The institution logo is not placed beside the Open WebUI logo.
4. **Sign-in.** Microsoft Entra ID, on-premises Active Directory over LDAPS on port 636, or break-glass only until you decide. Entra needs a tenant id, client id, and client secret. LDAP needs a domain controller, a read-only bind account, and a search base. Choose the workday length, idle timeout, and directory grace window.
5. **Groups.** One folder per group under `/srv/wingman/shared/<group>`. Use the Entra display name or the Active Directory CN, for example `Admissions`.
6. **Microsoft 365.** Optional. Work or school accounts, read-only, each person connects as themselves. The drafts box adds `Mail.ReadWrite` and still does not send mail. Leave the group box blank to keep the tool off until you grant it.
7. **Add-ons.** SearXNG, n8n, and Laya are checkboxes and start off. Docling, the code runner, ClamAV, restic, and Uptime Kuma are part of a normal install.
8. **Security.** Alert destination, reboot policy, and the certificate source (public Let's Encrypt, a company CA, or a certificate signed by this server).
9. **Review.** If Docker volumes named `localdesk_*` or `wingman_*` already exist, the page says so. Leave the reset box unchecked to keep the database. Check it only when you mean to delete those volumes and start empty. Then click **Install**.

The install page shows the current step, an overall percent, and per-download progress. `/etc/wingman/install.log` keeps ordinary lines and errors, not the progress bars. When install finishes, the wizard leaves port 8790. The done page prints the chat address. Open `http://<server-ip>`. Sign in with the emergency admin email, the password from `/etc/wingman/admin-credentials`, and the 6-digit code from the secret in `/etc/wingman/secrets/breakglass.totp`. The model chosen on the Hardware step is downloaded during install. You do not run a separate model pull for that first model.

To see the ranking from the command line after the package is installed:

```bash
sudo wingman models recommend --offline
```

`--offline` uses the catalog shipped in the package. Without it, the command tries to refresh published sizes.

To skip the prompts, copy the example and edit it. The example sets `model` to `qwen2.5:0.5b`, which is the automated-test model, so change `model` to one `models recommend` lists before you install on an office machine:

```bash
sudo cp /opt/wingman/examples/answers.example.json /root/answers.json
sudo wingman install --config /root/answers.json
```

A hostname comes after install. Point a DNS A record at the server, leave ports 80 and 443 open, then:

```bash
sudo wingman domain assistant.example.edu
```

Chat is then `https://assistant.example.edu`. Caddy requests a certificate when the certificate source is the public Let's Encrypt option, which is the wizard default.

Explanations for the choices:

```bash
wingman help
wingman help vllm
wingman help groups
```

`wingman help` does not need root. `wingman --help` lists the commands. Every other command needs root and prints `Run as root: sudo wingman ...` if you forget.

## Day to day

Open the address from the done page. Staff instructions are `/opt/wingman/docs/USER.md` on the server.

```bash
sudo wingman status
sudo wingman logs open-webui
sudo wingman settings
```

`settings` opens `http://127.0.0.1:8790/settings` on the server. Forward that port the same way as setup. Changes made only in the Open WebUI settings panel are replaced from `/etc/wingman/env` on the next restart.

Drop files for a group into `/srv/wingman/shared/<group>`. `sudo wingman status` includes how many files are indexed. `sudo wingman sources sync` pulls remote sources that are due.

Enroll an authenticator for a local or LDAP account. The command prints the secret once:

```bash
sudo wingman mfa enroll person@example.edu
```

The code runner, web search, and Microsoft 365 start with no group grants. Models start out available to the groups you listed.

```bash
sudo wingman grant Admissions search
sudo wingman grant Admissions m365
sudo wingman group "Financial Aid"
```

Sign-in and sessions:

```bash
sudo wingman sessions list
sudo wingman sessions revoke person@example.edu --reason "Lost laptop"
sudo wingman sessions revoke-all --reason "End of term" --confirm "sign out everyone"
```

`sessions list` shows the email, device, IP address, and last seen time. It does not print the session token. Signing someone out refuses their next request. A chat socket the server is already relaying is checked about once a second while it is quiet, and closed when the session is revoked. **Sign out everyone** on the web page asks you to type `sign out everyone`. Disabling a person blocks sign-in on this server and does not change the directory account.

Another catalog model, after setup has finished:

```bash
sudo wingman models install qwen2.5:7b
sudo wingman models test
sudo wingman engine status
sudo wingman engine set ollama
```

`engine set` accepts `ollama`, `vllm`, or `both`. `models install` refuses a name that is not in the catalog, and it refuses to run before `/etc/wingman/env` exists.

### Upgrade

Installing this `.deb` over 1.0.0-rc6 or LocalDesk 0.2.4 keeps Docker volumes, chats, and `/etc/wingman`. The `localdesk` command still runs and prints that the command is now `wingman`. Docker volumes keep the name `localdesk`.

```bash
sha256sum -c SHA256SUMS
sudo apt install ./wingman_1.0.0_amd64.deb
```

A machine that is already configured does not restart the wizard. The live upgrade from LocalDesk 0.2.4 to this 1.0.0 package was not run again on the test machine. See [Testing](#testing).

Pinned components are updated from the admin page **Updates & Components**, or:

```bash
sudo wingman components
sudo wingman components check
sudo wingman update --check
sudo wingman components update open-webui --yes
sudo wingman components revert open-webui --yes
```

The web page asks you to type `UPDATE`, `UPDATE ALL`, or `REVERT`. `--yes` is that confirmation on the command line. Each update takes one restic backup and a database snapshot first. A failed health check or model self-test restores the pin you were leaving. Revert keeps three known-good pins. A failed backup does not pull a new image. `sudo wingman update` is the older whole-appliance update and also wants a backup first. These commands have not been exercised against a real newer version than the recorded pin.

ClamAV signatures update on their own. Nothing else does.

### Backup and restore

```bash
sudo wingman backup
```

That writes the encrypted repository at `/srv/wingman/restic` and removes the temporary SQL file. Read `RESTIC_PASSWORD` from `/etc/wingman/env` on the server when you need it. Do not print it.

Restore from the admin console: **Backups**, snapshot name `latest` unless you mean an older one, then type `RESTORE`. Chat stops for about 15 seconds. The site shows a maintenance page: "Chat is briefly unavailable. This usually takes about 15 seconds. Refresh this page." If the database load fails, the previous database is put back and chat is started again. A weekly check reads a snapshot and does not replace the live database.

### Uninstall

Stop the containers and keep the Docker volumes, shared folders, and configuration:

```bash
sudo wingman uninstall --yes
sudo apt remove wingman
```

`uninstall` without `--yes` refuses. `docker compose down` does not delete volumes.

Remove the program files, `/srv/wingman` (shared folders, the restic repository, and the audit log), and `/etc/wingman`, and still keep Docker volumes:

```bash
sudo wingman uninstall --purge --yes
sudo apt purge wingman
```

When `/etc/wingman/env` exists, purge copies it to `/var/lib/wingman/recovery/env` (mode 0600) before deleting `/etc/wingman`. A later install can reuse that password. The command tells you that `localdesk_*` and `wingman_*` volumes were not deleted. Delete them with `docker volume rm` only when you mean to erase chats and models. `sudo wingman install --reset-volumes` does that on a later install.

## Troubleshooting

**The checksum failed.** `sha256sum -c SHA256SUMS` must be run in the directory that contains `wingman_1.0.0_amd64.deb`. The expected hash is `3e65c405b0733cb48d6c8f89419337762feb15b52ed83a9f3230a3a87650f101`. Download both files again. Do not install a file that does not match.

**The download fails.** Use the links on the [v1.0.0 release page](https://github.com/MontarxDev/wingman-download/releases/tag/v1.0.0). No login is needed.

**The wizard did not start.** The package prints `The setup wizard did not start.` Read `/var/log/wingman-setup.log`. If the port is taken, setup stops with `That port is already in use` and names the port. Set `WINGMAN_SETUP_PORT` in `/etc/wingman/setup.env` to a free port and run `sudo systemctl restart wingman-setup.service`.

**The operating system was refused.** The message is `Wingman supports Ubuntu 24.04 and Ubuntu noble derivatives.` Set `WINGMAN_ALLOW_OTHER_OS=1` in `/etc/wingman/setup.env` only if you accept that this OS is outside what was tested.

**An NVIDIA card is present and `nvidia-smi` does not work.** Setup tries to install a driver and prints `If a driver was just installed, reboot and run Wingman setup again.` It continues on CPU for that attempt.

**Install stopped on a step.** The wizard names the step and stays on port 8790 with the text that the setup wizard is still waiting. After the answers file exists:

```bash
sudo wingman install --config /etc/wingman/answers.json
```

**You were locked out by SSH.** The firewall allows SSH only from private networks. Use the console, or reinstall after writing `WINGMAN_SKIP_FIREWALL=1` into `/etc/wingman/setup.env` as shown in [System requirements](#system-requirements).

**The first reply takes about a minute.** After the model has been idle, the first reply waits while the model loads. One failed request during that wait is tried again. Later replies in the same conversation are the normal speed for that machine.

**Chat shows a maintenance page for about 15 seconds.** That is a restore, or any 502, 503, or 504 from the chat service. The page asks you to refresh. During a restore this usually takes about 15 seconds.

**Sign-in says the authenticator failed.** Password accounts need the Wingman authenticator. Entra accounts need Microsoft MFA. The emergency secret is `/etc/wingman/secrets/breakglass.totp`. A new enrollment is `sudo wingman mfa enroll person@example.edu`.

**A command prints `Run as root`.** Use `sudo`. `wingman help` is the exception.

**Volumes from an older install have no saved password.** Install stops and explains. The wizard offers two choices: delete `localdesk_*` and `wingman_*` volumes, or type the previous database password and the previous admin password. It does not invent a password.

## Testing

Be exact about what this release has been run on.

**Tested on one test machine.** It is a Linux Mint 22.3 machine. The 0.2.4 install report for that machine recorded an NVIDIA RTX 3080 with 10 GB. The package installed and exercised there for this release is **1.0.0-rc9**, package sha256 `414ca92ad27c515e60c3d5733d22e09d7aaa2bd269c2c2198d909262f40a9865`. The report before that candidate was that rc8 already passed nearly everything on the test machine. rc9 is the build that fixed the remaining items from that machine, and that rc9 build is the one that passed there:

- Replacing an older SkillSpector virtualenv whose `python` is a symlink to `/usr/bin/python3`. Rebuilding it in place fails with "are the same file". The installer builds a new virtualenv in a temporary directory and swaps it into place.
- If an install step fails after it has stopped the code runner, that service is started again.
- Spreadsheet comparisons use group totals. The office question is North 480 against South 460. The month question uses month totals. A question that asks for an average uses the mean.
- A fresh install turns office files, meeting notes, and spreadsheet analysis on for each group chosen in the wizard, and leaves mail off. A policy that is already saved is kept.
- Restoring a backup shows the maintenance page instead of a 502.

1.0.0 is that rc9 program with the version string and the delivery docs set to 1.0.0. No other behavior was changed. The `.deb` hash changed because the version string and the packaged docs changed. It is `3e65c405b0733cb48d6c8f89419337762feb15b52ed83a9f3230a3a87650f101`.

**Not tested.**

- The final 1.0.0 `.deb` was not installed again. rc9 was the tested build.
- Component update and revert have not been run against a real newer version than the pin recorded on the server. The commands and the admin buttons are implemented and covered by tests that use a recorded digest.
- An upgrade from LocalDesk 0.2.4 to 1.0.0 was not run again on the test machine. The upgrade path is in the package. That live rerun was not part of this pass.
- Other distributions and other hardware were not part of this pass. Ubuntu 24.04 is the supported family. Linux Mint 22.3 on the test machine is the machine that was actually used.

Automated checks are separate from the test machine. For the rc9 commit, the CI checks job passed shellcheck, unit tests, purge and reinstall, browser sign-in, and the sandbox code runner. The artifact upload then failed because Actions storage quota was full. The release files are the download. The local suite at that commit was 277 tests, with 4 skipped. A browser sign-in through Caddy received a model reply. The end-to-end installer job was not used as the proof for this release. That job skips Docling, the code runner, Laya, vLLM, the firewall, the GPU, ClamAV, Uptime Kuma, and the large downloads, and it pins `qwen2.5:0.5b`.

## Further reading

After install, on the server under `/opt/wingman/docs/`:

- `QUICKSTART.md` — one-page install
- `USER.md` — staff
- `ADMIN.md` — directories, folders, Microsoft 365, backups
- `HANDOFF.md` — what to hand a colleague
- `requirements-matrix.md` — done, partial, and not done

The command list is `wingman --help`.

## License and use

Wingman's own code has no open-source license. The `.deb` is provided as-is, without warranty, for you to install and run on your own machines. Do not redistribute or resell it without permission from the author. Third-party components installed by setup (Open WebUI, Ollama, vLLM, Docling, ClamAV, n8n, and others) keep their own licenses.

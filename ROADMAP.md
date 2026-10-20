# Roadmap

## Backups

**Status: postponed.** No external storage is available yet. The NAS has a single hard drive, and a backup stored on that same drive would not survive its failure. This work will start as soon as an external drive (USB SSD or HDD) is available.

### Why it matters

The NAS runs on **a single HDD, without RAID**. If that drive fails, all data is lost at once: media, application configs and Nextcloud files.

RAID would not replace backups anyway: it protects against a drive failure, not against accidental deletion, corruption or ransomware.

### Data to back up

| Data | Location | Priority | If lost |
|---|---|---|---|
| Nextcloud files | `docker/data/nextcloud/data` | **Critical** | Permanent loss if the files exist nowhere else |
| Nextcloud Postgres database | `docker/data/nextcloud/db` | **Critical** | Shares, versions, accounts, calendars and contacts lost |
| Nextcloud `config.php` | `docker/data/nextcloud/config` | **Critical** | Holds the instance secrets: restore is impossible without it |
| Radarr, Sonarr, Prowlarr configs | `docker/data/{radarr,sonarr,prowlarr}/config` | High | Settings, indexers and history to redo by hand |
| Jellyfin config | `docker/data/jellyfin/config` | Medium | Accounts and watch history lost |
| qBittorrent, Seerr configs | `docker/data/{qbittorrent,jellyseerr}` | Low | Quick to reconfigure |
| Infrastructure definition | This git repository | — | Already backed up on GitHub |
| Traefik certificates (`acme.json`) | `docker/data/traefik/letsencrypt` | — | Regenerated automatically |
| Media | `media/` | Optional | Can be downloaded again, but large |

Secrets (`vault.yml`, the Vault password) are not in git and must also be kept outside the NAS, for example in a password manager.

### The Nextcloud case

A working restore requires a consistent copy of three things taken **at the same time**: the files, the database and `config.php`.

Copying Postgres files while it is running can produce a corrupted copy. The planned procedure is:

1. Enable maintenance mode: `occ maintenance:mode --on`
2. Export the database with `pg_dump`
3. Back up `data/`, `config/` and the database export
4. Disable maintenance mode: `occ maintenance:mode --off`

**Prerequisite:** the Postgres database currently lives in `docker/data/nextcloud/db`, inside the directory mounted in the Nextcloud container (`/var/www/html`). It must be moved to a separate directory (for example `docker/data/nextcloud-db`). This isolates the database from the web container and avoids copying it live along with the rest.

### Planned solution

- **Storage:** an external USB SSD or HDD. No USB flash drive: their memory wears out quickly under regular writes and can silently corrupt data.
- **Tool:** [restic](https://restic.net/). Each backup is a snapshot: a deletion or corruption on the NAS doesn't overwrite previous versions. Only changes are stored, and the repository is encrypted.
- **Frequency:** one backup per night, through a systemd timer.
- **Retention:** 7 daily, 4 weekly and 6 monthly backups (`restic forget --prune`).
- **Verification:** regular `restic check`, and **a full test restore** before considering the system reliable.

### Implementation

An Ansible `backup` role, consistent with the rest of the repository:

- install restic and initialize the repository on the external drive;
- restic repository password stored in Ansible Vault;
- backup script (maintenance mode, `pg_dump`, `restic backup`, retention);
- systemd timer for the nightly run;
- notification on failure.

### Next step: the 3-2-1 rule

Once local backups are in place, aim for the **3-2-1** rule: 3 copies of the data, on 2 different media, 1 of them off-site.

An always-connected drive shares the NAS risks (power surge, theft, fire, ransomware). The next step will be an off-site copy with restic, to low-cost storage such as Backblaze B2 or a Hetzner Storage Box.

### In the meantime

- **Monitor drive health (SMART)** and enable NAS alerts: a drive often shows signs of weakness before failing.
- **Don't store data in Nextcloud that exists nowhere else**, or keep it synced on another device through the Nextcloud client.

### Definition of done

- [ ] External drive available and connected to the NAS
- [ ] Postgres database moved out of the Nextcloud directory
- [ ] Ansible `backup` role deployed
- [ ] Nightly backup working, retention applied
- [ ] Successful test restore (Nextcloud and application configs)
- [ ] Off-site copy in place (3-2-1 rule)

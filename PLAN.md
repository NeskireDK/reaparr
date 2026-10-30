# Plan

This fork replaces reaparr. It keeps reaparr's dashboard and scheduled sweep, adds a second
deletion rule, a dry run and disk-level safety checks, and upgrades an existing reaparr install
automatically. It does not track upstream: changes from `Miista/reaparr` are taken by hand when
they are worth having.

The name is undecided. `Reclaim` is too close to two existing projects in the same niche,
`jessielw/Reclaimerr` and `Okhr/reclaimarr`. Until it is chosen, this document writes the env
var prefix as `APP_`.

## What decides what

- **Jellyfin says what was watched**: played flags and stop events, plus Playback Reporting
  history read through the plugin's API, never from its database file.
- **Inodes say what is safe**: a file still in a torrent's file list is never deleted, a
  leftover hardlink is removed so the space is actually freed, and untracked downloads are
  offered as orphans.
- **reaparr's UI and sweep, with every value configurable and a dry run in front of it.**

## Settings

Every setting can come from an env var or the dashboard; an env var wins and locks the field,
as reaparr does today.

### Deployment, env only

| setting | default | |
|---|---|---|
| data dir | `/app/data` | settings, state database, auth |
| listen port | `8767` | |
| data folder mount | none | needed only by the inode features; same path inside and out |

### Auth, reaparr's `authy`

`AUTH_DISABLED`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_BASE_URL`,
`LOCAL_ENABLED`, `ROLE_ADMIN_GROUP`, `ROLE_USER_GROUP`, `ROLES_DISABLED`. Forward-auth header
trust (`Remote-User`, `Remote-Groups`) is not carried over; an OIDC provider gives the same
admin and user split.

### Connections

| service | required | |
|---|---|---|
| Jellyfin | yes | URL and API key |
| Radarr | one of Radarr or Sonarr | more than one instance, each owning its libraries |
| Sonarr | one of Radarr or Sonarr | more than one instance, each owning its libraries |
| Seerr | no | stale-media cleanup |
| qBittorrent | no | URL, username, password; enables the seeding guard and orphans |

### Paths, only with the data folder mounted

Data folder, library folder, downloads folder, subtitle archive, the library folder as Jellyfin
sees it, and the data folder as qBittorrent and the arrs see it. Each is checked on setup:
inside the data folder, same filesystem, and a sample of known files actually found.

### Libraries

Name, folder, owning Radarr or Sonarr instance, profile, on or off.

### Profiles

A profile is the rule a library is judged by. Both are presets; every value can be changed.

| | `watched` | `stale` |
|---|---|---|
| deletable when | finished by a counted user | untouched for a long time |
| grace | `7d` after the last stop, minimum `1d`, movies and TV separate | `425d` without activity |
| minimum time in the library | none | `180d`, from the first arr import, latched |
| counts as watched | `Played=true` | a sitting of 50%+ of runtime, rows under `14d` apart, or `LastPlayedDate` |
| season | every episode aired, downloaded and played | 80%+ of episodes stale |
| counted users | Jellyfin users list, empty means everyone | same |

The latched age is the earliest import in Radarr or Sonarr history, stored once per TMDB, IMDB
or TVDB id because the arrs prune history. An item with no import history is never a `stale`
candidate.

### Automation and safety

| setting | default | |
|---|---|---|
| mode | `dry-run` | `off`, `dry-run` or `live`; replaces reaparr's `DAEMON_ENABLED`. Switching to `live` needs `I AM SURE` and a reason, both audited |
| schedule | `@daily` | cron |
| batch delay | `24h` | each sweep builds a batch that can be cancelled until it runs; `0` deletes at once |
| batch cap | `500 GB` | |

A dry run runs the real delete path and records every action instead of taking it. There is no
separate preview code to drift from it.

### Protection

The keep tag in Radarr and Sonarr, with who kept what. The arrs' `copyUsingHardlinks` setting
is checked every sweep, and a service with it off deletes nothing that sweep.

### Extra cleanup

| feature | default | needs |
|---|---|---|
| seeding guard | on | qBittorrent, data folder |
| remove leftover hardlink | on | data folder |
| orphans in downloads, with skip folders | on | qBittorrent, data folder |
| archive subtitles before deleting | on | data folder |
| Seerr stale-media cleanup | on | Seerr |
| remove ended series with no files | off | |
| Jellyfin rescan after a batch | on | |

### Notifications and logging

ntfy URL and token, log level, audit log.

## Migrating from reaparr

On first start, an install that has `/app/data/config.json` with `settings` and `connections`,
or any `REAPARR_SETTING_*` env var, is treated as a reaparr install. Setup shows what will be
imported and an admin confirms it.

| reaparr | becomes |
|---|---|
| connections and API keys | connections |
| `movies_grace_period`, `tv_grace_period` | a `watched` profile on every library |
| `jellyfin_users` | counted users |
| `poll_schedule` | schedule |
| `keep_tag`, `keepers` | keep tag and keep owners |
| `remove_ended_series` | the same setting |
| `daemon_enabled` | ignored; a migrated install always starts in `dry-run` |
| `REAPARR_SETTING_*` | read as aliases of the new env vars, with a deprecation warning |
| `/app/data/auth/` | kept as is |

`config.json` is only read, never written, so going back to reaparr still works. CI runs the
import against sample `config.json` files from each reaparr release.

## Prior art

| project | state | what it shows |
|---|---|---|
| `jessielw/Reclaimerr` | active, GPL-3.0, Python and Svelte | a rule engine already covering `arr.days_since_first_import`, `playback.days_since_last_activity` and `season.watched_percent`; Playback Reporting imported over the plugin's API; active-playback protection; no qBittorrent awareness |
| `Okhr/reclaimarr` | one day of work in 2025, no licence | deletes oldest-unwatched first until disk use falls under a target; watch history from Jellystat |

## Open questions

1. Can the `stale` rule be written in Reclaimerr's rule engine with the same candidates? If so,
   what this project adds over Reclaimerr is the inode side, and the scope should be weighed
   against that before building.
2. Which Playback Reporting API endpoint to use, and from which plugin version.
3. The state database for latched ages: SQLite beside `config.json`.
4. The default profile for a new install. `watched` keeps reaparr's behaviour for upgraders.
5. The name.

## Build order

1. Mode setting with dry run, around reaparr's existing sweep.
2. Migration from reaparr.
3. Profiles and the state database.
4. Playback Reporting over the API.
5. Seeding guard, leftover-hardlink removal and orphans.
6. Subtitle archive, more than one arr instance, ntfy, audit log.

# yadisk-download

Resumable mirror of Yandex.Disk folders to a local directory. Single-file Go tool, no dependencies.

Built for an unreliable storage backend: files that cannot be served are skipped quickly, everything else is downloaded, and a rerun fetches only what is still missing.

## Features

- Lists folders and downloads in parallel (streaming), no need to wait for the full listing.
- Resumable downloads (`.part` files, `Range` requests) with md5/sha256 verification.
- Rerun skips files that are already present and verified.
- Fail fast: a transfer with no data for 60 s is dropped; a file with no progress after 2 attempts is skipped.
- Two passes over the list; files still failing are reported and the run ends.
- Adaptive concurrency, up to 25 slots: a new slot is added every 5 s while no data is flowing.
- Free-space guard (keeps 2 GiB free), single instance per state directory (lock file).
- Refresh-token rotation is saved to the state directory.

Result of a run:

- `ALL DONE` and a `complete` marker in the state directory: every file is verified.
- `GAVE UP` with the list of missing files, exit code 1: run again later to retry them.

## Build

OAuth credentials are baked into the binary at build time. Put them in an env file (git-ignored `.env` next to `build.sh` by default):

```
YADISK_CLIENT_ID=...
YADISK_CLIENT_SECRET=...
YADISK_REFRESH_TOKEN=...
```

```
bash build.sh [path/to/env-file]
GOOS=darwin GOARCH=arm64 bash build.sh   # other target; default is linux/amd64
```

Output: `bin/yadisk-download`. The binary contains the credentials, do not share it.

## Usage

```
yadisk-download [flags] disk:/folder1 [disk:/folder2 ...]
```

| Flag | Meaning |
|---|---|
| `-dest` | local folder; each Disk folder lands in `<dest>/<its name>` (default: `downloads` next to the binary) |
| `-state` | folder for `refresh-token`, `download.log`, `lock`, `complete` (default: `.state` next to the binary) |
| `-exclude` | full Disk path to skip, repeatable (e.g. `disk:/folder1/cache`) |
| `-tree` | do not download: print subfolder names of this Disk folder, then exit |
| `-depth` | with `-tree`: how many folder levels to print (default 2) |

Example:

```
yadisk-download -dest /data/mirror -state /data/mirror-state \
  -exclude disk:/folder1/cache \
  disk:/folder1 disk:/folder2
```

For a long run on a remote machine, start it detached (`setsid -f ... </dev/null >/dev/null 2>&1`) and read `download.log` in the state directory to follow progress.

## Tests

```
go test ./...
```

Tests run against a fake Yandex.Disk server.

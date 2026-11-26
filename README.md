# Popamp apt repository

Builds and mirrors <https://apt.popamp.app>, the apt repository for
Popamp, its skins and its plug-ins.

```
release in popamp, skin-*, plugin-* (.deb assets)
   │  gh workflow run index.yml -R popamp/apt   (and once a day)
   ▼
Index workflow: bin/build-index ─► signs with APT_SIGNING_KEY
   ─► release "index": apt-index.tar.gz (dists/ + pool.txt), pool.txt
   │
   ▼  server, cron: bin/mirror (no secrets)
checks InRelease against site/popamp.gpg ─► downloads new .debs from
GitHub ─► checks each against the signed index ─► switches dists/
```

The index lists the `.deb` files of the latest release of every public
repository of popamp, so a new skin needs no change here: it publishes
a release, and the index picks it up. The packages stay on GitHub and are
never uploaded again; only the index, a few kilobytes, is rebuilt. Two
repositories may not ship the same package for the same architecture.

One suite, `stable`, component `main`, for amd64 and arm64 (and `all`,
for skins and plug-ins). Indexes are plain and `.xz`, never `.gz`:
HestiaCP's nginx and Cloudflare cache `*.gz` for a long time.

## Files

| Path | What |
| --- | --- |
| `bin/build-index` | builds and signs the index; run by CI |
| `bin/mirror` | mirrors the index and its packages into a web root; for cron |
| `.github/workflows/index.yml` | the Index workflow |
| `bin/build-addon-deb` | builds the `.deb` of a skin or plug-in repository |
| `.github/workflows/addon-release.yml` | release workflow that skin and plug-in repositories call |
| `bin/test-install`, `.github/workflows/test-install.yml` | installs from apt.popamp.app on Debian 12, 13 and Ubuntu 22.04, 24.04, 26.04, amd64 and arm64, daily |
| `bin/build-flatpak-repo`, `.github/workflows/flatpak.yml` | builds Popamp Plus into the signed Flatpak repository; the Flatpak workflow |
| `bin/mirror-flatpak` | mirrors that repository into `<web root>/flatpak`; `bin/mirror` runs it |
| `bin/import-signing-key` | imports `APT_SIGNING_KEY` in a CI job |
| `site/` | copied into the web root on every mirror run: the public key, the sources file, `popamp.flatpakref` and `popamp.flatpakrepo` |

## Signing key

The secret part exists only in the repository secret `APT_SIGNING_KEY`
(armored, no passphrase) and in an offline backup. `site/popamp.gpg` is
its public part; the Index workflow refuses a secret that does not match
it, and `bin/mirror` accepts only indexes it signed. The same key signs
the Flatpak repository; `site/popamp.flatpakref` and `popamp.flatpakrepo`
carry its public part, and Flatpak checks every download against it.

## Skins and plug-ins

A repository `skin-<name>` becomes the package `popamp-skin-<name>`, its
files in `/usr/share/popamp/skins/<name>`; `plugin-<id>` becomes
`popamp-plugin-<id>` in `/usr/share/popamp/plugins/<id>`. Both are
`Architecture: all`. A skin depends on `popamp (>> 0.5.0)`, a plug-in on
`popamp-plus (>> 0.5.0)`: 0.5.0 was the last Popamp that read only the
user's folders, and popamp-plus provides `popamp (= its version)`. An
add-on that needs a newer player says so with the workflow's `min-popamp`
and depends on `(>= that version)` instead.
A plug-in stays off until the user allows it, packaged or not.
The package's description is the README paragraph just above its first
picture. Its copyright file says MIT or Apache-2.0 by the Popamp
contributors, unless the repository has a `COPYRIGHT` file of its own in
the Debian copyright format, as a skin ported from other authors' artwork
does.

Such a repository has a `.github/workflows/release.yml` that calls
`addon-release.yml` (the caller is in its header). Pushing a tag
`v<version>` builds the package (a plug-in's version must match its
`plugin.toml`), publishes it in that release and starts the Index
workflow; a manual run only builds it. The repository needs the secret
`APT_INDEX_TOKEN`, below.

## A release elsewhere starts the Index workflow

`addon-release.yml` does this for skins and plug-ins. Any other
repository whose releases carry `.deb` files (popamp itself) ends its
release job with:

```yaml
      - name: Rebuild the apt index
        env:
          GH_TOKEN: ${{ secrets.APT_INDEX_TOKEN }}
        run: |
          if [ -z "$GH_TOKEN" ]; then
            echo "::warning::APT_INDEX_TOKEN is not set; the daily Index run will pick this release up"
            exit 0
          fi
          gh workflow run index.yml -R popamp/apt --ref main
```

`APT_INDEX_TOKEN` is a fine-grained token for popamp/apt only, with
Actions: read and write. popamp is a user, not an organisation, so there
are no shared secrets: set it in each repository, e.g.

```sh
gh secret set APT_INDEX_TOKEN -R popamp/skin-steel
```

## Flatpak

<https://apt.popamp.app/flatpak/> is an OSTree repository with one app,
`io.github.popamp.popamp` (Popamp Plus, x86_64), on the branch `stable`.
It is not on Flathub: Flathub takes no manifest written with an AI.
The runtime and its GL extension come from Flathub, which
`popamp.flatpakref` names.

```
release in popamp (with packaging/flatpak/)
   │  gh workflow run flatpak.yml -R popamp/apt   (and once a day)
   ▼
Flatpak workflow: the published repository ─► flatpak-builder at the
   release's tag, signed ─► prune to it and the one before, with a static
   delta ─► install from it, as a check ─► release "flatpak":
   flatpak-repo.tar, built.txt
   │
   ▼  server, cron: bin/mirror ─► bin/mirror-flatpak (no secrets)
built.txt changed ─► downloads the tar ─► switches flatpak/
```

A manual run takes a `ref` of popamp/popamp (a branch or a commit) to
build in place of its latest release. A release that has no manifest is
skipped. The release job of popamp starts the Flatpak workflow with the
same `APT_INDEX_TOKEN` as the Index workflow.

## Server (HestiaCP)

The web domain `apt.popamp.app` needs nothing special: it is all static
files, and none of them has an extension HestiaCP caches for long (the
Flatpak repository's `summaries/*.gz` are named by their hash).

```sh
git clone https://github.com/popamp/apt ~/apt
~/apt/bin/mirror ~/web/apt.popamp.app/public_html
```

Then, as root: `v-add-cron-job Popamp '*/10' '*' '*' '*' '*' '/home/Popamp/apt/bin/mirror /home/Popamp/web/apt.popamp.app/public_html'`.
A run that finds nothing new prints nothing. A change to `site/` reaches
the server with `git -C ~/apt pull`.

## For users

```sh
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSLo /etc/apt/keyrings/popamp.gpg https://apt.popamp.app/popamp.gpg
sudo curl -fsSLo /etc/apt/sources.list.d/popamp.sources https://apt.popamp.app/popamp.sources
sudo apt update
sudo apt install popamp           # or popamp-plus
```

Flatpak:

```sh
flatpak install --user https://apt.popamp.app/popamp.flatpakref
```

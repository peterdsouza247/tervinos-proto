## Prerequisites

- A GitHub account (the build runs entirely in GitHub Actions)
- A Fedora Kinoite VM or spare machine for testing (VirtualBox works)
- Optionally, any Linux shell with `podman` for generating ISOs

## Step 1: Create your repo from the template

1. Open https://github.com/blue-build/template
2. Click **Use this template → Create a new repository**, name it
   `tervinos`, make it public.
3. Enable Actions: repo → **Actions** tab → enable workflows.

## Step 2: Set up image signing (one-time)

Images are cryptographically signed so users can trust updates.

```bash
# in any terminal with cosign, or use the Codespace from tervin-shell
cosign generate-key-pair
```

- Commit `cosign.pub` to the repo root (replace the placeholder file).
- Add the private key as a repo secret: **Settings → Secrets and
  variables → Actions → New repository secret**, name `SIGNING_SECRET`,
  paste the contents of `cosign.key`. Never commit `cosign.key`.

## Step 3: Write the tervinOS recipe

Edit `recipes/recipe.yml`:

```yaml
name: tervinos
description: Linux, distilled to what actually matters.
base-image: ghcr.io/ublue-os/kinoite-main
image-version: latest

modules:
  # System packages layered into the image
  - type: rpm-ostree
    install:
      - ghostwriter        # markdown notepad (Tervin Notes stand-in)
      - wmctrl             # window capture for Tervin Shell
      - power-profiles-daemon
      - distrobox
    remove: []

  # Default apps every user gets, installed on first boot
  - type: default-flatpaks
    notify: true
    system:
      install:
        - app.zen_browser.zen          # Zen Browser
        - com.valvesoftware.Steam      # Steam
        - com.github.tchx84.Flatseal   # permission manager (Privacy)
        - org.kde.filelight            # storage visualizer

  # Ship your own files into the image (branding, Tervin Shell, configs)
  - type: files
    files:
      - source: system
        destination: /   # copies repo's files/system/* into the image

  # Enable services
  - type: systemd
    system:
      enabled:
        - power-profiles-daemon.service

  - type: signing
```

Notes:

- `kinoite-main` is Universal Blue's Fedora Atomic KDE base with codecs
  and drivers already fixed up. For the Gaming edition, base on
  `ghcr.io/ublue-os/bazzite` instead and Steam/Proton tuning comes
  built in.
- Steam as a Flatpak keeps the base image lean; Bazzite users get the
  native, deeply tuned version.

## Step 4: Add tervinOS branding and Tervin Shell

Anything under `files/system/` in the repo lands in the image:

```
files/system/
  usr/share/tervin/                     # Tervin Shell app files
  usr/share/applications/tervin-shell.desktop
  usr/share/wallpapers/tervinos/        # default wallpaper
  etc/skel/.config/tervin-shell/intents.yaml   # default intents per user
```

A minimal `tervin-shell.desktop` launcher:

```ini
[Desktop Entry]
Name=Tervin Shell
Exec=python3 /usr/share/tervin/main.py
Icon=applications-system
Type=Application
Categories=System;
```

Add `python3-pyqt6`, `python3-psutil`, and `python3-pyyaml` to the
`rpm-ostree` install list so Tervin Shell runs out of the box.

## Step 5: Push and let GitHub build it

```bash
git add .
git commit -m "tervinOS v0.1"
git push
```

The included workflow builds the image and publishes it to
`ghcr.io/<your-username>/tervinos`. Check progress in the **Actions**
tab. Every future push rebuilds; a scheduled workflow also rebuilds
regularly so users keep getting upstream Fedora security fixes.

## Step 6: Test it on a real system

Install stock **Fedora Kinoite** in a VM, then switch it to tervinOS:

```bash
# first rebase, unverified, to fetch the image
rpm-ostree rebase ostree-unverified-registry:ghcr.io/<your-username>/tervinos:latest
systemctl reboot

# second rebase, now signed and verified
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/<your-username>/tervinos:latest
systemctl reboot
```

You are now running tervinOS. Updates arrive automatically. Rollback at
any time with `rpm-ostree rollback` (the "Everything is Undoable" pillar,
for real).

## Step 7: Generate an installable ISO

For the website's Download buttons. On any Linux shell with podman:

```bash
sudo podman run --rm --privileged --pull=always \
  -v ./output:/build-container-installer/build \
  ghcr.io/jasonn3/build-container-installer:latest \
  IMAGE_REPO=ghcr.io/<your-username> \
  IMAGE_NAME=tervinos \
  IMAGE_TAG=latest \
  VARIANT=Kinoite
```

Or with the BlueBuild CLI: `bluebuild generate-iso image
ghcr.io/<your-username>/tervinos`. The resulting ISO is an offline
installer you can host as a GitHub Release and link from the site.

## Step 8: Iterate toward the feature list

| Site feature | How to deliver it |
|---|---|
| Immutable system, transactional updates | Free with the Fedora Atomic base |
| Snapshot time machine | Base is ostree-versioned already; add `snapper` for user data |
| Universal search | KRunner + Baloo, on by default in Kinoite |
| Native containers | Add `distrobox` (done in recipe above) |
| Privacy dashboard | Ship Flatseal + Plasma Firewall in the recipe |
| Dynamic power profiles | power-profiles-daemon (done above) |
| Intent Workspaces, Human Time, Resume Card, Why? | Ship Tervin Shell in the image (Step 4) |

## Suggested edition mapping

| Edition | Base image |
|---|---|
| Desktop | `ghcr.io/ublue-os/kinoite-main` |
| Minimal | `ghcr.io/ublue-os/kinoite-main`, drop Steam flatpak |
| Gaming | `ghcr.io/ublue-os/bazzite` |

One repo can build all three: put one recipe per edition in `recipes/`
and list them in the GitHub workflow matrix.

## References

- BlueBuild docs: https://blue-build.org/
- Template repo: https://github.com/blue-build/template
- ISO generation: https://blue-build.org/how-to/generate-iso/
- Building on Universal Blue: https://blue-build.org/learn/universal-blue/

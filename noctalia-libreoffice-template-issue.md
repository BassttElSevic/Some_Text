# libreoffice: META-INF/manifest.xml is not distributed, so apply.sh aborts before building the .oxt

> Issue draft for `noctalia-dev/community-templates`.
> The heading above goes in the title field; the body starts below the rule.
> Suggested labels: bug, template:libreoffice

---

The `libreoffice` template never applies. Two independent things are wrong; the first one alone is enough to break it for everyone, the second only affects installations where `unopkg` is not on `PATH`. I've kept both in one issue because they are on the same code path and the second is only reachable once the first is fixed, but I'm happy to split them.

## Environment

- Noctalia v5.0.0
- Templates as served by `api.noctalia.dev/templates` on 2026-08-31; repo HEAD at the time was `d740773`
- Kali Linux Rolling 2026.3, niri 26.04, Wayland session
- LibreOffice 26.2.4.2, installed from the official TDF `.deb` packages into `/opt/libreoffice26.2`
- python3 3.14.6, zip 3.0

## Problem 1: files in template subdirectories are not distributed

`libreoffice/META-INF/manifest.xml` exists in this repository but is not present in the template as delivered to clients. `apply.sh` copies it unconditionally under `set -euo pipefail`, so the hook exits before it can build or install the extension.

### Evidence

The repository has 10 files in `libreoffice/`:

```console
$ find libreoffice -type f | sort
libreoffice/META-INF/manifest.xml
libreoffice/Paths.xcu
libreoffice/README.md
libreoffice/Theme_Colors.xcu
libreoffice/apply.sh
libreoffice/description.xml
libreoffice/light-screenshot.png
libreoffice/pkg-description.en
libreoffice/screenshot.png
libreoffice/template.toml
```

The API returns 9 for the same template, without `META-INF/manifest.xml`:

```console
$ curl -s https://api.noctalia.dev/templates | <filter to name == "libreoffice">
apply.sh
description.xml
light-screenshot.png
Paths.xcu
pkg-description.en
README.md
screenshot.png
template.toml
Theme_Colors.xcu
9
```

The client-side cache agrees — `~/.local/state/noctalia/community-templates/libreoffice/.noctalia-cache.json` lists the same 9 entries, so this is not a download or extraction failure on my machine; the file is absent from what the API advertises.

No entry anywhere in the catalog has a path separator in its name:

```text
files containing "/": 0 (out of 212 total files across all templates)
```

And `libreoffice` is the only template in the repo that ships a nested file at all:

```console
$ find . -mindepth 3 -type f -not -path "./.git/*" -not -path "./.github/*"
./libreoffice/META-INF/manifest.xml
```

That is consistent with the catalog builder enumerating only the top level of each template directory, and it would explain why this hasn't come up before: no other template depends on a nested file, so nothing else loses anything.

Worth noting that CI does recurse — `validate-templates.py:237` uses `manifest_path.parent.rglob("*")` — so the file is visible to validation and the template passes. The mismatch is only between the repository layout and what gets distributed.

### Effect

Relevant lines in `libreoffice/apply.sh`:

```sh
6:   set -euo pipefail
19:  mkdir -p "$BUILD_DIR/pkg/META-INF"
44:  cp "$CONFIG_DIR/META-INF/manifest.xml" "$BUILD_DIR/pkg/META-INF/manifest.xml"
47:  (cd "$BUILD_DIR/pkg" && zip -qr "$OXT_PATH" .)
```

Line 19 creates the directory, line 44 fails, and `set -e` ends the script there. Line 47 never runs, so no `.oxt` is ever produced and the `unopkg` block below it is never reached.

Reproduced in an isolated copy of the template with `META-INF/` removed:

```console
$ bash <template>/apply.sh
cp: cannot stat '<template>/META-INF/manifest.xml': No such file or directory
exit code: 1

$ find <template>/build
<template>/build
<template>/build/pkg
<template>/build/pkg/pkg-description.en
<template>/build/pkg/description.xml
<template>/build/pkg/Paths.xcu
<template>/build/pkg/Theme_Colors.xcu
<template>/build/pkg/META-INF          # empty

$ ls <template>/build/*.oxt
ls: cannot access '<template>/build/*.oxt': No such file or directory
```

That empty `META-INF/` plus a missing `.oxt` is exactly the state I found on my real installation before touching anything, and `unopkg list` returned nothing.

From a user's point of view there is no indication of failure. The rendered file at `$XDG_STATE_HOME/noctalia/libreoffice-theme-staging/Theme_Colors.xcu` is written normally and its mtime updates on every theme change, so the template looks like it is working. I could not find the `cp` error surfaced anywhere — nothing in `~/.cache/noctalia/noctalia.log` or `noctalia.log.1` matched `libreoffice`, `unopkg`, `apply.sh`, or `META-INF`.

### Steps to reproduce

1. Enable the `libreoffice` template in Settings -> Templates.
2. Change the theme (or re-apply the current one).
3. `ls ~/.local/state/noctalia/community-templates/libreoffice/META-INF` — does not exist.
4. `ls ~/.local/state/noctalia/community-templates/libreoffice/build/*.oxt` — does not exist.
5. `unopkg list` — `dev.noctalia.libreoffice.theme` is not installed.
6. Start LibreOffice — no `Noctalia` entry under Tools -> Options -> Application Colors -> Scheme.

### Suggested fix

Either of these would work:

1. Make the catalog builder recurse into template subdirectories, preserving relative paths, and have the client create parent directories when writing files. This keeps the template as it is.
2. If nested files are deliberately out of scope, drop the dependency on one. `manifest.xml` is static, so `apply.sh` can write it directly:

```sh
cat > "$BUILD_DIR/pkg/META-INF/manifest.xml" <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<manifest:manifest xmlns:manifest="http://openoffice.org/2001/manifest">
  <manifest:file-entry manifest:full-path="Theme_Colors.xcu" manifest:media-type="application/vnd.sun.star.configuration-data"/>
  <manifest:file-entry manifest:full-path="Paths.xcu" manifest:media-type="application/vnd.sun.star.configuration-data"/>
</manifest:manifest>
EOF
```

and `libreoffice/META-INF/` can then be removed from the repo. This needs no infrastructure change and is the smaller fix.

If nested paths are meant to be unsupported, it would also be worth having CI reject them, since at the moment a template can pass validation and still ship incomplete.

I don't know whether the catalog builder lives in this repo or on the Noctalia/API side, so this may belong in `noctalia-dev/noctalia` instead — move it if so.

## Problem 2: `command -v unopkg` does not find official TDF `.deb` installations

`apply.sh` locates LibreOffice like this:

```sh
80:  if command -v unopkg >/dev/null 2>&1; then
89:  if flatpak info org.libreoffice.LibreOffice >/dev/null 2>&1; then
99:  echo "noctalia libreoffice: no LibreOffice installation found (neither native unopkg on PATH nor the org.libreoffice.LibreOffice Flatpak)" >&2
```

The official TDF `.deb` packages (the ones from the LibreOffice download page, as opposed to a distro's `libreoffice` package) install into `/opt/libreoffice<major>.<minor>/` and put only a version-suffixed launcher on `PATH`. There is no unversioned `libreoffice`, no `soffice`, and no `unopkg`:

```console
$ env PATH="/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin" \
    bash -c 'for c in unopkg libreoffice soffice libreoffice26.2; do
               printf "%-16s %s\n" "$c" "$(command -v $c || echo "(not found)")"; done'
unopkg           (not found)
libreoffice      (not found)
soffice          (not found)
libreoffice26.2  /usr/local/bin/libreoffice26.2
```

`/usr/local/bin/libreoffice26.2` is a symlink to `/opt/libreoffice26.2/program/soffice`. `unopkg` is present at `/opt/libreoffice26.2/program/unopkg` but nothing links it onto `PATH`. The package names differ too (`libobasis26.2-core`, `libreoffice26.2-*` rather than `libreoffice-core`), which rules out detecting the install via dpkg under the usual names.

With no Flatpak LibreOffice installed:

```console
$ flatpak info org.libreoffice.LibreOffice
error: org.libreoffice.LibreOffice/*unspecified*/*unspecified* not installed
```

both branches fail and line 99 reports no installation found, on a machine where LibreOffice is installed and running.

The template README already flags this path as untested — under "Native install": "Not verified live on this machine, only Flatpak LibreOffice is installed here." So this is a gap rather than a regression.

### Suggested fix

Fall back to the known layout when `unopkg` is not on `PATH`:

```sh
UNOPKG="$(command -v unopkg || true)"
if [ -z "$UNOPKG" ]; then
  for candidate in /opt/libreoffice*/program/unopkg \
                   /usr/lib/libreoffice/program/unopkg \
                   /usr/lib64/libreoffice/program/unopkg; do
    [ -x "$candidate" ] && UNOPKG="$candidate" && break
  done
fi
```

then use `"$UNOPKG"` in place of `unopkg` in the remove/add calls, keeping the existing "nothing found" message for when it stays empty. Distro packages do put `unopkg` on `PATH`, so this only adds coverage for the official `.deb`/`.rpm` layout and changes nothing for existing users.

## Verification that the rest of the template is fine

With both problems worked around locally — `manifest.xml` recreated by hand from the repo copy, `unopkg` symlinked into `~/.local/bin` — `apply.sh` completes and installs correctly:

```console
$ bash ~/.local/state/noctalia/community-templates/libreoffice/apply.sh; echo "exit: $?"
exit: 0

$ unzip -l .../build/noctalia-theme.oxt
      416  Paths.xcu
      681  description.xml
        0  META-INF/
      430  META-INF/manifest.xml
     6360  Theme_Colors.xcu
      192  pkg-description.en

$ unopkg list
Identifier: dev.noctalia.libreoffice.theme
  Version: 1.0.0
  is registered: yes
  bundled Packages: {
      ...Theme_Colors.xcu   is registered: yes
      ...Paths.xcu          is registered: yes
  }
```

The hex-to-decimal conversion is correct: I compared every one of the 42 `Color` values in the built `Theme_Colors.xcu` against `int(hex, 16)` of the corresponding value in the rendered staging file, and all 42 match. After selecting the scheme, LibreOffice picks up the palette as expected.

One note for anyone else verifying this: grepping the built file for leftover hex with something like `<value>[0-9a-fA-F]{6}</value>` reports one match, which looks like a conversion miss but isn't. `BASICKeyword` is `0b57d0`, which converts to `743376` — six digits, all of them also valid hex characters. Comparing against the converted source value rather than pattern-matching avoids the false positive.

So once the manifest is present and `unopkg` is found, the template works as designed on a native install.

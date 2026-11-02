# runelite-dev-build

Standalone build pipeline for a **RuneLite dev client** — a patched launcher that runs a
**bundled `client.jar`** and supports **developer mode**, packaged into a Windows installer.

## What it produces

`RuneLiteSetup.exe` — an installer that installs RuneLite with the bundled client and the
dev-mode-capable launcher.

## How the workflow builds it

`.github/workflows/build.yml` runs on `windows-2022`:

1. Checks out **`runelite/runelite`** and builds `:client:shadowJar` → the client shaded jar.
2. Checks out a **pinned `runelite/launcher`** commit and applies
   `patches/launcher-custom.patch` (bundled client + developer mode + configurable
   plugin-hub version), then builds the launcher jar.
3. Builds the native launcher + Detours DLL (MSVC), downloads the JRE, assembles
   `build/win-x64`, and compiles the installer with Inno Setup (`build-win64.sh`).
4. Uploads **`RuneLiteSetup.exe`** as an artifact.

> **Upstream-only:** the build uses only `runelite/runelite` and `runelite/launcher` (pinned) plus
> the local patch — it does **not** depend on any personal fork. To move to a newer upstream
> launcher, regenerate the patch and bump `launcher_ref` (see "Keeping up to date" below).

## Run a build

- Push to `main`, **or**
- Actions → **build** → *Run workflow* (optionally override `client_ref` / `launcher_ref`).

Then download the **RuneLiteSetup** artifact from the run and execute it.

## Enable developer mode after installing

Open the launcher config — Start Menu → **RuneLite (configure)** (or `RuneLite.exe --configure`) —
set **Client arguments** to `--developer-mode`, then click **Save**. That writes:

```json
{ "clientArguments": ["--developer-mode"] }
```

Developer mode requires assertions. **The patched launcher adds `-ea` automatically** when it sees
`--developer-mode`, so you don't need to set JVM arguments by hand (setting `-ea` explicitly under
**JVM arguments** is harmless — the launcher won't add a duplicate).

> `-ea` only takes effect in a forked JVM, so keep **Launch mode** on `AUTO` (FORK/JVM).
> Do **not** use `REFLECT` — it runs the client in the launcher's own JVM and ignores JVM args.

Alternatively set `RUNELITE_ARGS=--developer-mode` (the launcher still auto-adds `-ea`).

## The patch

`patches/launcher-custom.patch` changes only launcher source:

| File | Change |
|---|---|
| `Launcher.java` | run a bundled `client.jar`; forward `--developer-mode` and suppress the launcher-version stream so the client enables dev mode; add `--pluginhub-version` |
| `LauncherSettings.java` | plugin-hub version setting |
| `ConfigurationFrame.java` | plugin-hub version GUI field |
| `innosetup/runelite.iss` | ship `client.jar` in the installer |
| `build-win64.sh` | copy the client shaded jar into `build/win-x64` |

The plugin-hub version is `DEFAULT_PLUGINHUB_VERSION` in `Launcher.java` (fallback `1.13.5`).
The client fetches `…/plugins/manifest/<ver>_lite.js`. **During CI this is auto-resolved** to the
newest released RuneLite that actually has a published manifest (see the "Resolve & set default
plugin-hub version" step), so you don't need to bump it by hand. Override it with the
`pluginhub_version` workflow input if needed.

## Keeping up to date with upstream

The launcher is pinned to a tested upstream commit (`launcher_ref`). When you want to move to
a newer upstream launcher:

1. Rebase the patch onto the new upstream commit (resolve any conflicts in `Launcher.java`).
2. Regenerate and commit the patch:
   ```bash
   git clone https://github.com/runelite/launcher && cd launcher
   git checkout <new-upstream-commit>
   git apply /path/to/launcher-custom.patch
   # fix conflicts, then regenerate:
   git diff <base-against-which-you-patched> -- \
     src/main/java/net/runelite/launcher/Launcher.java \
     src/main/java/net/runelite/launcher/LauncherSettings.java \
     src/main/java/net/runelite/launcher/ConfigurationFrame.java \
     innosetup/runelite.iss build-win64.sh > launcher-custom.patch
   ```
3. Update the default `launcher_ref` in the workflow to the new commit.

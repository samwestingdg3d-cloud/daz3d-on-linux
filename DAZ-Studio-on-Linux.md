# Running Daz Studio on Linux with GE-Proton (including GPU Iray rendering)

This is a write-up of how I got **Daz Studio 6** and the **Daz Install Manager (DIM)** running on Linux with Proton. Iray renders on the GPU with CUDA and OptiX, the same as it does on Windows. The guide covers each problem I hit, how I found it, and the fix, so you can adapt it to your own setup.

> **TL;DR:** Use `umu-run` with GE-Proton, not plain Wine, and set **`PROTON_NVIDIA_LIBS=1`** or Iray falls back to CPU-only.
> GE-Proton11-5 also needs a one-line symlink workaround for that setting to work (see [Problem 3](#problem-3-proton-crashes-with-nvidia-libsnvapinvapi64dll-not-found)).
> The full launcher script is at the [end of this guide](#the-complete-launcher-script).

---

## Tested setup

| Component | Version |
|---|---|
| Distro / kernel | CachyOS (Arch-based), Linux 7.2 |
| Session | KDE Plasma on Wayland; Wine runs through XWayland (4K monitor at 1.3× plus two 1440p at 1×) |
| GPU / driver | NVIDIA GeForce RTX 5090, proprietary driver 615.78.08 (with `lib32-nvidia-utils`) |
| Proton | GE-Proton11-5 |
| Launcher | umu-launcher 1.4.3 (the Steam Linux Runtime `steamrt4` container) |
| Daz Install Manager | 1.4.1.96 (Win64) |
| Daz Studio | 6.25.2026.14722 |

What works:
- DIM: login, downloading and installing.
- Daz Studio 6: the OpenGL viewport and the PostgreSQL CMS (the content database).
- Iray on the GPU with OptiX. The log shows `Rendering with ... CUDA device 0 (NVIDIA GeForce RTX 5090)`.

> **AMD / Intel GPUs:** Iray is NVIDIA-only (it needs CUDA), so on other GPUs it can only render on the CPU. I haven't tested this guide on AMD or Intel. Everything up to Problem 2 should apply, and you can drop the NVIDIA-specific environment variables.

---

## Why umu + GE-Proton instead of plain Wine or Lutris?

- **Iray needs CUDA.** A Windows program looks for `nvcuda.dll`. Plain Wine doesn't have a working one. Proton (and GE-Proton) can install a bridge DLL that forwards CUDA and OptiX calls to the Linux driver's `libcuda.so` and `libnvoptix.so`.
- **umu-launcher** runs Proton outside Steam, in the same Steam Runtime container Steam uses. It's what Lutris uses internally for Proton games, but calling it directly lets you control every environment variable.
- **Lutris** failed for me because the Lutris config pointed umu at a prefix directory and ended up with a broken nested layout (`prefix/pfx -> .`). Calling `umu-run` directly with `WINEPREFIX` set avoided that.

### Prerequisites (Arch package names; adjust for your distro)

- `umu-launcher`
- A GE-Proton build in `~/.local/share/Steam/compatibilitytools.d/` (install it with ProtonUp-Qt or a similar tool, or extract the release tarball there)
- NVIDIA: `nvidia-utils` **and** `lib32-nvidia-utils`, with versions matching your kernel module. Check that `/usr/lib/libcuda.so.1` and `/usr/lib/libnvoptix.so.1` exist.
- Optional: `p7zip` and `imagemagick` if you want to extract icons for launcher entries
- The **DIM installer** from your Daz account (`DAZ3DIM_<version>_Win64.exe`)

The first time umu runs, it downloads the Steam Runtime (around 650 MB) unless Steam already installed it.

---

## Optional: let Claude Code do the setup for you

This whole setup was worked out in a session with [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), Anthropic's command-line coding agent. You can give it this guide and have it do the same work on your machine. It reads your system, writes the scripts, runs the commands, and reads the Daz and Proton logs when something goes wrong. That last part is where it helps most, because your versions and paths will differ from mine.

1. **Install Claude Code** (see the [setup docs](https://docs.claude.com/en/docs/claude-code/setup) for the current method). At the time of writing:
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```
   Then run `claude` once and sign in.
2. **Make a working folder.** Put this guide (`DAZ-Studio-on-Linux.md`) and your DIM installer (`DAZ3DIM_*_Win64.exe`) in it. If you have backed-up content archives, put them in a folder of their own.
3. **Start Claude Code in that folder** (`cd` into it and run `claude`), and give it a prompt like this:

   ```text
   Read DAZ-Studio-on-Linux.md. I want to set up Daz Studio on this machine the same way.
   First, check my system against the guide's prerequisites (distro, GPU and driver,
   Wayland or X11, which GE-Proton and umu versions are installed) and tell me what's missing.
   Ask me before you install or download anything.
   Then create daz.sh and the prefix, install DIM, and once I've installed Daz Studio
   through DIM, launch it and check the Studio log to confirm Iray sees my GPU.
   Adapt the guide where my setup differs. For example, check whether my GE-Proton
   version still needs the nvidia-libs/nvapi workaround before applying it.
   ```

   Follow-up prompts that work well:
   - "My UI is tiny on my 4K monitor. Check my desktop scale and set the prefix DPI to match."
   - "I put my third-party content zips in ~/3D/Daz Content. Unpack and organize them like Step 6 of the guide, do a dry run first, and register the folder as a content library."
   - "Point my Daz render output at ~/3D/Renders/Daz."
   - "Add app launcher entries for Daz Studio and DIM with their real icons."
   - "Check whether Wine's native Wayland driver draws the Daz viewport yet with my GE-Proton version (Problem 5)."

**Things you'll still do yourself:**
- Accept the Visual C++ license popup.
- Log in to DIM and pick what to download.
- Close Studio when Claude asks: it needs Studio closed before changing settings, because Studio writes its settings back when it exits.
- Look at the result. Claude can read logs, but you're the one who can see whether the UI is the right size or the hover behaves.

**Good habits:**
- Read the permission prompts before approving them.
- Tell it explicitly to ask before downloading, installing, or deleting anything.
- Have it dry-run anything that moves files, as the content script below does by default.

---

## Step 1: Create the launcher script and the prefix

Everything goes through one small script, `daz.sh` (the full version is at the end). Its core is a handful of environment variables:

```bash
export WINEPREFIX="$HOME/Games/daz-studio-proton"   # dedicated prefix, NOT shared with anything else
export PROTONPATH="$HOME/.local/share/Steam/compatibilitytools.d/GE-Proton11-5-x86_64"
export GAMEID=umu-dazstudio      # any "umu-<name>" ID; it also sets the window class (see Step 6)
export STORE=none
export UMU_RUNTIME_UPDATE=0      # stop umu re-downloading the runtime on every launch
```

Use the **full path** to your Proton build in `PROTONPATH`. If you give it a name like `GE-Proton`, umu may try to download the latest release.

Create the prefix:

```bash
./daz.sh init        # runs: umu-run wineboot -u
```

You'll see `Proton: Error: unable to use parent for game drive, path ...`. That's **harmless** for non-Steam apps.

---

## Step 2: Install the Daz Install Manager

The DIM installer is built with BitRock InstallBuilder, so it supports an unattended mode:

```bash
./daz.sh install ~/Downloads/DAZ3DIM_1.4.1.96_Win64.exe --mode unattended
```

**Expect a popup** asking you to accept the *Microsoft Visual C++ 2010 Runtime* license, even in unattended mode. That runtime is bundled inside the DIM installer (nothing extra is downloaded), and DIM needs it. Accept it and the install finishes.

DIM ends up at `C:\Program Files\DAZ 3D\DAZ3DIM1\DAZ3DIM.exe` in the prefix.

## Step 3: Install Daz Studio through DIM

```bash
./daz.sh dim
```

Log in, open **Ready to Download**, and install Daz Studio (4.x or 6; both install side by side as `DAZStudio4` and `DAZStudio6`) plus whatever content you want. DIM also installs the **PostgreSQL CMS**, the content database Studio uses. In my case it worked out of the box under Proton.

Default locations inside the prefix:
- Software: `C:\Program Files\DAZ 3D\`
- Content library: `C:\users\Public\Documents\My DAZ 3D Library`
  (on Linux: `~/Games/daz-studio-proton/drive_c/users/Public/Documents/My DAZ 3D Library`)
- DIM log: `C:\users\steamuser\AppData\Roaming\DAZ 3D\InstallManager\log.txt`
- Studio log: `C:\users\steamuser\AppData\Roaming\DAZ 3D\Studio6\log.txt` (`Studio4` for 4.x)

> **Tip:** if DIM doesn't list all the products you own on first login, close it completely and relaunch. My full product list showed up after a restart.

## Step 4: Launch Daz Studio

```bash
./daz.sh studio
```

The script finds the newest `DAZStudio*/DAZStudio.exe` automatically. With the fixes below applied, Studio starts, the viewport runs on native OpenGL (`RHI driver: NVIDIA ... 4.6.0 NVIDIA 615.78.08` in the Studio log), and Iray uses the GPU.

---

## The problems I hit, and the fixes

Turn on Proton's own logging to debug any of these. The log is written to `$PROTON_LOG_DIR/steam-<gameid>.log`:

```bash
PROTON_LOG=1 ./daz.sh studio
```

### Problem 1: Daz Studio won't start while DIM is open

**Symptom:** You launch Studio and nothing happens. The `umu-run` / `proton` processes are running, but `DAZStudio.exe` never appears, and the Proton log stops after its header.

**Cause:** By default umu starts programs with Proton's `waitforexitandrun` mode, which **waits until every other program in the prefix has exited** before starting. With DIM open, Studio sits in a queue forever.

**Fix:**

```bash
export PROTON_VERB=run
```

Now DIM and Studio can run together in the same prefix.

### Problem 2: Iray says "CUDA driver version 0.0" and renders on CPU only

**Symptom:** In the Studio log, `log.txt`:

```
CUDA   rend warn : The version of your CUDA driver is 0.0, but 12.9 is the required minimum
IRAY   rend warn : Your NVIDIA driver supports CUDA version up to 0.0; iray photoreal can only run in CPU mode.
IRT    rend warn : Your NVIDIA driver API 'nvcuda' cannot be found. All GPUs will be disabled.
       CUDA Device ID: -42
```

The "please update your driver to at least 576.02" message is misleading. Your driver is fine.

**Cause:** By default Proton installs a **stub** `nvcuda.dll` that reports no CUDA. The real bridge to `libcuda.so` lives in Proton's `files/lib/wine/nvidia-libs/` folder and only gets used if you opt in. `PROTON_ENABLE_NVAPI=1` alone is **not** enough.

**Fix:**

```bash
export PROTON_ENABLE_NVAPI=1
export PROTON_NVIDIA_LIBS=1    # enables nvcuda + nvoptix + nvml + nvenc from nvidia-libs
```

(You can enable individual pieces with `PROTON_NVIDIA_NVCUDA=1`, `PROTON_NVIDIA_NVOPTIX=1` and so on, but Iray wants both CUDA and OptiX, so `NVIDIA_LIBS` is simplest.)

After this the Studio log shows:

```
CUDA   rend info : Found 1 CUDA device.
IRAY   rend info : Your NVIDIA driver supports CUDA version up to 13.4; iray requires CUDA version 12.9; all is good.
IRAY   rend info : CUDA device 0 (NVIDIA GeForce RTX 5090): compute capability 12.0, 31.445 GiB total
```

and during a render:

```
IRAY   rend info : Rendering with 2 devices:
IRAY   rend info :     CPU
IRAY   rend info :     CUDA device 0 (NVIDIA GeForce RTX 5090) (FB master) (guidance master)
IRAY   rend info : Initializing OptiX for CUDA device 0
```

> **Close Studio fully before changing these variables.** Proton swaps the DLLs while setting up the prefix at launch, so the new setting only takes effect on a fresh start.

### Problem 3: Proton crashes with `nvidia-libs/nvapi/nvapi64.dll` not found

**Symptom:** After setting `PROTON_NVIDIA_LIBS=1`, the launch fails right away with a Python traceback:

```
FileNotFoundError: [Errno 2] No such file or directory:
  '.../GE-Proton11-5-x86_64/files/lib/wine/nvidia-libs/nvapi//x86_64-windows/nvapi64.dll'
```

**Cause:** This is a packaging issue in **GE-Proton11-5**. When any NVIDIA lib is enabled, Proton's script loads `nvapi` from `nvidia-libs/nvapi/` instead of the usual `nvapi/`, but this GE build doesn't ship that folder. It does ship `nvcuda`, `nvoptix`, `nvml` and `nvenc` under `nvidia-libs/`, plus the regular DXVK-NVAPI build under `files/lib/wine/nvapi/`.

**Fix:** Point the missing folder at the regular nvapi build. Iray talks to `nvcuda` and `nvoptix` directly, so the stock nvapi works fine:

```bash
W="$PROTONPATH/files/lib/wine"
[ -e "$W/nvidia-libs/nvapi" ] || ln -s ../nvapi "$W/nvidia-libs/nvapi"
```

The launcher script does this automatically before every launch, so it survives a reinstall of the same GE-Proton version. Newer GE-Proton releases may fix this; if the folder already exists, the script leaves it alone.

### Problem 4: The UI is microscopic on a 4K / HiDPI screen

**Symptom:** Menus, panes and text in Studio are tiny on a high-DPI monitor, even though the desktop is scaled.

**Cause:** KDE (in "Apply scaling themselves" mode for legacy apps) and some other desktops tell X11 apps their scale through `Xft.dpi` (check with `xrdb -query | grep dpi`). **Wine ignores `Xft.dpi`.** The prefix stays at 96 DPI (100%), so Studio draws at 1× on a 4K screen.

**Fix:** Set the DPI inside the prefix. Daz Studio 6 is built on Qt 6, which follows the Windows DPI setting. Match your desktop scale × 96: 1.25 → 120, 1.3 → 124, 1.5 → 144, 2.0 → 192.

```bash
./daz.sh dpi 124     # writes LogPixels to HKCU\Control Panel\Desktop and the HKLM Fonts key
```

Restart Studio afterwards. (`winecfg` → Graphics → Screen resolution does the same thing.) Two caveats:
- It's one value for **all** monitors. On a mixed 4K + 1440p setup, Studio is scaled on every screen.
- **DIM** is built on Qt 4.8, so expect only its text to grow, not its whole layout.

The scaling applied with both XWayland and Wine's native Wayland driver (see Problem 5).

> **Tip:** while Wine is running it keeps the registry in memory, so `user.reg` on disk won't show the change straight away. Check the live value with `./daz.sh run reg query 'HKCU\Control Panel\Desktop' /v LogPixels`.

### Problem 5 (known issue, not fully solved): UI quirks under XWayland vs. a blank viewport under native Wayland

On a **Wayland desktop** you have to choose between two imperfect options. (On an X11 session neither applies; Proton ignores the Wayland setting there.)

**Option A: XWayland (the default, recommended).** Everything works, including the OpenGL viewport and Iray. There's some UI jank, though. Most noticeably, moving the mouse over the top toolbar or menu bar often highlights the **wrong item** until you click, which resets it. On my setup (KDE Wayland, with one monitor at 1.3× and two at 1×) this was annoying but didn't stop me using Studio. I didn't pin down the exact cause.

**Option B: Wine's native Wayland driver** (`PROTON_ENABLE_WAYLAND=1`, or `DAZ_WAYLAND=1 ./daz.sh studio` with the launcher script). This **fixed the hover problem completely**, but the **3D viewport stays blank**. A loaded character appears in the Scene pane, and clicking **Render** shows it correctly in the render window (Iray on the GPU works), but the viewport never draws anything. That makes it unusable for real work.

Why: Daz's viewport is an OpenGL child window embedded in the main window. GE-Proton11-5's Wayland driver does have code for OpenGL in child windows (Wayland subsurfaces), and a debug log (`WINEDEBUG=warn+waylanddrv,warn+opengl,warn+wgl`) showed no errors, so this looks like a presentation bug in the driver rather than a missing feature. It's worth retesting with newer GE-Proton or Wine releases:

```bash
DAZ_WAYLAND=1 ./daz.sh studio    # load a figure: if the viewport draws, the hover fix comes for free
```

### Harmless warnings you can ignore

| Message | Where | Meaning |
|---|---|---|
| `Proton: Error: unable to use parent for game drive` | terminal | Only matters for Steam games |
| `err:openxr:get_vulkan_extensions ...` | Proton log | VR headset (OpenXR) support; no headset present |
| `WDDM driver used, consider switching to TCC driver model` | Studio log | Windows-only advice |
| `dsp_dzfacetransfer.dll is disabled by evaluation condition` | Studio log | Daz disables Face Transfer here; doesn't look Wine-related |
| `Query (SELECT * FROM "customer") failed ... relation "customer" does not exist` | Studio log | Cloud-notifier query against the CMS; Studio works normally regardless |
| `nvcuvid_video_decoder.dll: Module not found` | Studio log | Iray's optional video-decoder plugin; not needed for rendering |

---

## Step 5 (optional): App launcher entries with real icons

You can pull the icons straight out of the Windows executables with `7z`:

```bash
PF="$HOME/Games/daz-studio-proton/drive_c/Program Files/DAZ 3D"
mkdir -p /tmp/daz-ico/{studio,dim} ~/.local/share/icons/hicolor/256x256/apps
7z x -y -o/tmp/daz-ico/studio "$PF/DAZStudio6/DAZStudio.exe" '.rsrc/ICON/*'
7z x -y -o/tmp/daz-ico/dim    "$PF/DAZ3DIM1/DAZ3DIM.exe"     '.rsrc/ICON/*'

# Studio's 256px icon is already a PNG (resource "1"); DIM's is an .ico that needs converting
cp /tmp/daz-ico/studio/.rsrc/ICON/1 ~/.local/share/icons/hicolor/256x256/apps/daz-studio.png
magick /tmp/daz-ico/dim/.rsrc/ICON/1.ico ~/.local/share/icons/hicolor/256x256/apps/daz-install-manager.png
```

Use `file` on the extracted resources to find the 256×256 one if the numbering differs in your version.

`~/.local/share/applications/daz-studio.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Daz Studio
GenericName=3D Figure Posing and Rendering
Comment=Daz Studio 6 via GE-Proton
Exec="/path/to/daz.sh" studio
Icon=daz-studio
Terminal=false
Categories=Graphics;3DGraphics;
StartupWMClass=steam_app_dazstudio
StartupNotify=true
```

`~/.local/share/applications/daz-install-manager.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Daz Install Manager
Comment=Download and install Daz 3D content via GE-Proton
Exec="/path/to/daz.sh" dim
Icon=daz-install-manager
Terminal=false
Categories=Graphics;3DGraphics;
StartupWMClass=steam_app_daz3dim
StartupNotify=true
```

Then refresh:

```bash
update-desktop-database ~/.local/share/applications
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor
```

**About `StartupWMClass`:** Proton sets each window's X11 class to `steam_app_<id>`, where `<id>` is your `GAMEID` without the `umu-` prefix. If DIM and Studio share a `GAMEID`, they share a window class and get grouped under one taskbar icon. The script gives DIM its own `GAMEID=umu-daz3dim` for that reason. Check a window's class with `xprop WM_CLASS` (click the window). This applies under XWayland, the default. If you opt into Wine's native Wayland driver (`DAZ_WAYLAND=1`), the taskbar matches on the Wayland app ID instead, which I haven't checked.

---

## Step 6: Content from outside the Daz store (Renderosity, backups, freebies)

### Where Daz can see your Linux files

The prefix maps Windows drive letters to Linux paths. `C:` is the prefix itself, and **`Z:` is the root of your Linux filesystem**, so any folder in your home directory is reachable:

| Linux path | Path inside Daz |
|---|---|
| `~/Games/daz-studio-proton/drive_c/...` | `C:\...` |
| `/home/you/3D/Daz Content` | `Z:\home\you\3D\Daz Content` |

(Look at `~/Games/daz-studio-proton/dosdevices/` to see all the mappings. Mine also had an `X:` for my home folder, but `Z:` is the one that's always there.)

**Keep third-party content in its own library outside the prefix**, for example `~/3D/Daz Content`. DIM keeps track of every file it installs in `My DAZ 3D Library`, and a separate folder survives if you ever rebuild the prefix.

### How the files must be laid out

A Daz library root holds folders like `data/`, `People/`, `Props/`, `Runtime/`, `Environments/` and `Light Presets/`. Every product has to be merged so that **its** `data/`, `People/` and so on land in **the library's** `data/`, `People/` and so on. Archives from other stores often don't come that way. In my 13 archives I found:

| Archive layout | What to do |
|---|---|
| `data/`, `People/`, `Runtime/` at the top | Extract straight into the library |
| Wrapped in `My Library/` or `Content/` | Extract what's *inside* the wrapper |
| Wrapped in a product folder (`Feet Variations/data/...`) | Same: use what's inside |
| A zip of zips (Daz-store `IM000…zip` packages, bundle `.rar`s) | Extract the inner zips, then each one's `Content/` folder |
| Readmes, PDFs, promo `.jpg` next to the content folders | Not content; I put them in `ReadMe's/<product>/` the way DIM does |

### Two Linux-specific traps

1. **Capitalisation.** Windows treats `data` and `Data`, or `Daz 3D` and `DAZ 3D`, as the same folder; Linux filesystems don't. My archives used `data` *and* `Data`, `Runtime` *and* `runtime`, and `Daz 3D` *and* `DAZ 3D`. Extracting them naively leaves you with duplicate sibling folders, and Daz (through Wine) may only see one of them. Always merge into the capitalisation that already exists. DIM's own library uses `data/Daz 3D`, `Runtime/Textures` and `Runtime/geometries`.
2. **Backslash paths.** Some zips are written with Windows `\` separators. On Linux, `unzip` and `7z` then produce **single files** named `My Library\data\...\morph.dsf` instead of folders. Daz sees nothing, and the bad files are easy to miss. Check with `find . -name '*\\*'` after extracting.

### Automating it: `organize_content.py`

I wrote a script that deals with all of the above (it's in the [appendix](#appendix-organize_contentpy)). Drop your archives (`.zip`, `.rar`, `.7z`, or already-extracted folders) into the library folder and run:

```bash
# dry run: shows each archive's detected content root and file counts, moves nothing
python3 organize_content.py ~/3D/"Daz Content" ~/3D/"Daz Content Archives" \
    --reference ~/Games/daz-studio-proton/drive_c/users/Public/Documents/"My DAZ 3D Library"

# do it
python3 organize_content.py ... --apply
```

The script:
- extracts each archive, including nested ones, and fixes backslash paths;
- finds the content root inside each package;
- merges the files in ignoring case, copying the capitalisation of the DIM library given with `--reference`;
- skips identical duplicates, and reports (without overwriting) files that clash with different content;
- moves the originals to the archives folder, so nothing is deleted.

### Registering the library in Daz Studio

**From the GUI:** open the Content Library pane menu → **Content Directory Manager**. Add the folder (for example `Z:\home\you\3D\Daz Content`) under **DAZ Studio Formats**. If the folder contains Poser-format files (`.cr2`, `.pz2`, `.pp2`, `.mc6`, …), add it under **Poser Formats** too.

**From the registry** (Studio must be **closed**, or it overwrites your change when it exits). Studio 6 keeps the lists in `HKCU\Software\DAZ\Studio6` as `ContentDir0..N` / `NumContentDirs` and `PoserDir0..N` / `NumPoserDirs`. Check the existing count first, then add the next index:

```bash
./daz.sh run reg query 'HKCU\Software\DAZ\Studio6' | grep -E 'ContentDir|PoserDir|NumContent|NumPoser'

cat > ~/Games/daz-studio-proton/drive_c/"contentdir.reg" <<'EOF'
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\DAZ\Studio6]
"ContentDir2"="Z:/home/you/3D/Daz Content"
"NumContentDirs"=dword:00000003
"PoserDir2"="Z:/home/you/3D/Daz Content"
"NumPoserDirs"=dword:00000003
EOF
./daz.sh run regedit /S 'C:\contentdir.reg'
```

(Studio 4.x uses `HKCU\Software\DAZ\Studio4`.)

**Smart Content:** Daz-store packages include metadata (`Runtime/Support/*.dsx`). To make them show up in Smart Content, use the Content Library pane menu → **Content DB Maintenance** → **Re-Import Metadata**. Products without metadata still appear under their folder in the **Content Library** pane.

## Step 7: Send renders to a folder of your choice

By default Studio saves renders to `C:\users\steamuser\Documents\DAZ 3D\Studio\Render Library`, which is buried inside the prefix (`~/Games/daz-studio-proton/drive_c/users/steamuser/...`).

**From the GUI:** choose your folder in the render window's **Save** dialog (Studio remembers it). If you render with Render Settings → **Render Target: Direct to File**, set the path there as well.

**From the registry** (Studio **closed**), all in one go:

```bash
mkdir -p ~/3D/Renders/Daz
cat > ~/Games/daz-studio-proton/drive_c/"renderdir.reg" <<'EOF'
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\DAZ\Studio6]
"RenderDir0"="Z:/home/you/3D/Renders/Daz"
"ImageSaveDir"="Z:/home/you/3D/Renders/Daz"

[HKEY_CURRENT_USER\Software\DAZ\Studio6\RenderOptions]
"RenderFileName"="Z:/home/you/3D/Renders/Daz/.png"
"RenderMovFileName"="Z:/home/you/3D/Renders/Daz/.avi"
"RenderSerFileName"="Z:/home/you/3D/Renders/Daz/.png"
EOF
./daz.sh run regedit /S 'C:\renderdir.reg'
```

| Value | Controls |
|---|---|
| `RenderDir0` | The folder shown in the **Render Library** pane |
| `ImageSaveDir` | Where the render window's Save dialog opens |
| `RenderFileName`, `RenderSerFileName`, `RenderMovFileName` | "Direct to File" output for still images, image series and movies |

Replace `/home/you` with your real home path. Wine's `.reg` import doesn't expand `~` or `$HOME`.

---

## The complete launcher script

Save as `daz.sh`, `chmod +x daz.sh`, and adjust `PROTONPATH` to your GE-Proton version.

```bash
#!/usr/bin/env bash
# Run DAZ Install Manager / DAZ Studio under GE-Proton via umu-launcher.
#
#   ./daz.sh init               create the Wine prefix
#   ./daz.sh install <exe>      run a Windows installer in the prefix
#   ./daz.sh dim                launch DAZ Install Manager
#   ./daz.sh studio             launch DAZ Studio
#   ./daz.sh run <exe> [args]   run anything else in the prefix
#   ./daz.sh dpi <value>        set UI scaling (96 = 100%, 124 ~= 130%)
#   ./daz.sh winecfg | kill     prefix utilities
set -euo pipefail

export WINEPREFIX="${WINEPREFIX:-$HOME/Games/daz-studio-proton}"
export PROTONPATH="${PROTONPATH:-$HOME/.local/share/Steam/compatibilitytools.d/GE-Proton11-5-x86_64}"
export GAMEID="${GAMEID:-umu-dazstudio}"
export STORE=none
# Don't let umu fetch runtime/Proton updates behind our back.
export UMU_RUNTIME_UPDATE=0
# Default verb (waitforexitandrun) blocks until the prefix is idle, so DIM and Studio couldn't run together.
export PROTON_VERB="${PROTON_VERB:-run}"
# Iray needs real CUDA/OptiX: NVIDIA_LIBS swaps Proton's stub nvcuda.dll for the
# nvidia-libs bridge (nvcuda/nvoptix/nvml/nvenc) to the host libcuda/libnvoptix. Requires nvapi.
export PROTON_ENABLE_NVAPI=1
export PROTON_NVIDIA_LIBS=1
# Daz is an OpenGL app; NVIDIA threaded GL optimisations are off as a precaution (untested whether Daz needs it).
export __GL_THREADED_OPTIMIZATIONS=0
# Opt-in: DAZ_WAYLAND=1 uses Wine's native Wayland driver. It fixes the XWayland toolbar-hover
# glitch, but the OpenGL viewport renders blank there (Iray renders still work), so XWayland stays default.
[[ "${DAZ_WAYLAND:-0}" == 1 ]] && export PROTON_ENABLE_WAYLAND=1
export PROTON_LOG="${PROTON_LOG:-0}"
export PROTON_LOG_DIR="${PROTON_LOG_DIR:-$WINEPREFIX/logs}"

PF="$WINEPREFIX/drive_c/Program Files"
DIM_EXE="$PF/DAZ 3D/DAZ3DIM1/DAZ3DIM.exe"

# Prefer the newest DAZStudio*/ install (e.g. DAZStudio6 over DAZStudio4); override with STUDIO_EXE.
find_studio() {
    find "$PF/DAZ 3D" -maxdepth 2 -iname 'DAZStudio.exe' -ipath '*/DAZStudio*/*' 2>/dev/null | sort -V | tail -n1
}

# GE-Proton11-5 ships nvidia-libs without its nvapi/ dir, so NVIDIA_LIBS crashes prefix setup.
# Point it at the stock dxvk-nvapi build; Iray only needs nvcuda/nvoptix from nvidia-libs.
fix_nvapi() {
    local w="$PROTONPATH/files/lib/wine"
    [[ -d "$w/nvidia-libs" && ! -e "$w/nvidia-libs/nvapi" && -d "$w/nvapi" ]] && ln -s ../nvapi "$w/nvidia-libs/nvapi"
    return 0
}

run() { fix_nvapi; mkdir -p "$PROTON_LOG_DIR"; exec umu-run "$@"; }

# Wine ignores Xft.dpi, so HiDPI scaling has to be set in the prefix (96 = 100%, 124 ~= 130%, 192 = 200%).
set_dpi() {
    local hex reg="$WINEPREFIX/drive_c/dpi.reg"
    [[ "${1:-}" =~ ^[0-9]+$ ]] || { echo "usage: $0 dpi <value>" >&2; exit 1; }
    hex=$(printf '%08x' "$1")
    printf 'Windows Registry Editor Version 5.00\r\n\r\n[HKEY_CURRENT_USER\\Control Panel\\Desktop]\r\n"LogPixels"=dword:%s\r\n\r\n[HKEY_LOCAL_MACHINE\\System\\CurrentControlSet\\Hardware Profiles\\Current\\Software\\Fonts]\r\n"LogPixels"=dword:%s\r\n' "$hex" "$hex" > "$reg"
    run regedit /S 'C:\dpi.reg'
}

case "${1:-}" in
    init)     fix_nvapi; mkdir -p "$WINEPREFIX"; umu-run wineboot -u ;;
    install)  shift; run "$(realpath "$1")" "${@:2}" ;;
    # Distinct GAMEID -> distinct WM_CLASS (steam_app_daz3dim) so DIM gets its own taskbar icon.
    dim)      GAMEID=umu-daz3dim run "$DIM_EXE" ;;
    studio)   exe="${STUDIO_EXE:-$(find_studio)}"
              [[ -n "$exe" ]] || { echo "DAZStudio.exe not found under $PF/DAZ 3D" >&2; exit 1; }
              run "$exe" ;;
    run)      shift; run "$@" ;;
    dpi)      set_dpi "${2:-}" ;;
    winecfg)  run winecfg ;;
    kill)     umu-run wineserver -k ;;
    *)        sed -n '2,11p' "$0"; exit 1 ;;
esac
```

A note on `__GL_THREADED_OPTIMIZATIONS=0`: I set it as a precaution, because NVIDIA's threaded GL optimizations are a common source of trouble with Qt OpenGL apps. I **haven't tested** whether Daz actually needs it. Feel free to experiment.

---

## Quick reference

```bash
./daz.sh init                                       # one time: create the prefix
./daz.sh install DAZ3DIM_x.x.x.x_Win64.exe --mode unattended
./daz.sh dim                                        # Install Manager
./daz.sh studio                                     # Daz Studio
PROTON_LOG=1 ./daz.sh studio                        # with Proton debug log
./daz.sh dpi 124                                    # HiDPI: 96 = 100%, 124 ~= 130%, 192 = 200%
DAZ_WAYLAND=1 ./daz.sh studio                       # native Wayland: fixes hover, but blank viewport
./daz.sh kill                                       # force-close everything in the prefix
python3 organize_content.py LIB ARCHIVES --reference DIM_LIB [--apply]   # unpack third-party content
```

**Checklist if Iray is CPU-only:**
1. Is `PROTON_NVIDIA_LIBS=1` set? (Check that `Options:` in the Proton log includes `nvcuda` / `nvoptix`.)
2. Did Proton crash looking for `nvidia-libs/nvapi`? Apply the symlink from Problem 3.
3. Do `/usr/lib/libcuda.so.1` and `/usr/lib/libnvoptix.so.1` exist, and do `nvidia-utils` and `lib32-nvidia-utils` have the same version as your loaded kernel module (`nvidia-smi`)?
4. Did you fully close Studio (and check `pgrep -fa DAZStudio`) before relaunching?

---

## Appendix: organize_content.py

Used in [Step 6](#step-6-content-from-outside-the-daz-store-renderosity-backups-freebies). Needs `python3` (3.10+) and `7z` (`p7zip`; it reads `.rar` too). It only reads archives and moves files, and it never executes anything from them.

```python
#!/usr/bin/env python3
"""Unpack third-party Daz content archives into a Daz content library.

    organize_content.py <library_dir> <archives_dir> [--reference <dim_library>] [--apply]

Each archive (.zip/.rar/.7z, or an already-extracted folder) in <library_dir> is
extracted to a staging dir (nested archives too), its content root is located
(skipping wrappers like "My Library/", "Content/", "<Product Name>/"), and its
files are merged into <library_dir> case-insensitively, using the casing of
existing folders (seeded from --reference) so Linux doesn't end up with both
"data/Daz 3D" and "data/DAZ 3D". Readmes/PDFs/promo images go to
"ReadMe's/<product>/". The originals are moved to <archives_dir>.
Without --apply, only prints what would happen.
"""
import argparse
import filecmp
import os
import shutil
import subprocess
import sys
from pathlib import Path

ARCHIVE_EXTS = {".zip", ".rar", ".7z"}
# Top-level folders that mark a Daz library root (lowercased).
CONTENT_DIRS = {
    "data", "people", "runtime", "props", "environments", "light presets", "lights",
    "scenes", "shader presets", "render presets", "render settings", "scripts",
    "camera presets", "animals", "general", "aniblocks", "poses", "pose presets",
    "figures", "presets", "scene builder", "architecture", "vehicles", "landscapes",
}
SKIP_NAMES = {"manifest.dsx", "supplement.dsx", "thumbs.db", ".ds_store", "desktop.ini"}


def extract(archive: Path, dest: Path) -> None:
    out = subprocess.run(["7z", "l", "-slt", "-ba", str(archive)], capture_output=True, text=True, check=True).stdout
    for line in out.splitlines():
        if line.startswith("Path = "):
            p = line[7:].replace("\\", "/")
            if p.startswith("/") or ".." in p.split("/"):
                raise RuntimeError(f"unsafe path {p!r} in {archive.name}")
    dest.mkdir(parents=True, exist_ok=True)
    subprocess.run(["7z", "x", "-y", "-bso0", "-bsp0", f"-o{dest}", str(archive)], check=True)
    # Zips written with Windows "\" separators extract as single files named "a\b\c.dsf".
    for f in [p for p in dest.rglob("*") if "\\" in p.name]:
        if f.is_dir():
            continue
        target = f.parent.joinpath(*[part for part in f.name.split("\\") if part])
        target.parent.mkdir(parents=True, exist_ok=True)
        f.rename(target)


def expand_nested(stage: Path) -> None:
    """Extract archives found inside the staging tree, in place, until none remain."""
    while True:
        nested = [p for p in stage.rglob("*") if p.is_file() and p.suffix.lower() in ARCHIVE_EXTS]
        if not nested:
            return
        for a in nested:
            extract(a, a.with_suffix(""))
            a.unlink()


def content_roots(stage: Path) -> list[Path]:
    """Shallowest dirs that contain at least one known library folder."""
    roots = []

    def walk(d: Path) -> None:
        kids = [k for k in d.iterdir() if k.is_dir()]
        if any(k.name.lower() in CONTENT_DIRS for k in kids):
            roots.append(d)
            return
        for k in kids:
            walk(k)

    walk(stage)
    return roots


class CaseMap:
    """Resolve relative paths case-insensitively against the library + a reference library."""

    def __init__(self, library: Path, reference: Path | None):
        self.library = library
        self.reference = reference

    @staticmethod
    def _match(parent: Path, name: str) -> str | None:
        if not parent.is_dir():
            return None
        low = name.lower()
        for k in parent.iterdir():
            if k.name.lower() == low:
                return k.name
        return None

    def resolve(self, rel: Path, planned: dict) -> Path:
        out = Path()
        for i, part in enumerate(rel.parts):
            is_dir = i < len(rel.parts) - 1
            key = (str(out).lower(), part.lower())
            name = planned.get(key)
            if name is None:
                name = self._match(self.library / out, part)
            if name is None and is_dir and self.reference:
                name = self._match(self.reference / out, part)
            name = name or part
            planned[key] = name
            out = out / name
        return out


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("library", type=Path)
    ap.add_argument("archives", type=Path)
    ap.add_argument("--reference", type=Path)
    ap.add_argument("--apply", action="store_true")
    a = ap.parse_args()
    lib, arch = a.library.resolve(), a.archives.resolve()
    stage_base = lib.parent / f".{lib.name}-staging"
    cm = CaseMap(lib, a.reference)
    planned: dict = {}  # keeps casing decisions consistent within a dry run

    sources = sorted(p for p in lib.iterdir()
                     if (p.is_file() and p.suffix.lower() in ARCHIVE_EXTS)
                     or (p.is_dir() and p.name.lower() not in CONTENT_DIRS
                         and p.name != "ReadMe's" and not p.name.startswith(".")))
    if not sources:
        print("nothing to do")
        return 0

    totals = {"new": 0, "same": 0, "conflict": 0, "docs": 0}
    conflicts = []
    for src in sources:
        product = src.stem if src.is_file() else src.name
        stage = stage_base / product
        if stage.exists():
            shutil.rmtree(stage)
        if src.is_file():
            extract(src, stage)
            expand_nested(stage)
        else:
            shutil.copytree(src, stage)
            expand_nested(stage)
        roots = content_roots(stage)
        if not roots:
            print(f"!! {product}: no content root found, leaving it alone")
            continue
        counts = {"new": 0, "same": 0, "conflict": 0, "docs": 0}
        root_files = set()
        for root in roots:
            for f in root.rglob("*"):
                if not f.is_file():
                    continue
                root_files.add(f)
                rel = f.relative_to(root)
                top = rel.parts[0].lower()
                if f.name.lower() in SKIP_NAMES:
                    continue
                is_doc = len(rel.parts) == 1 or top not in CONTENT_DIRS
                if is_doc:
                    dst_rel = Path("ReadMe's") / product / rel
                else:
                    dst_rel = cm.resolve(rel, planned)
                dst = lib / dst_rel
                if dst.exists():
                    if filecmp.cmp(f, dst, shallow=False):
                        counts["same"] += 1
                        continue
                    counts["conflict"] += 1
                    conflicts.append(f"{product}: {dst_rel}")
                    continue
                counts["docs" if is_doc else "new"] += 1
                if a.apply:
                    dst.parent.mkdir(parents=True, exist_ok=True)
                    shutil.move(str(f), dst)
        # Files outside the content roots: README pdfs, promo jpgs, licenses.
        for f in stage.rglob("*"):
            if f.is_file() and f not in root_files and f.name.lower() not in SKIP_NAMES:
                dst = lib / "ReadMe's" / product / f.name
                counts["docs"] += 1
                if a.apply and not dst.exists():
                    dst.parent.mkdir(parents=True, exist_ok=True)
                    shutil.move(str(f), dst)
        for k in totals:
            totals[k] += counts[k]
        rels = ", ".join(str(r.relative_to(stage)) or "." for r in roots)
        print(f"{product}\n    root: {rels}\n    files: {counts['new']} new, {counts['same']} identical, "
              f"{counts['conflict']} conflicting, {counts['docs']} docs")
        if a.apply:
            arch.mkdir(parents=True, exist_ok=True)
            shutil.move(str(src), arch / src.name)
            shutil.rmtree(stage)
    if a.apply and stage_base.exists() and not any(stage_base.iterdir()):
        stage_base.rmdir()

    print(f"\nTOTAL: {totals['new']} new, {totals['same']} identical, {totals['conflict']} conflicting, {totals['docs']} docs")
    for c in conflicts:
        print("  conflict (kept existing):", c)
    if not a.apply:
        print("\n(dry run: nothing moved; staging left in", stage_base, ")")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

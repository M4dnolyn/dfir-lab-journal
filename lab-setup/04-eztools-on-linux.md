# 04 — Eric Zimmerman's Tools on Linux (.NET 9)

## Objective

Deploy Eric Zimmerman's Tools (EZ Tools) on the Ubuntu VM for Windows artifact analysis — filesystem, registry, event logs, prefetch, links, and jump lists.

## Prerequisites

- Operational Ubuntu 24.04 LTS VM (see [01-vm-provisioning.md](01-vm-provisioning.md))
- .NET SDK 8.x already installed (`dotnet --version` returns 8.0.x)
- Internet access from the VM

## 1. Context — EZ Tools on Linux

EZ Tools are developed in .NET and distributed in two forms:

| Build | Extension | Compatibility | Distribution |
|-------|-----------|---------------|--------------|
| .NET Framework 4.x | `.exe` | Windows only | Default link on website |
| .NET 9 cross-platform | `.dll` | Windows, Linux, macOS | <https://download.ericzimmermanstools.com/net9/> |

The .NET 9 build is the only functional option on Linux. It requires the .NET 9 runtime, which is not provided by Ubuntu 24.04 repositories (stops at .NET 8). The runtime is therefore installed via Microsoft's official script.

> **Note**: .NET Framework builds (`.exe` only, without `.dll` or `.runtimeconfig.json`) do not work with the `dotnet` command on Linux, regardless of installed runtime version.

## 2. Install .NET 9 Runtime

The `dotnet-runtime-9.0` package is not available in Ubuntu 24.04 repositories. Installation is via Microsoft's official script:

```bash
cd ~
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 9.0 --runtime dotnet
```

The script installs the runtime in `~/.dotnet/` and displays the installed version:

```
dotnet-install: Installed version is 9.0.x
```

### Permanent PATH Addition

```bash
echo 'export DOTNET_ROOT=$HOME/.dotnet' >> ~/.bashrc
echo 'export PATH=$HOME/.dotnet:$PATH' >> ~/.bashrc
source ~/.bashrc
```

**Verification** — both runtimes must be listed:

```bash
dotnet --list-runtimes
# Microsoft.NETCore.App 8.0.x [...]
# Microsoft.NETCore.App 9.0.x [...]
```

## 3. Download Tools

.NET 9 builds are available at: <https://download.ericzimmermanstools.com/net9/<ToolName>.zip>

Tools downloaded for this lab:

| Tool | Target Artifacts |
|------|------------------|
| MFTECmd | `$MFT`, `$LogFile`, `$UsnJrnl`, `$Boot` |
| EvtxECmd | Windows event logs (`.evtx`) |
| RECmd | Registry hives (SYSTEM, SOFTWARE, NTUSER.DAT, SAM) |
| PECmd | Prefetch (`.pf`) |
| AmcacheParser | Amcache.hve |
| AppCompatCacheParser | ShimCache (SYSTEM\CurrentControlSet\Control\Session Manager\AppCompatCache) |
| LECmd | LNK files |
| JLECmd | Jump Lists |

Download each archive to `~/Téléchargements/` via VM browser, or command line:

```bash
BASE="https://download.ericzimmermanstools.com/net9"
for tool in MFTECmd EvtxECmd RECmd PECmd AmcacheParser \
            AppCompatCacheParser LECmd JLECmd; do
  wget -q "$BASE/$tool.zip" -O ~/Téléchargements/$tool.zip
done
```

## 4. Extraction

```bash
mkdir -p ~/eztools
for z in ~/Téléchargements/*.zip; do
  n=$(basename "$z" .zip)
  unzip -oq "$z" -d ~/eztools/$n
done
```

**Verification** — each tool must expose a main `.dll` file:

```bash
find ~/eztools -name "*.dll" -not -path "*/Plugins/*"
```

Expected result (one `.dll` per tool; RECmd additionally has a `Plugins/` subfolder containing registry plugins):

```
~/eztools/MFTECmd/MFTECmd.dll
~/eztools/EvtxECmd/EvtxeCmd/EvtxECmd.dll
~/eztools/RECmd/RECmd/RECmd.dll
~/eztools/PECmd/PECmd.dll
~/eztools/AmcacheParser/AmcacheParser.dll
~/eztools/AppCompatCacheParser/AppCompatCacheParser.dll
~/eztools/LECmd/LECmd.dll
~/eztools/JLECmd/JLECmd.dll
```

> Internal archive structure varies by tool — some place the `.dll` directly at root, others in a subfolder named after the tool. Trust the `find` output rather than assumed paths.

## 5. Alias Configuration

To avoid typing full paths on each invocation, aliases are defined in `~/.bashrc`. Paths used are those returned by `find` at step 4:

```bash
cat >> ~/.bashrc << 'EOF_ALIAS'
alias mftecmd="dotnet $HOME/eztools/MFTECmd/MFTECmd.dll"
alias evtxecmd="dotnet $HOME/eztools/EvtxECmd/EvtxeCmd/EvtxECmd.dll"
alias recmd="dotnet $HOME/eztools/RECmd/RECmd/RECmd.dll"
alias pecmd="dotnet $HOME/eztools/PECmd/PECmd.dll"
alias amcacheparser="dotnet $HOME/eztools/AmcacheParser/AmcacheParser.dll"
alias appcompatcacheparser="dotnet $HOME/eztools/AppCompatCacheParser/AppCompatCacheParser.dll"
alias lecmd="dotnet $HOME/eztools/LECmd/LECmd.dll"
alias jlecmd="dotnet $HOME/eztools/JLECmd/JLECmd.dll"
EOF_ALIAS
source ~/.bashrc
```

## 6. Global Verification

Test all tools in a single command:

```bash
for dll in $(find ~/eztools -name "*.dll" -not -path "*/Plugins/*"); do
  echo "=== $(basename $dll .dll) ==="
  dotnet "$dll" --help 2>&1 | head -3
  echo
done
```

Each tool must display its version and author name. Example for MFTECmd:

```
Description:
  MFTECmd version 2026.5.0
  Author: Eric Zimmerman (saericzimmerman@gmail.com)
```

Quick alias test:

```bash
mftecmd --help | head -3
recmd --help | head -3
```

## 7. Quick Usage Reference

Commands below are provided as syntax reference. Their application on case files will be performed during corresponding analysis phases of the course.

### MFTECmd — `$MFT` Analysis

```bash
mftecmd -f /path/to/\$MFT --csv ~/output/ --csvf mft.csv
```

### EvtxECmd — Event Logs

```bash
evtxecmd -d /path/to/EventLogs/ --csv ~/output/ --csvf evtx.csv
```

### RECmd — Registry Hives

```bash
recmd -f /path/to/SYSTEM --csv ~/output/ --csvf system.csv
```

### PECmd — Prefetch

```bash
pecmd -d /path/to/Prefetch/ --csv ~/output/ --csvf prefetch.csv
```

### AmcacheParser

```bash
amcacheparser -f /path/to/Amcache.hve --csv ~/output/
```

### AppCompatCacheParser

```bash
appcompatcacheparser -f /path/to/SYSTEM --csv ~/output/
```

### LECmd — LNK Files

```bash
lecmd -d /path/to/Recent/ --csv ~/output/ --csvf lnk.csv
```

### JLECmd — Jump Lists

```bash
jlecmd -d /path/to/AutomaticDestinations/ --csv ~/output/
```

## 8. Issues Encountered and Resolutions

### Tools Non-Functional After Download from Main Site

**Symptom**: Archives downloaded via the default link on `ericzimmerman.github.io` contain only a `.exe` file, without `.dll` or `.runtimeconfig.json`. Running `dotnet ToolName.exe` returns:

```
Could not execute because the specified command or file
was not found.
```

**Cause**: The default download link on the site points to .NET Framework 4.x builds — standalone Windows executables. These builds are not compatible with `dotnet` on Linux.

**Resolution**: Download exclusively from the `net9` subdirectory: <https://download.ericzimmermanstools.com/net9/<ToolName>.zip>

### .NET 9 Runtime Not Found via apt

**Symptom**:

```
E: Unable to locate package dotnet-runtime-9.0
```

**Cause**: Ubuntu 24.04 LTS only provides .NET 8 runtimes in its official repositories. .NET 9 is not yet packaged.

**Resolution**: Use Microsoft's official install script (§2 of this document) — it installs the runtime in `~/.dotnet/` independently of the system package manager.

## ✅ Expected Final Configuration State

- [x] .NET 9.0.x runtime installed in `~/.dotnet/`
- [x] `DOTNET_ROOT` and `PATH` configured in `~/.bashrc`
- [x] 8 EZ Tools extracted in `~/eztools/`
- [x] Aliases configured for each tool (mftecmd, evtxecmd, recmd, etc.)
- [x] All tools respond to `--help` without error
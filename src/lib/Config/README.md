# Configuration System

This directory contains the configuration system for A Movement Suite, managing program paths and user preferences.

## Quick Start

**Most users don't need to configure anything!** The system auto-detects common installations.

### If Auto-Detection Fails

1. **Copy the template:**
   ```
   UserConfig.template.ahk2  →  UserConfig.local.ahk2
   ```

2. **Edit `UserConfig.local.ahk2`** with your custom paths

3. **Reload the script**

## File Purposes

| File | Purpose | Version Controlled? |
|------|---------|---------------------|
| `GlobalConfig.ahk2` | System defaults, timing, feature toggles | ✓ Yes |
| `UserConfig.ahk2` | Auto-detection, program paths | ✓ Yes |
| `UserConfig.template.ahk2` | Template for user customization | ✓ Yes |
| `UserConfig.local.ahk2` | **Your machine-specific overrides** | ✗ No (gitignored) |

## Configured Programs

### WezTerm (Terminal Emulator)
**Hotkey:** RAlt + F12

**Auto-detection checks:**
1. Environment variable: `WEZTERM_PATH`
2. `C:\Program Files\WezTerm\wezterm-gui.exe`
3. `C:\PortableApps\Programs\WezTerm\wezterm-gui.exe`
4. `%LOCALAPPDATA%\Microsoft\WindowsApps\wezterm-gui.exe`
5. Scoop installation
6. Local bin

**Override example:**
```ahk
ProgramPaths.WezTerm := "D:\Tools\WezTerm\wezterm-gui.exe"
```

### Everything (File Search)
**Hotkey:** Win + Space

**Auto-detection checks:**
1. Environment variable: `EVERYTHING_PATH`
2. `C:\Program Files\Everything\Everything.exe`
3. `C:\PortableApps\Programs\Everything\everything.exe`
4. `%LOCALAPPDATA%\Programs\Everything\Everything.exe`
5. Scoop installation

**Override example:**
```ahk
ProgramPaths.Everything := "C:\PortableApps\Programs\Everything\everything.exe"
```

## Advanced Configuration

### Environment Variables

Instead of creating `UserConfig.local.ahk2`, you can set environment variables:

```
WEZTERM_PATH=C:\Your\Path\wezterm-gui.exe
EVERYTHING_PATH=C:\Your\Path\Everything.exe
```

### Computer-Specific Configuration

```ahk
; In UserConfig.local.ahk2:
if (A_ComputerName = "WORK-PC") {
    ProgramPaths.WezTerm := "C:\Tools\WezTerm\wezterm-gui.exe"
} else if (A_ComputerName = "HOME-PC") {
    ProgramPaths.WezTerm := "D:\Apps\WezTerm\wezterm-gui.exe"
}
```

### Shell Preference

```ahk
ProgramPaths.Shell := "cmd.exe"  ; Use Command Prompt instead of PowerShell
```

Options: `pwsh.exe` (PowerShell 7+), `cmd.exe`, `bash.exe`, `wsl.exe`

## Troubleshooting

### Check Detected Paths

**Tray Menu → Program Paths Configuration**

This shows:
- Which paths were detected
- Whether files exist (✓/✗)
- Instructions for customization

### Error: "WezTerm not found"

1. Check if WezTerm is installed
2. Verify installation path
3. Create `UserConfig.local.ahk2` with correct path
4. OR set `WEZTERM_PATH` environment variable
5. Reload script

### Error: "Everything not found"

Same steps as WezTerm, but for Everything.exe

### Path Detection Not Working

1. Open Tray Menu → Program Paths Configuration
2. Check which paths are being detected
3. If path is wrong, override in `UserConfig.local.ahk2`
4. Reload script

## Adding New Programs

To add support for a new program:

1. **Add auto-detection in `UserConfig.ahk2`:**
   ```ahk
   class ProgramPaths {
       static MyProgram := ProgramPaths._DetectMyProgram()

       static _DetectMyProgram() {
           locations := [
               A_ProgramFiles "\MyProgram\myprogram.exe",
               "C:\PortableApps\Programs\MyProgram\myprogram.exe"
           ]

           for path in locations {
               if FileExist(path)
                   return path
           }

           return A_ProgramFiles "\MyProgram\myprogram.exe"
       }

       static ValidateMyProgram() {
           return FileExist(ProgramPaths.MyProgram) ? true : false
       }
   }
   ```

2. **Update `GetAllPaths()` method:**
   ```ahk
   static GetAllPaths() {
       return Map(
           "WezTerm", ProgramPaths.WezTerm,
           "Everything", ProgramPaths.Everything,
           "MyProgram", ProgramPaths.MyProgram  ; Add here
       )
   }
   ```

3. **Use in your hotkey:**
   ```ahk
   RAlt & F10::{
       if (!ProgramPaths.ValidateMyProgram()) {
           MsgBox("MyProgram not found!")
           return
       }

       Run '"' ProgramPaths.MyProgram '"'
   }
   ```

## Architecture

```
GlobalConfig.ahk2
    ↓ Includes
UserConfig.ahk2 (auto-detection)
    ↓ Optionally includes (if exists)
UserConfig.local.ahk2 (machine-specific overrides)
```

**Benefits:**
- ✓ No source code editing required
- ✓ Works across different machines
- ✓ Git-friendly (local config is ignored)
- ✓ Auto-detection works for 90% of users
- ✓ Easy to extend with new programs

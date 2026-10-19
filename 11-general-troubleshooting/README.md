# General Troubleshooting Method (Master Checklist)

The per-section troubleshooting blocks tell you *what* to check for each area. This is the mindset that ties them together — apply it to any problem, even one not listed above.

**1. Read the actual error message.** It almost always names the file, permission, port, or dependency at fault. Don't skim it. If a command is quiet, make it talk: add `-v`/`--verbose`, or run it with `bash -x`.

**2. Reproduce it deliberately.** Know the exact command or action that triggers the problem. A bug you can reproduce on demand is halfway solved; an intermittent one needs logging to catch.

**3. Check the logs.** This is the single highest-value habit. `journalctl -xe` for recent errors, `journalctl -u <service>` for a specific service, `dmesg` for kernel/hardware, `/var/log/auth.log` for access. The answer is usually already written down.

**4. Isolate the layer.** Work outward and ask "which layer fails?" — is it the application, the config file, permissions, the service manager, the network, or the hardware? The network troubleshooting ladder (interface → gateway → internet → DNS) is the model: test one link at a time so you know where it breaks.

**5. Question what changed.** Most breakage follows a change: an update, an edited config, a new rule, a full disk. `history` shows your recent commands; package logs show recent updates. If it worked yesterday, something moved.

**6. Check the boring basics first**, in this rough order — they cause a huge share of real incidents:
- **Disk full?** `df -h` and `df -i` (bytes *and* inodes).
- **Out of memory?** `free -h` (watch *available* and swap).
- **Service running?** `systemctl status <name>`.
- **Permissions right?** owner, mode, and every parent directory.
- **Right path / typo?** `which`, absolute paths.
- **Firewall or SELinux blocking?** especially when something "should" work.
- **DNS / time correct?** `dig`, `timedatectl`.

**7. Change one thing at a time.** Make a single change, test, observe. Changing several at once means you won't know which fixed (or worsened) it. Note what you tried.

**8. Test in permissive/safe mode to locate, then fix properly.** Temporarily loosening SELinux, a firewall, or permissions is a *diagnostic* to confirm the cause — never the final fix. Once you've confirmed it, put the correct, minimal rule back.

**9. Search the exact error text.** Copy the precise message (strip your hostnames/paths) into a search engine or the man page. Someone has almost certainly hit it. Trust official docs and man pages (`man cmd`, `cmd --help`) over random snippets.

**10. Keep a way back in.** Before changing SSH, firewalls, fstab, bootloader, or sudoers: keep a second session open, back up the file (`cp file file.bak`), and know your recovery route (console access, live USB, rescue mode). These are the changes that lock you out.

A compact "first five commands" you can run on almost any misbehaving Linux box: `systemctl status <thing>`, `journalctl -xe`, `df -h`, `free -h`, `dmesg | tail`. They surface the majority of everyday problems in under a minute.

---
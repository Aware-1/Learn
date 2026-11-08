# LPIC-1 (101 & 102) Study Handout – Stage 1

For each topic: a short explanation, the most important man pages, and one tiny example per command. Note: your list starts at item 2 (item 1 is missing), so numbering follows your list.

## Part A – Exam 101

### 2. Hardware, Drivers, procfs, sysfs, D-Bus, udev, HAL

The kernel exposes hardware info through `/proc` and `/sys`; udev creates device nodes in `/dev` dynamically; D-Bus lets desktop services talk to each other. HAL is obsolete (replaced by udev + D-Bus). **Man pages:** `proc(5)`, `sysfs(5)`, `udev(7)`, `udevadm(8)`, `lspci(8)`, `lsusb(8)`, `lsmod(8)`, `modprobe(8)`, `dbus-daemon(1)`

- `lspci -k` – PCI devices and the driver in use
- `lsusb` – list USB devices
- `lsmod` – loaded kernel modules
- `modprobe loop` – load a module (`modprobe -r loop` unloads it)
- `udevadm info /dev/sda` – udev properties of a device
- `cat /proc/cpuinfo` – CPU info from procfs

### 3. How Linux Boots

BIOS/UEFI → bootloader (GRUB) → kernel + initramfs → init/systemd → services. **Man pages:** `boot(7)`, `bootparam(7)`, `dmesg(1)`, `journalctl(1)`

- `dmesg | less` – kernel boot messages
- `journalctl -b` – logs of the current boot
- `cat /proc/cmdline` – kernel parameters used at boot

### 4. Runlevels and systemd

Runlevels (SysV) define system states (0 halt, 1 single, 3 multi-user text, 5 GUI, 6 reboot); systemd replaces them with targets. **Man pages:** `runlevel(8)`, `telinit(8)`, `inittab(5)`, `systemctl(1)`, `systemd.target(5)`, `systemd.unit(5)`

- `runlevel` – shows previous and current level (e.g. `N 5`)
- `telinit 3` – switch to runlevel 3
- `systemctl get-default` – show default target
- `systemctl isolate multi-user.target` – switch target now
- `systemctl enable --now sshd` – start a service and enable at boot

### 5. Disk Partitioning, SWAP, Mount Points, fdisk

Disks are split into partitions (MBR/GPT); each is formatted and mounted in the directory tree; swap is disk space used as extra memory. **Man pages:** `fdisk(8)`, `lsblk(8)`, `mkswap(8)`, `swapon(8)`, `df(1)`, `hier(7)`

- `lsblk` – tree of disks and partitions
- `fdisk -l /dev/sda` – print the partition table
- `mkswap /dev/sda2` – format a partition as swap
- `swapon /dev/sda2` – enable swap (`swapon --show` to verify)
- `df -h` – mounted filesystems and free space

### 6. Boot Managers (LILO, GRUB)

The bootloader loads the kernel. LILO is legacy (must be re-run after every change); GRUB/GRUB2 reads its config at boot. **Man pages:** `grub-install(8)`, `grub-mkconfig(8)`, `lilo(8)`, `lilo.conf(5)` (RHEL: `grub2-*`)

- `grub-install /dev/sda` – install GRUB to the MBR
- `grub-mkconfig -o /boot/grub/grub.cfg` – regenerate GRUB config
- `lilo -v` – write LILO config to the boot sector

### 7. Shared and Static Libraries

Shared libraries (`.so`) are loaded at run time and shared by programs; static libraries (`.a`) are linked into the binary at build time. **Man pages:** `ldd(1)`, `ldconfig(8)`, `ld.so(8)`, `ar(1)`

- `ldd /bin/ls` – libraries a binary needs
- `ldconfig -p | grep libc` – search the library cache
- `ldconfig` – rebuild the cache after editing `/etc/ld.so.conf`
- `LD_LIBRARY_PATH=/opt/lib ./app` – temporary extra library path
- `ar t libfoo.a` – list members of a static library

### 8. Package Management (Debian family)

A package manager installs, upgrades, removes software and resolves dependencies. `dpkg` is the low-level tool; `apt` handles repositories and dependencies. **Man pages:** `dpkg(1)`, `apt(8)`, `apt-get(8)`, `apt-cache(8)`, `sources.list(5)`

- `apt update` – refresh package lists
- `apt install htop` – install with dependencies
- `apt-cache search editor` – search packages
- `dpkg -l | grep nginx` – is it installed?
- `dpkg -L nginx` – files owned by a package
- `dpkg -S /bin/ls` – which package owns a file
- `dpkg -i pkg.deb` – install a local .deb

### 9. RedHat Family: rpm and yum

`rpm` handles single packages; `yum` (now `dnf`) resolves dependencies from repos defined in `/etc/yum.repos.d/`. **Man pages:** `rpm(8)`, `yum(8)`, `dnf(8)`, `yum.conf(5)`, `rpm2cpio(8)`

- `rpm -qa` – list all installed packages
- `rpm -qi bash` – package info
- `rpm -ql bash` – files in a package
- `rpm -qf /bin/ls` – which package owns a file
- `rpm -ivh pkg.rpm` – install
- `yum install httpd` – install with dependencies
- `yum provides */ifconfig` – which package gives a file
- `rpm2cpio pkg.rpm | cpio -idmv` – extract an rpm without installing

### 10. help, export, exec, Environment Variables

Variables are local to a shell unless exported; `exec` replaces the current shell with a command. **Man pages:** `bash(1)` (section SHELL BUILTIN COMMANDS), `env(1)`, `printenv(1)`

- `help cd` – help for a shell builtin
- `export PATH=$PATH:/opt/bin` – make a variable visible to child processes
- `env` – show the environment
- `unset VAR` – remove a variable
- `exec bash` – replace this shell with a new one

### 11. Text Streams and Filters

Filters read stdin, transform it, write stdout; chain them with pipes. **Man pages:** `cat`, `head`, `tail`, `cut`, `sort`, `uniq`, `wc`, `tr`, `tee`, `paste`, `join`, `nl`, `od`, `fmt`, `pr`, `split`, `expand`, `unexpand`, `sed`, `hexdump` (all section 1)

- `cat file` – print file
- `head -n 5 file` / `tail -n 5 file` – first / last lines (`tail -f` follows)
- `cut -d: -f1 /etc/passwd` – extract field 1
- `sort -n file` – numeric sort
- `uniq -c` – count adjacent duplicates (sort first)
- `wc -l file` – count lines
- `tr a-z A-Z` – translate characters
- `tee out.txt` – write to file and stdout
- `paste a.txt b.txt` – merge files side by side
- `join f1 f2` – join on a common field
- `nl file` – number lines
- `od -c file` – dump in octal/chars
- `fmt -w 40 file` – re-wrap text
- `pr -2 file` – paginate in columns
- `split -l 100 big.txt` – split into 100-line pieces
- `expand file` – tabs to spaces
- `sed 's/a/b/' file` – substitute text

### 12. File Management, Archive and Compression

Core file tools plus archivers (`tar`, `cpio`) and compressors (`gzip`, `bzip2`, `xz`). **Man pages:** `ls`, `cp`, `mv`, `rm`, `mkdir`, `touch`, `file`, `dd`, `tar`, `gzip`, `bzip2`, `xz`, `cpio`

- `ls -lah` – long listing, hidden files, human sizes
- `cp -r dir1 dir2` – copy recursively
- `mv a b` – move/rename
- `rm -rf dir` – delete recursively (careful!)
- `touch f` – create empty file / update timestamp
- `file /bin/ls` – detect file type
- `dd if=/dev/zero of=t.img bs=1M count=10` – raw copy / make a 10 MB file
- `tar czf a.tar.gz dir/` – create gzip archive (`xzf` to extract)
- `gzip f` / `bzip2 f` / `xz f` – compress a file
- `find . | cpio -o > a.cpio` – create a cpio archive

### 13. Pipes and Redirection

Every process has stdin (0), stdout (1), stderr (2); `|` connects stdout of one command to stdin of the next. **Man pages:** `bash(1)` (section REDIRECTION), `tee(1)`, `xargs(1)`

- `ls | wc -l` – pipe
- `echo hi > f` – overwrite file; `>>` appends
- `sort < f` – read stdin from file
- `cmd 2> err.log` – redirect stderr
- `cmd > all.log 2>&1` – stdout and stderr to one file
- `find . -name '*.tmp' | xargs rm` – build command lines from stdin

### 14. Process Management

Monitor and control processes, jobs and signals. **Man pages:** `ps(1)`, `top(1)`, `kill(1)`, `pkill(1)`, `pgrep(1)`, `jobs`/`bg`/`fg` (bash), `nohup(1)`, `free(1)`, `uptime(1)`, `signal(7)`

- `ps aux` – all processes
- `top` – live view (`q` to quit)
- `kill -9 1234` – force-kill PID 1234
- `pkill firefox` – kill by name
- `sleep 100 &` then `jobs` – background job list
- `fg %1` / `bg %1` – bring to foreground / resume in background
- `nohup ./job.sh &` – survive logout
- `free -m` – memory usage in MB

### 15. Process Priority and Nice Level

Nice ranges from -20 (highest priority) to 19 (lowest); only root can lower it below 0. **Man pages:** `nice(1)`, `renice(1)`, `top(1)`, `ps(1)`

- `nice -n 10 ./job` – start with lower priority
- `renice -n 5 -p 1234` – change a running process
- `ps -o pid,ni,comm -p 1234` – show nice value
- In `top`: press `r` to renice

### 16. grep and sed

`grep` searches text with regex; `sed` edits streams. **Man pages:** `grep(1)`, `egrep(1)`, `sed(1)`, `regex(7)`

- `grep -i error log` – case-insensitive search
- `grep -rn 'TODO' .` – recursive with line numbers
- `grep -v '^#' conf` – hide comment lines
- `egrep 'a|b' f` – extended regex (same as `grep -E`)
- `sed 's/old/new/g' f` – replace all matches
- `sed -n '2,4p' f` – print lines 2–4
- `sed -i '/^$/d' f` – delete empty lines in place

### 17. vi Editor

Modal editor: Normal mode (commands), Insert mode (typing), Command-line mode (`:`). **Man pages:** `vi(1)`, `vim(1)`, `vimtutor(1)`

- `i` / `a` / `o` – insert before / after / on new line; `Esc` back to Normal
- `dd` / `yy` / `p` – delete line / copy line / paste
- `x` – delete char; `u` – undo
- `/word` – search; `n` next
- `:%s/a/b/g` – replace everywhere
- `:wq` / `:q!` – save and quit / quit without saving

### 18. Partitioning and Filesystem Setup

Create partitions, then make a filesystem (ext4, xfs, etc.) on each. **Man pages:** `fdisk(8)`, `gdisk(8)`, `parted(8)`, `mkfs(8)`, `mke2fs(8)`, `tune2fs(8)`, `blkid(8)`

- `fdisk /dev/sdb` – interactive MBR partitioning
- `gdisk /dev/sdb` – GPT partitioning
- `parted /dev/sdb print` – show partitions
- `mkfs.ext4 /dev/sdb1` – create ext4
- `mkfs.xfs /dev/sdb2` – create XFS
- `tune2fs -L data /dev/sdb1` – set label
- `blkid` – show UUIDs and types

### 19. Repairing Filesystems (inode, integrity)

An inode stores file metadata; `fsck` checks integrity of an **unmounted** filesystem. **Man pages:** `fsck(8)`, `e2fsck(8)`, `debugfs(8)`, `dumpe2fs(8)`, `xfs_repair(8)`, `df(1)`

- `fsck /dev/sdb1` – check and repair
- `e2fsck -f /dev/sdb1` – force a full ext check
- `dumpe2fs -h /dev/sdb1` – show superblock info
- `debugfs /dev/sdb1` – interactive ext inspector
- `xfs_repair /dev/sdb2` – repair XFS
- `df -i` – inode usage

### 20. mount, umount, fstab

`/etc/fstab` fields: device, mount point, type, options, **dump** (backup flag 0/1), **pass** (fsck order: 1 root, 2 others, 0 skip). **Man pages:** `mount(8)`, `umount(8)`, `fstab(5)`, `findmnt(8)`, `lsof(8)`

- `mount /dev/sdb1 /mnt` – mount
- `mount -o ro /dev/sdb1 /mnt` – mount read-only
- `umount /mnt` – unmount
- `mount -a` – mount everything in fstab
- `findmnt` – tree of mounts
- `lsof /mnt` – who keeps it busy?
- fstab line: `UUID=xxxx /data ext4 defaults 0 2`

### 21. Disk Quota

Limits disk usage per user/group using soft limit (grace period) and hard limit; needs `usrquota`/`grpquota` mount options. **Man pages:** `quota(1)`, `quotacheck(8)`, `quotaon(8)`, `edquota(8)`, `repquota(8)`, `setquota(8)`

- `quotacheck -cug /data` – create quota files
- `quotaon /data` – enable quotas
- `edquota -u ali` – edit user limits
- `setquota -u ali 5000 6000 0 0 /data` – set block limits non-interactively
- `quota -u ali` – show a user's quota
- `repquota /data` – report for the filesystem

### 22. Permissions and Ownership (rwx, sticky bit)

r=4, w=2, x=1 for user/group/other. Special bits: SUID (4), SGID (2), sticky (1; only owner may delete files in the dir). **Man pages:** `chmod(1)`, `chown(1)`, `chgrp(1)`, `umask(1)`, `stat(1)`, `ls(1)`

- `chmod 755 script.sh` – rwxr-xr-x
- `chmod u+x f` – add execute for owner
- `chmod +t /shared` – set sticky bit (like `/tmp`)
- `chmod u+s prog` – set SUID
- `chown ali:dev f` – change owner and group
- `chgrp dev f` – change group only
- `umask 022` – default permission mask
- `stat f` – full metadata

### 23. Soft Link vs Hard Link

Hard link = another name for the same inode (same filesystem, no directories). Soft (symbolic) link = a small file pointing to a path; breaks if the target is removed. **Man pages:** `ln(1)`, `readlink(1)`, `stat(1)`

- `ln file hard` – hard link
- `ln -s /etc/hosts hosts.lnk` – soft link
- `ls -li` – compare inode numbers
- `readlink hosts.lnk` – show link target

### 24. Finding Files and Commands

**Man pages:** `type` (bash), `which(1)`, `whereis(1)`, `find(1)`, `locate(1)`, `updatedb(8)`

- `type ls` – alias, builtin or file?
- `which python3` – path of the executable
- `whereis ls` – binary, source and man page
- `find / -name '*.conf' -mtime -1` – by name and age
- `find / -perm -4000` – find SUID files
- `updatedb` – refresh the locate database
- `locate passwd` – fast name search

## Part B – Exam 102

### 25. Course Intro – LPIC-1 Exam 102

Exam 102 covers shells and scripting, SQL basics, X11 and desktop, administration (users, cron, time, logs, mail, printing), networking and security. No commands here.

### 26. Shells, Profiles, alias, Environment Variables

Login shells read `/etc/profile` then `~/.bash_profile`; interactive non-login shells read `~/.bashrc`. New users get files from `/etc/skel`. **Man pages:** `bash(1)`, `chsh(1)`, `shells(5)`, `alias` (builtin)

- `alias ll='ls -l'` – create alias; `unalias ll` removes it
- `source ~/.bashrc` – reload config
- `echo $SHELL` – current login shell
- `chsh -s /bin/zsh` – change login shell (must be in `/etc/shells`)
- `cat /etc/shells` – valid shells

### 27. Shell Scripting

Start with a shebang (`#!/bin/bash`), make it executable, use variables, tests, loops and functions. **Man pages:** `bash(1)`, `test(1)`, `read` and `expr(1)`, `seq(1)`

- `#!/bin/bash` – first line of a script
- `if [ -f /etc/passwd ]; then echo ok; fi` – test a file
- `for i in $(seq 1 3); do echo $i; done` – loop
- `read -p 'Name: ' n` – read input
- `greet() { echo "Hi $1"; }` – function; call with `greet Ali`
- `echo $?` – exit status of last command

### 28. SQL Basics

Relational databases store data in tables; SQL queries them. **Man pages:** `mysql(1)`, `psql(1)`, `sqlite3(1)`

- `CREATE DATABASE shop;` – new database
- `CREATE TABLE users (id INT, name VARCHAR(50));` – new table
- `INSERT INTO users VALUES (1,'Ali');` – add a row
- `SELECT * FROM users WHERE id=1;` – query
- `UPDATE users SET name='Sara' WHERE id=1;` – modify
- `DELETE FROM users WHERE id=1;` – remove
- `SELECT name FROM users ORDER BY name;` – sorting (also `GROUP BY`, `JOIN`)

### 29. X11 (X Window System)

X is client/server: the **X server** controls display, keyboard and mouse; **X clients** are applications. `DISPLAY` tells a client which server to use. **Man pages:** `X(7)`, `Xorg(1)`, `xorg.conf(5)`, `xhost(1)`, `xdpyinfo(1)`, `xrandr(1)`

- `echo $DISPLAY` – e.g. `:0`
- `xdpyinfo` – display details
- `xrandr` – resolution and monitors
- `xhost +10.0.0.5` – allow a remote client (insecure)
- `xterm -display :0` – run a client on a given display

### 30. Display / Desktop Managers

A display manager provides the graphical login (XDM, GDM, KDM/SDDM, LightDM); desktop environments are GNOME, KDE, etc. **Man pages:** `xdm(1)`, `startx(1)`, `xinit(1)`, `gdm(1)`, `lightdm(1)`

- `startx` – start X from a text console
- `systemctl restart gdm` – restart the display manager
- `systemctl set-default graphical.target` – boot into GUI
- Config: `/etc/X11/xdm/`, `/etc/lightdm/lightdm.conf`

### 31. Accessibility

Keyboard aids (StickyKeys, SlowKeys, BounceKeys, ToggleKeys, MouseKeys), screen readers and on-screen keyboards, high contrast themes. **Man pages:** `xset(1)`, `setxkbmap(1)`, `xmodmap(1)`, `xkbcomp(1)`

- `xset q` – show keyboard/screen settings
- `xset r rate 300 30` – keyboard repeat delay and rate
- `setxkbmap us` – set keyboard layout
- `setxkbmap -option grp:alt_shift_toggle us,ir` – switch layouts with Alt+Shift
- Tools to know: `orca` (screen reader), `onboard` (on-screen keyboard)

### 32. Users and Groups

Accounts live in `/etc/passwd`, hashed passwords in `/etc/shadow`, groups in `/etc/group`. **Man pages:** `useradd(8)`, `usermod(8)`, `userdel(8)`, `groupadd(8)`, `passwd(1)`, `chage(1)`, `gpasswd(1)`, `id(1)`, `passwd(5)`, `shadow(5)`, `group(5)`, `sudo(8)`

- `useradd -m -s /bin/bash ali` – create user with home
- `passwd ali` – set password
- `usermod -aG dev ali` – add to a group (keep `-a`!)
- `userdel -r ali` – delete user and home
- `groupadd dev` – create group
- `chage -l ali` – password aging info
- `id ali` – UID, GID, groups
- `getent passwd ali` – look up via NSS

### 33. Scheduled Tasks

`cron` repeats jobs, `at` runs once, `anacron` catches up missed jobs on machines that aren't always on. Cron format: `min hour day month weekday command`. **Man pages:** `crontab(1)`, `crontab(5)`, `cron(8)`, `at(1)`, `anacron(8)`, `systemd.timer(5)`

- `crontab -e` – edit your crontab; `crontab -l` lists it
- `30 2 * * * /backup.sh` – daily at 02:30
- `at now + 1 hour` – schedule one-time job (end with Ctrl+D)
- `atq` / `atrm 3` – list / remove at jobs
- Access control: `/etc/cron.allow`, `/etc/cron.deny`

### 34. Localization, Internationalization, Time Zone

Locale sets language, character set and formats (`LANG`, `LC_*`); `LC_ALL` overrides everything. **Man pages:** `locale(1)`, `localectl(1)`, `iconv(1)`, `locale-gen(8)`, `timedatectl(1)`

- `locale` – current settings
- `locale -a` – available locales
- `localectl set-locale LANG=en_US.UTF-8` – set system locale
- `iconv -f ISO-8859-1 -t UTF-8 in.txt` – convert encoding
- `LANG=C ls` – run one command with a different locale

### 35. Time, NTP, UTC

System clock (kernel) vs hardware clock (RTC); keep time synced using NTP or chrony. **Man pages:** `date(1)`, `hwclock(8)`, `timedatectl(1)`, `ntpd(8)`, `ntpq(1)`, `chronyd(8)`, `chronyc(1)`, `tzselect(8)`

- `date` – show time; `date -u` for UTC
- `hwclock --show` – read hardware clock
- `hwclock --systohc` – write system time to the RTC
- `timedatectl set-timezone Asia/Tehran` – set time zone
- `ntpq -p` – NTP peers; `chronyc sources` for chrony
- Time zone file: `/etc/localtime`

### 36. Logging

`rsyslog` (facility.priority rules) and `systemd-journald` collect logs; `logrotate` rotates them. **Man pages:** `rsyslogd(8)`, `rsyslog.conf(5)`, `logger(1)`, `journalctl(1)`, `logrotate(8)`, `last(1)`

- `tail -f /var/log/syslog` – follow the log (RHEL: `/var/log/messages`)
- `logger 'test message'` – write to syslog
- `journalctl -u sshd` – logs of one service
- `journalctl -p err` – errors only
- `last` – login history; `lastb` – failed logins
- `logrotate -f /etc/logrotate.conf` – force rotation
- Rule example: `mail.* /var/log/mail.log`

### 37. MTA (Mail Transfer Agent)

An MTA moves mail between servers over SMTP (Postfix, Sendmail, Exim, qmail). **Man pages:** `postfix(1)`, `sendmail(8)`, `mailq(1)`, `newaliases(1)`, `aliases(5)`, `mail(1)`

- `mailq` – show the mail queue
- `echo 'hi' | mail -s Test ali` – send local mail
- `newaliases` – rebuild `/etc/aliases` database
- `postfix reload` – reload Postfix config
- `~/.forward` – per-user forwarding

### 38. Printing (CUPS)

CUPS is the standard print system (web UI on port 631, config in `/etc/cups/`). **Man pages:** `cupsd(8)`, `lp(1)`, `lpr(1)`, `lpq(1)`, `lprm(1)`, `lpstat(1)`, `lpadmin(8)`, `cupsenable(8)`, `cupsaccept(8)`

- `lpstat -p` – list printers
- `lp -d office file.pdf` – print to a printer
- `lpq` – show queue
- `lprm 12` – cancel job 12
- `lpadmin -p office -E -v ipp://host/ipp` – add a printer
- `cupsdisable office` / `cupsenable office` – stop / start printer

### 39. IPv4 Addressing

32-bit address in four octets, plus a netmask/CIDR (e.g. `192.168.1.10/24`). Private ranges: `10/8`, `172.16/12`, `192.168/16`. **Man pages:** `ip(8)`, `ipcalc(1)`, `ping(8)`

- `ipcalc 192.168.1.10/24` – network, broadcast, range
- `ip -4 addr` – show IPv4 addresses
- `ping -c 3 8.8.8.8` – test reachability

### 40. Network Interface Configuration

Set IP, gateway and DNS; persist in distro files (`/etc/network/interfaces`, NetworkManager, `ifcfg-*`). **Man pages:** `ip(8)`, `ifconfig(8)`, `route(8)`, `ifup(8)`, `hostname(1)`, `hosts(5)`, `resolv.conf(5)`, `nmcli(1)`

- `ip addr add 192.168.1.50/24 dev eth0` – add IP
- `ip link set eth0 up` – bring interface up (`ifup eth0` also)
- `ip route add default via 192.168.1.1` – default gateway
- `ifconfig eth0` – legacy view
- `hostnamectl set-hostname srv1` – set hostname
- `nmcli device status` – NetworkManager view
- Files: `/etc/hosts`, `/etc/resolv.conf`

### 41. Basic Network Troubleshooting

Work bottom-up: link → IP → gateway → DNS → service. **Man pages:** `ping(8)`, `traceroute(8)`, `tracepath(8)`, `netstat(8)`, `ss(8)`, `dig(1)`, `host(1)`, `nc(1)`

- `ping -c 3 host` – connectivity
- `traceroute host` – path hop by hop
- `ss -tulpn` – listening ports (modern `netstat -tulpn`)
- `dig example.com` – full DNS query
- `host example.com` – short DNS lookup
- `nc -zv host 80` – test a TCP port
- `ip route` – routing table

### 42. DNS Client Settings (nsswitch.conf)

`/etc/nsswitch.conf` decides lookup order (e.g. `hosts: files dns`); `/etc/hosts` is checked first, then DNS servers in `/etc/resolv.conf`. **Man pages:** `nsswitch.conf(5)`, `resolv.conf(5)`, `hosts(5)`, `getent(1)`, `dig(1)`, `host(1)`

- `getent hosts example.com` – resolve using the NSS order
- `grep hosts /etc/nsswitch.conf` – see lookup order
- `echo 'nameserver 8.8.8.8' >> /etc/resolv.conf` – add DNS server
- `dig @8.8.8.8 example.com` – query a specific server

### 43. Basic Security Administration

Find risky files, limit privileges, audit open ports and logins. **Man pages:** `sudo(8)`, `sudoers(5)`, `su(1)`, `ulimit` (bash), `nmap(1)`, `ss(8)`, `lsof(8)`, `fuser(1)`, `nologin(8)`, `last(1)`

- `find / -perm -4000 -type f` – SUID files
- `sudo -l` – what may I run as root?
- `visudo` – safely edit sudoers
- `ulimit -u 100` – cap processes per user (shell)
- `ss -tulpn` / `lsof -i :22` – open ports and owners
- `nmap localhost` – scan own ports
- `fuser -v /var/log/syslog` – who uses a file
- `usermod -s /sbin/nologin ali` – block interactive login

### 44. Host Security Hardening

Disable unneeded services, restrict access, use strong passwords and keep patches current. **Man pages:** `systemctl(1)`, `xinetd(8)`, `hosts_access(5)`, `passwd(1)`, `chage(1)`, `sshd_config(5)`, `iptables(8)`

- `systemctl disable --now telnet.socket` – stop and disable a service
- `/etc/hosts.allow`, `/etc/hosts.deny` – TCP wrappers, e.g. `sshd: 10.0.0.0/24`
- `passwd -l ali` – lock an account
- `chage -M 90 ali` – password expires after 90 days
- `PermitRootLogin no` in `sshd_config` – block root SSH
- `touch /etc/nologin` – block non-root logins temporarily

### 45. Encryption (SSH, GPG)

SSH encrypts remote access with key pairs; GPG encrypts and signs files; SSH tunnels protect other traffic. **Man pages:** `ssh(1)`, `ssh-keygen(1)`, `ssh-copy-id(1)`, `ssh-agent(1)`, `ssh-add(1)`, `scp(1)`, `sshd_config(5)`, `gpg(1)`

- `ssh-keygen -t ed25519` – create a key pair
- `ssh-copy-id ali@host` – install public key on server
- `ssh ali@host` – log in
- `scp file ali@host:/tmp` – copy over SSH
- `ssh -L 8080:localhost:80 ali@host` – local port-forward tunnel
- `ssh-agent` + `ssh-add` – cache key passphrase
- `gpg --gen-key` – create a GPG key
- `gpg -c file` – symmetric encryption; `gpg -d file.gpg` decrypts
- Key files: `~/.ssh/authorized_keys`, `~/.ssh/known_hosts`

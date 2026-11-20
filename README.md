# Seagate GoFlex Home → Debian NAS with Samba and Tailscale

A step-by-step guide to giving the **Seagate GoFlex Home** (STAM1000100 and similar) a second life. Seagate has abandoned the device: its remote-access service (Seagate Share) was shut down in 2018, and its Flash-based web interface stopped working when Flash reached end of life in 2020.

At the end you will have:

- a **modern U-Boot** (2017.07, by bodhi / doozan forum) in NAND, with **netconsole**;
- **Debian 12** (bodhi's Kirkwood rootfs) booting from a USB stick;
- the SATA drive serving **per-user Samba shares**;
- **remote access** through **Tailscale**, with no port forwarding (it works behind carrier-grade NAT).

Everything was done **over SSH, without a serial cable**. Having a serial adapter on hand is still strongly recommended (see [Recovery](#recovery)).

> ⚠️ **Warning.** Flashing a bootloader can brick your device. Read the whole guide before you start, and proceed at your own risk. If **any** error appears while writing to flash, **do not reboot or power off**: as long as the original system is still running, you can still fix things.

This guide does not replace the original documentation. It gathers in one place what is spread across several forum threads and wiki pages, adds the pitfalls met in a real install, and credits every source in [Credits and sources](#credits-and-sources).

---

## Contents

1. [Hardware and performance](#hardware-and-performance)
2. [What you need](#what-you-need)
3. [Phase 0 — SSH into the Seagate firmware and back up](#phase-0--ssh-into-the-seagate-firmware-and-back-up)
4. [Phase 1 — Prepare the Debian USB stick](#phase-1--prepare-the-debian-usb-stick)
5. [Phase 2 — Flash the new U-Boot](#phase-2--flash-the-new-u-boot)
6. [Phase 3 — Basic Debian setup](#phase-3--basic-debian-setup)
7. [Phase 4 — Data drive](#phase-4--data-drive)
8. [Phase 5 — Samba](#phase-5--samba)
9. [Phase 6 — Tailscale](#phase-6--tailscale)
10. [Known pitfalls](#known-pitfalls)
11. [Maintenance](#maintenance)
12. [Recovery](#recovery)
13. [Alternatives](#alternatives)
14. [Credits and sources](#credits-and-sources)

### Placeholders used in this guide

| Placeholder | Meaning | Example |
|---|---|---|
| `<GOFLEX_IP>` | Current LAN IP of the GoFlex | `192.168.1.155` |
| `<UBOOT_IP>` | A **free** static IP that U-Boot uses at boot | `192.168.1.250` |
| `<PC_IP>` | IP of the computer that receives netconsole | `192.168.1.72` |
| `<ACCOUNT>` | User name from the original Seagate web interface | `john` |
| `<PRODUCT_KEY>` | Product Key on the label under the dock (`XXXX-XXXX-XXXX-XXXX`) | — |
| `<MAC>` | MAC address from the label, as `00:10:75:XX:XX:XX` | — |
| `<USER>` | Main Linux/Samba user (uid 1000) | `john` |
| `<DISK_UUID>` | UUID of the data partition (`blkid`) | — |

---

## Hardware and performance

| Component | Value |
|---|---|
| SoC | Marvell Kirkwood 88F6281 (ARMv5TE, 1.2 GHz) |
| RAM / NAND | 128 MB / 256 MB |
| Network | 1× Gigabit Ethernet |
| Ports | 1× SATA (in the dock), 1× USB 2.0 |

Specifications from the OpenWrt Table of Hardware page [[1]](#ref-1).

Measured on this install, with a 1.5 TB NTFS drive:

| Operation | Speed |
|---|---|
| Local read, `ntfs3` (in-kernel driver) | ~100 MB/s |
| Local read, `ntfs-3g` (FUSE) | ~26 MB/s |
| Write over Samba from a Mac | ~20 MB/s |

With `ntfs3`, the CPU becomes the bottleneck rather than the disk. Reformatting an NTFS drive that already holds data brings little benefit.

---

## What you need

- The GoFlex Home, wired to your network.
- A USB stick (8–16 GB is plenty). It will be **erased**.
- A Linux system to prepare the stick: a VM (UTM, VirtualBox), a Raspberry Pi or a live USB. macOS will not do, because of ext3 and file permissions.
- A computer with a terminal and `ssh`/`scp`.
- *(Recommended)* a **3.3 V** USB-TTL serial adapter (CP2102 or FT232) for emergencies.

### Downloads

Download everything **on a modern computer**: the Seagate firmware's `wget` cannot fetch over modern HTTPS. **Always** check the hashes.

| File | Source | md5 |
|---|---|---|
| `uboot.2017.07-tld-1.goflexhome.bodhi.tar` | bodhi, U-Boot thread [[2]](#ref-2) | `b014dcf14697d573ca9626fbc6a6c489` |
| `uboot.2016.05-tld-1.environment.bodhi.tar` | bodhi, U-Boot thread [[2]](#ref-2) | `3823eef10011b864859d31a76470e0e3` |
| `linux-tools-installation-bodhi.tar.gz` | bodhi, flashing utilities thread [[3]](#ref-3) | `e58f442411eb35e641d40ea0577e00ff` |
| `Debian-6.5.7-kirkwood-tld-1-rootfs-bodhi.tar.bz2` | bodhi, kernel & rootfs thread [[4]](#ref-4) | `398aff3717b0d739e82815650b5c7f81` |

Direct Dropbox links (as of November 2026; always check the threads for current versions):

```
https://www.dropbox.com/s/klksl0vawkhie9r/uboot.2017.07-tld-1.goflexhome.bodhi.tar?dl=1
https://www.dropbox.com/scl/fi/bau404owcfxysok/uboot.2016.05-tld-1.environment.bodhi.tar?rlkey=oda7yvw38624n4bb8v0hfudas&dl=1
https://bitly.com/1RdYW8S        (redirects to linux-tools-installation-bodhi.tar.gz)
https://www.dropbox.com/scl/fi/t2zv6g1sydq019urfnsd6/Debian-5.6.7-kirkwood-tld-1-rootfs-bodhi.tar.bz2?rlkey=1uukhwhpmccvjcv8msnqbqkbd&dl=1
```

```bash
curl -L -o uboot.2017.07-tld-1.goflexhome.bodhi.tar "<link>"
md5sum *.tar *.tar.gz *.tar.bz2      # on macOS: md5
```

Notes:
- `dl=1` at the end of the link makes Dropbox serve the file instead of the web page.
- The rootfs is published under a mistyped name ("5.6.7"). bodhi notes this himself [[4]](#ref-4). Rename it to `Debian-6.5.7-…`.

---

## Phase 0 — SSH into the Seagate firmware and back up

### Log in over SSH

The original firmware keeps SSH enabled. The user name has this format:

```
<ACCOUNT>_hipserv2_seagateplug_<PRODUCT_KEY>
```

The password is that account's password. The `root`/`stxadmin` login listed in some guides (e.g. [[1]](#ref-1)) had been disabled by Seagate's firmware updates on this unit. Modern SSH clients must re-enable the legacy algorithms:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa,ssh-dss \
    -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    <ACCOUNT>_hipserv2_seagateplug_<PRODUCT_KEY>@<GOFLEX_IP>
sudo -s          # same password
```

### Look around (read-only)

```bash
uname -a
cat /proc/mtd
dmesg | grep -i bad
```

Typical stock values: kernel 2.6.22; partitions `mtd0` 1 MB (u-boot), `mtd1` 6 MB (uImage) and `mtd2` 249 MB (root).

> 🛑 If `dmesg` shows any **"Bad eraseblock"** in the first 1 MB (blocks 0–7, up to address 0xE0000), **do not flash U-Boot**. This check comes from bodhi's instructions [[2]](#ref-2). Ask on the doozan forum first.

### Back up the original U-Boot and kernel

```bash
dd if=/dev/mtd0 of=/tmp/mtd0.img      # must be 1048576 bytes
dd if=/dev/mtd1 of=/tmp/mtd1.img      # must be 6291456 bytes
md5sum /tmp/mtd*.img
chmod 644 /tmp/mtd*.img
strings /tmp/mtd0.img | grep -i "u-boot"     # stock U-Boot version (here: 1.1.4, Marvell 3.4.27)
```

On your computer, copy the files and check that the md5 hashes match:

```bash
scp -O -o HostKeyAlgorithms=+ssh-rsa,ssh-dss \
       -o KexAlgorithms=+diffie-hellman-group1-sha1 \
       '<ACCOUNT>_hipserv2_seagateplug_<PRODUCT_KEY>@<GOFLEX_IP>:/tmp/mtd*.img' .
```

The GoFlex's `/tmp` lives in RAM and is lost at power-off. Keep a second copy of the backups somewhere safe.

---

## Phase 1 — Prepare the Debian USB stick

This follows bodhi's rootfs instructions [[4]](#ref-4). On Linux, **as root**:

```bash
lsblk                                   # identify the stick CAREFULLY
umount /dev/sdX1 2>/dev/null
fdisk /dev/sdX                          # o, n, p, 1, Enter, Enter, w
mkfs.ext3 -L rootfs /dev/sdX1           # the "rootfs" label is required
mount /dev/sdX1 /mnt && cd /mnt
tar xjf /path/to/Debian-6.5.7-kirkwood-tld-1-rootfs-bodhi.tar.bz2
ls /mnt/boot/dts | grep goflex          # kirkwood-goflexhome.dtb must be there
ls -lh /mnt/boot/uImage /mnt/boot/uInitrd
cd / && sync && umount /mnt
```

> The step in bodhi's guide that appends the DTB to `uImage` (`zImage.fdt`) is only for people **keeping** the stock U-Boot. **Skip it** here.

The rootfs's root password is `root` [[4]](#ref-4).

---

## Phase 2 — Flash the new U-Boot

Adapted from bodhi's flashing instructions [[2]](#ref-2), with the tools from [[3]](#ref-3).

### 2.1 Copy the files and set up the tools

Copy the three tarballs (U-Boot, environment, tools) to the GoFlex's `/tmp` with `scp -O` as above. Then, on the GoFlex, as root:

```bash
cd /tmp
tar xzf linux-tools-installation-bodhi.tar.gz
cd /tmp/tools && ln -s fw_printenv fw_setenv && cp -a fw_env.config /etc/ && cd /tmp
tar xf uboot.2017.07-tld-1.goflexhome.bodhi.tar       # .kwb is 524288 bytes
tar xf uboot.2016.05-tld-1.environment.bodhi.tar
/tmp/tools/fw_printenv
```

A `Warning: Bad CRC` at this point is **expected**: the stock U-Boot keeps its environment somewhere else.

### 2.2 Flash U-Boot and verify

```bash
cd /tmp/tools
./flash_erase /dev/mtd0 0 4
./nandwrite /dev/mtd0 /tmp/uboot.2017.07-tld-1.goflexhome.mtd0.kwb
./nanddump --noecc --omitoob -l 0x80000 -f /tmp/verify.kwb /dev/mtd0
md5sum /tmp/verify.kwb /tmp/uboot.2017.07-tld-1.goflexhome.mtd0.kwb
```

**The two hashes must match.** If they don't, do not reboot, and ask for help. The read-back check is an extra safety step, not part of bodhi's instructions.

### 2.3 Flash the default environment (at 0xC0000)

```bash
./flash_erase /dev/mtd0 0xc0000 1
./nandwrite -s 786432 /dev/mtd0 /tmp/uboot.2016.05-tld-1.environment.img
./fw_printenv | head          # "Bad CRC" should be gone
```

### 2.4 MAC, DTB and netconsole

The netconsole variables come from bodhi's U-Boot thread [[2]](#ref-2) and the original netconsole thread by Jeff Doozan [[5]](#ref-5).

```bash
./fw_setenv ethaddr '<MAC>'
./fw_setenv dtb_file '/boot/dts/kirkwood-goflexhome.dtb'
./fw_setenv preboot_nc 'setenv nc_ready 0; for pingstat in 1 2 3 4 5; do; sleep 1; if run if_netconsole; then setenv nc_ready 1; fi; done; if test $nc_ready -eq 1; then run start_netconsole; fi'
./fw_setenv preboot 'run preboot_nc'
./fw_setenv ipaddr '<UBOOT_IP>'
./fw_setenv serverip '<PC_IP>'
./fw_printenv                 # review everything before rebooting
```

netconsole only starts if the GoFlex can ping `<PC_IP>`. On a Mac, turn off the firewall's "stealth mode".

> The stock U-Boot has no netconsole support. You only get it after installing the new U-Boot, as bodhi explains in [[5]](#ref-5). So set netconsole up **right after flashing, before the first reboot**.

### 2.5 First boot

On your computer, start listening. The BSD `nc` shipped with macOS tends to misbehave with netconsole; `socat`, suggested by forum users in [[5]](#ref-5), works well:

```bash
socat STDIO UDP-LISTEN:6666           # macOS: brew install socat
```

On the GoFlex:

```bash
sync
/sbin/poweroff                        # poweroff is not in PATH on the stock firmware
```

Plug the stick into the USB port and power the unit back on. netconsole should show `U-Boot 2017.07-tld-1 … GoFlex Home`, then `Starting kernel …`, after which it goes quiet (expected: netconsole only covers U-Boot). After 2–3 minutes:

```bash
ssh root@debian.local                 # password: root
```

> From now on the stick is the **system disk**. Leave it plugged in.

---

## Phase 3 — Basic Debian setup

bodhi's rootfs uses **sysvinit** [[4]](#ref-4): there is no `systemctl`. Use `service <name> start|stop|restart`.

```bash
passwd                                         # change the root password
dpkg-reconfigure tzdata
dd if=/dev/zero of=/swapfile bs=1M count=512 && chmod 600 /swapfile
mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
apt-get update && apt-get upgrade
apt-get install -y fake-hwclock                # no battery-backed RTC
echo 'export LC_ALL=C.UTF-8' >> /root/.profile # silences locale warnings sent by the SSH client
```

bodhi strongly recommends a swap file and `fake-hwclock`, and warns about regenerating `uInitrd` after upgrades [[4]](#ref-4).

### ⚠️ Regenerate `uInitrd` automatically

U-Boot boots `/boot/uInitrd`, which has to be **rebuilt with `mkimage` every time `update-initramfs` runs**: on upgrades, and when installing packages such as `fuse3`. Forget it and the box may not boot. bodhi gives the `mkimage` command [[4]](#ref-4); this hook runs it automatically:

```bash
mkdir -p /etc/initramfs/post-update.d
cat > /etc/initramfs/post-update.d/uinitrd << 'EOF'
#!/bin/sh
# Rebuild /boot/uInitrd for U-Boot whenever the initramfs changes
# $1 = kernel version, $2 = path to the initrd
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 \
  -n "initramfs-$1" -d "$2" /boot/uInitrd
EOF
chmod +x /etc/initramfs/post-update.d/uinitrd
update-initramfs -u && ls -lh /boot/uInitrd
```

`uInitrd` must stay **under 11 MB**, the limit set by the default environment [[2]](#ref-2) [[4]](#ref-4). If yours is bigger, see "Load addresses for large systems" in [[2]](#ref-2).

---

## Phase 4 — Data drive

Put the SATA drive back in the dock and identify it:

```bash
lsblk -o NAME,SIZE,MODEL,LABEL,FSTYPE
blkid
```

### Option A — NTFS drive with existing data (`ntfs3` driver)

`ntfs3` is the in-kernel NTFS driver [[6]](#ref-6). bodhi's kernel ships it as a module (`CONFIG_NTFS3_FS=m`).

```bash
apt-get install -y ntfs-3g                 # provides ntfsfix
ntfsfix -d /dev/sda1                       # only if ntfs3 reports "volume is dirty"
adduser <USER>                             # should get uid 1000
mkdir -p /srv/data
echo 'UUID=<DISK_UUID> /srv/data ntfs3 defaults,nofail,noatime,uid=1000,gid=1000,umask=022 0 0' >> /etc/fstab
mount -a && df -h /srv/data
```

`ntfsfix` only clears the "dirty" flag and does basic repairs. Run a full `chkdsk` on Windows when you get the chance.

### Option B — Empty drive (ext4)

```bash
mkfs.ext4 -L data /dev/sda1
mkdir -p /srv/data
echo 'LABEL=data /srv/data ext4 defaults,nofail,noatime 0 2' >> /etc/fstab
mount -a
```

### Folders and ownership

```bash
cd /srv/data && mkdir -p Person1 Person2 Shared
find /srv/data/Person1 /srv/data/Person2 /srv/data/Shared -user root \
     -exec chown <USER>:<USER> {} +
```

> With `ntfs3`, anything created as root is stored as owned by root, and the `uid=` mount option only applies to older files. Without the `chown`, Samba can't write.

To move existing data into a folder, **chain commands with `&&`**, so that nothing runs in the wrong place if `cd` fails:

```bash
cd /srv/data && for x in *; do case "$x" in Person1|Person2|Shared|'$RECYCLE.BIN'|'System Volume Information') ;; *) mv "$x" Shared/ ;; esac; done
```

---

## Phase 5 — Samba

Since NTFS keeps no per-user Unix permissions, Samba does the access control (`valid users`) and always writes as the disk owner (`force user`) [[7]](#ref-7).

```bash
adduser --no-create-home --shell /usr/sbin/nologin person2
apt-get install -y samba
smbpasswd -a <USER>
smbpasswd -a person2

cat >> /etc/samba/smb.conf << 'EOF'

[Person1]
   path = /srv/data/Person1
   valid users = <USER>
   read only = no
   force user = <USER>

[Person2]
   path = /srv/data/Person2
   valid users = person2
   read only = no
   force user = <USER>

[Shared]
   path = /srv/data/Shared
   valid users = <USER> person2
   read only = no
   force user = <USER>
EOF

testparm -s && service smbd restart
```

> **Don't** use `veto files = ._*`: the macOS Finder needs to create those AppleDouble files and refuses to copy if it can't.

Connect with Finder → Cmd+K → `smb://<GOFLEX_IP>`, or `\\<GOFLEX_IP>` on Windows.

---

## Phase 6 — Tailscale

Tailscale publishes no Debian package for **armel**, and a request for ARMv5 builds was closed [[8]](#ref-8). The official **static "arm" binary** [[9]](#ref-9), however, has been reported to run on armel Kirkwood boxes [[10]](#ref-10), and it ran on this unit (v1.104.7). If yours fails with `Illegal instruction`, build it yourself with `CGO_ENABLED=0 GOOS=linux GOARCH=arm GOARM=5` [[11]](#ref-11) [[12]](#ref-12).

`/tmp` is only ~50 MB, too small to unpack Tailscale, so work on the stick:

```bash
mkdir -p /root/ts && cd /root/ts
V=$(wget -qO- https://pkgs.tailscale.com/stable/ | grep -o 'tailscale_[0-9.]*_arm.tgz' | head -1)
wget "https://pkgs.tailscale.com/stable/$V" && tar xzf "$V"
cp tailscale_*_arm/tailscale tailscale_*_arm/tailscaled /usr/local/bin/
tailscale version

mkdir -p /var/lib/tailscale /run/tailscale
tailscaled --state=/var/lib/tailscale/tailscaled.state \
           --tun=userspace-networking > /var/log/tailscaled.log 2>&1 &
tailscale up --hostname=goflex             # open the login link it prints
```

`userspace-networking` mode [[13]](#ref-13) uses less memory and is enough to serve the box's own Samba shares.

**Start at boot:** add this to `/etc/rc.local`, **before** `exit 0`:

```bash
mkdir -p /run/tailscale; /usr/local/bin/tailscaled --state=/var/lib/tailscale/tailscaled.state --tun=userspace-networking > /var/log/tailscaled.log 2>&1 &
```

In the Tailscale admin console: **Machines → goflex → Disable key expiry** [[14]](#ref-14). Otherwise the box drops off the tailnet when its key expires (180 days by default).

On each client, install the Tailscale app, join the same tailnet and connect to `smb://goflex` or `smb://100.x.y.z`. The same address works at home and away. On Android, use a file manager with SMB support (e.g. Material Files, CX File Explorer).

---

## Known pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| SSH closes right away | Wrong user name | `<ACCOUNT>_hipserv2_seagateplug_<PRODUCT_KEY>` |
| Client refuses SSH | 2010-era algorithms | `-o HostKeyAlgorithms=… -o KexAlgorithms=…`; `scp -O` |
| `poweroff: command not found` | Not in PATH on stock firmware | `/sbin/poweroff` |
| `wget` can't download from Dropbox | No modern HTTPS on stock firmware | Download elsewhere, copy with `scp` |
| Commands ran in the wrong directory | `cd` failed, the rest ran anyway | Chain with `&&` |
| `tar: No space left on device` | `/tmp` is a small tmpfs | Work under `/root` |
| `systemctl: command not found` | The rootfs uses sysvinit | `service …`; `/etc/rc.local` |
| NTFS mounts read-only | Volume is "dirty" | `ntfsfix -d /dev/sdX1` |
| Finder can't copy | Root-owned folders / `veto files = ._*` | `chown`; don't veto `._*` |
| Date back to 1970 after reboot | No RTC | `fake-hwclock` + NTP |
| `locale` warnings | Client sends `LC_CTYPE=UTF-8` | `export LC_ALL=C.UTF-8` |
| Won't boot after an upgrade | Stale `uInitrd` | `post-update.d/uinitrd` hook |
| No netconsole output | PC firewall blocks ping | Allow ICMP / turn off stealth mode |

---

## Maintenance

- Run `apt-get update && apt-get upgrade` regularly. Before rebooting, check the timestamp of `/boot/uInitrd`.
- Always shut down with `poweroff`. Pulling the plug can leave NTFS "dirty".
- The static Tailscale binary **does not update itself**: repeat the download to upgrade.
- Newer bodhi kernels (7.x series) install with `dpkg` and `mkimage`, as described in [[4]](#ref-4).

## Recovery

1. **U-Boot shows on netconsole but Debian doesn't boot:** the problem is on the stick (`uInitrd`, `fstab`, label). Fix it on a Linux machine.
2. **Not even U-Boot shows:** only the **serial console** helps, using UART boot with `kwboot` [[2]](#ref-2) [[3]](#ref-3). The header is **J1** on the board: pin 8 Rx, pin 9 Tx, pin 10 GND, **3.3 V**, 115200 8N1 [[1]](#ref-1). Do not connect VCC.
3. The `mtd0.img` saved in Phase 0 lets you restore the stock U-Boot.

## Alternatives

- **OpenWrt** supports the GoFlex Home and provides factory/sysupgrade images, but the documented install from stock requires a serial cable and TFTP [[1]](#ref-1).
- If you only want the files, take the SATA drive out of the dock and read it from any computer. A USB-SATA adapter is enough.

---

## Credits and sources

This guide would not exist without the work of **bodhi** and the **doozan forum** community, which has maintained modern U-Boot builds, kernels and Debian root filesystems for Kirkwood devices for over a decade. Thanks also to **Jeff Doozan**, who started the forum, the original Dockstar/GoFlex U-Boot work and the netconsole method, and to **davygravy** for the UART boot (`kwboot`) instructions referenced by bodhi. The OpenWrt contributors who documented the GoFlex Home (credited on the wiki page and in the commit history) provided the hardware details and the serial pinout.

If this guide helped you, consider thanking bodhi on the forum ("bodhi's corner" in his signature).

<a id="ref-1"></a>[1] OpenWrt Wiki — *Seagate GoFlexHome* (Table of Hardware): specs, flash layout, serial pinout, OpenWrt install via serial/TFTP. <https://openwrt.org/toh/seagate/goflexhome>

<a id="ref-2"></a>[2] bodhi — *2017.07–2026.04 U-Boot Kirkwood* (doozan forum): U-Boot images, md5 hashes, flashing instructions, bad-block check, default environment, netconsole, load addresses. <https://forum.doozan.com/read.php?3,12381>

<a id="ref-3"></a>[3] bodhi — *U-Boot flashing utilities and kwboot binary* (doozan forum): the `linux-tools-installation-bodhi.tar.gz` binpack (`flash_erase`, `nandwrite`, `nanddump`, `fw_printenv`, `fw_env.config`) for use on the stock OS, and `kwboot`. <https://forum.doozan.com/read.php?3,27280>

<a id="ref-4"></a>[4] bodhi — *Linux Kernel Kirkwood package and Debian rootfs* (doozan forum): the Debian 12 rootfs, its install steps, swap, `fake-hwclock`, `uInitrd` regeneration and size limit, sysvinit. <https://forum.doozan.com/read.php?2,12096>

<a id="ref-5"></a>[5] Jeff Doozan et al. — *Use netconsole to troubleshoot uBoot without a serial cable* (doozan forum): netconsole method, `nc`/`socat` usage, and bodhi's note that the stock GoFlex Home U-Boot has no netconsole. <https://forum.doozan.com/read.php?3,14>

<a id="ref-6"></a>[6] Linux kernel documentation — *NTFS3*. <https://docs.kernel.org/filesystems/ntfs3.html>

<a id="ref-7"></a>[7] Samba — *smb.conf(5)* manual (`valid users`, `force user`, `veto files`). <https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html>

<a id="ref-8"></a>[8] Tailscale — GitHub issue #676, *ARM5 support?* <https://github.com/tailscale/tailscale/issues/676>

<a id="ref-9"></a>[9] Tailscale — *Install Tailscale with static binaries*. <https://tailscale.com/kb/1053/install-static>

<a id="ref-10"></a>[10] Tailscale forum — *How to build a Debian package* (report of the static "arm" binary running on an armel Pogoplug). <https://forum.tailscale.com/t/how-to-build-a-debian-package/240>

<a id="ref-11"></a>[11] Go wiki — *GoArm* (ARMv5 support with `GOARM=5`). <https://go.dev/wiki/GoArm>

<a id="ref-12"></a>[12] MikroTik forum — *Working Tailscale container on hEX S (arm32v5)* (building Tailscale with `GOARM=5`). <https://forum.mikrotik.com/t/working-tailscale-container-on-hex-s-en7562ct-arm32v5-54-mbit-s-exit-node/272193>

<a id="ref-13"></a>[13] Tailscale — *Userspace networking mode*. <https://tailscale.com/kb/1112/userspace-networking>

<a id="ref-14"></a>[14] Tailscale — *Key expiry*. <https://tailscale.com/kb/1028/key-expiry>

The doozan forum may block automated access. Open the threads in a normal browser.

---

*Written from a real install in November 2026. Versions and download links change, so always check the original threads. Corrections and contributions are welcome via issues or pull requests.*

## License

Text of this guide: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). The U-Boot, kernel, rootfs and tools referenced here belong to their respective authors and are distributed by them under their own licenses. This repository does not redistribute them.

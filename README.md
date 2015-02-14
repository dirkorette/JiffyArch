# Einleitung

Diese Anleitung beschreibt die Installation von [Arch Linux](https://www.archlinux.org/) auf einer
domainFactory [JiffyBox](http://www.df.eu/de/cloud-hosting/cloud-server/).

Für diese Anleitung wurde der kleinste *JiffyBox* Tarif (*CloudLevel 1*)
verwendet. Als Basisbetriebssystem kommt *Debian* 7 (64-bit) zum Einsatz.


# Partitionierung

Im Kundenmenü wird das bestehende Partitionslayout wie folgt geändert:

*   Die *Debian Wheezy* (7) 64-Bit Partition wird auf 5 GB verkleinert.
*   *arch-boot* als ext4 mit 200MB wird neu angelegt.
*   *arch-hd* als ext4 mit dem restlichen verfügbaren Speicherplatz
    wird neu angelegt.

Dann werden die beiden neuen angelegten Festplatten dem Standardprofil
hinzugefügt. Die Festplatten-Zuordnung sieht dann so aus:

*   ```/dev/xvda``` -> *Debian Wheezy* (7) 64-Bit (5.00, ext4)
*   ```/dev/xvdb``` -> *Swap* (0.50, swap)
*   ```/dev/xvdc``` -> *arch-boot* (0.20, ext4)
*   ```/dev/xvdd``` -> *arch-hd* (69.30, ext4)

Nun kann die Jiffybox das erste Mal gestartet werden.


# Systemaktualisierung

Zunächst wird eine Systemaktualisierung vorgenommen:

```bash
apt-get update && apt-get upgrade
```


# *Arch Linux* Root-Image

Zunächst wird das Root-Image von einem *Arch Linux* Spiegelserver
heruntergeladen:

```bash
curl -LO http://ftp.hosteurope.de/mirror/ftp.archlinux.org/iso/latest/arch/x86_64/airootfs.sfs
```

Als nächstes müssen die ```squashfs-tools``` heruntergeladen werden, um
das Root-Image zu "entpacken", damit es man es später einbinden kann:

```bash
apt-get install squashfs-tools
```

Das Root-Image wird dann ausgepackt und eingebunden:

```bash
unsquashfs -d /squashfs-root airootfs.sfs
mkdir /arch
mount -o loop /squashfs-root/airootfs.img /arch
```

Bevor man in das eingebundene Root-Image "hineingeht", müssen zunächst
weitere Sachen eingebunden werden:

```bash
mount -t proc none /arch/proc
mount -t sysfs none /arch/sys
mount -o bind /dev /arch/dev
mount -o bind /dev/pts /arch/dev/pts
cp -L /etc/resolv.conf /arch/etc
```

Danach kann das *Arch Linux* Root-Image betreten werden:

```bash
chroot /arch /bin/bash
```


# Einbinden der Festplatten

Jetzt werden die neu angelegten Festplatten eingebunden:

```bash
mount /dev/xvdd /mnt
mkdir /mnt.archboot
mount /dev/xvdc /mnt.archboot
mkdir /mnt.archboot/boot
mkdir /mnt/boot
mount -o bind /mnt.archboot/boot /mnt/boot
mkdir /run/shm
```

Nun werden Schlüssel importiert und geprüft für die Arch Linux
Paketverwaltung:

```bash
time pacman-key --init
pacman-key --populate archlinux
```

Der erste Befehl dauert rund 11 Minuten.


# Installation der Basispakete

Jetzt werden die nötigen Basispakete installiert:

```bash
pacstrap /mnt base base-devel
```


# Schreiben der fstab

```bash
cat << EOF > /mnt/etc/fstab
/dev/xvdb / ext4 defaults 0 0
/dev/xvda /mnt.archboot ext4 defaults 0 0
/mnt.archboot/boot /boot none bind defaults 0 0
proc /proc proc defaults 0 0
dev /dev tmpfs rw 0 0
EOF
```

*Hinweis*: Später wird ein neues Profil für das Starten der JiffyBox
angelegt. Dort sind nur die Festplatten *arch-boot* und *arch-hd*
eingetragen. Deswegen lauten die Einträge in fstab dann auch
```/dev/xvda``` bzw. ```/dev/xvdb```.


# arch-chroot

Jetzt wird in die eigentliche *Arch Linux* Installation gewechselt.
Dies geschieht mit dem Befehl:

```bash
arch-chroot /mnt /bin/bash
```


# Hostname

Der gewünschte Hostname kann wie folgt gesetzt werden:

```bash
echo "jiffyarch" > /etc/hostname
```


# Internationalisierung

*Locale* und Zeitzone werden gesetzt:

```bash
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
echo "LANG=de_DE.UTF-8" > /etc/locale.conf
echo KEYMAP=de > /etc/vconsole.conf
echo "de_DE.UTF-8 UTF-8" > /etc/locale.gen
locale-gen
```


# Kernelerzeugung
Die Arch Linux JiffyBox soll später einen eigenen Kernel verwenden.
Dazu müssen folgende Einstellung gemacht werden:

```bash
sed -i 's/^\(Compression=.*\)/#\1/g' /etc/mkinitcpio.conf
sed -i 's/^\(MODULES=\)"\(.*\)"/\1"xen-blkfront xen-fbfront xen-netfront xen-kbdfront"/g' /etc/mkinitcpio.conf
echo 'COMPRESSION="cat"' >> /etc/mkinitcpio.conf
echo "h0:2345:respawn:/sbin/agetty -8 -s 38400 hvc0 linux" > /etc/initab
mkinitcpio -p linux
```

Die Erzeugung sollte erfolgreich abschließen mit *Image generation successful".


# Grub

Als Bootloader wird *Grub2* verwendet, der zunächst installiert werden
muss:

```bash
pacman -Sy grub
```

Danach wird der entsprechende Menüeintrag für *Arch Linux* vorbereitet:

```bash
cat << EOF > /boot/grub/menu.lst
timeout 1
default 0

title Arch Linux
root (hd0)
kernel /boot/vmlinuz-linux root=/dev/xvdb ro
initrd /boot/initramfs-linux.img
EOF
```

Nun wird Grub konfiguriert:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```


# Netzwerk

Das Netzwerk wird zunächst per DHCP konfiguriert:

```bash
systemctl enable dhcpcd
```


# SSH-Server

Um später auf die *JiffyBox* zugreifen zu können, muss ein SSH-Server
installiert werden, der beim Hochfahren der *JiffBox* auch wieder
automatisch gestartet wird:

```bash
pacman -Sy openssh
systemctl enable sshd
```


# Rootpasswort

Zum Schluss sollte unbedingt ein Rootpasswort gesetzt werden -
**das auf keinen Fall vergessen**:

```bash
passwd
```


# Herunterfahren

Nun kann *arch-chroot* und *chroot* Umgebung wieder verlassen werden,
und das System heruntergefahren werden.

Hinweis: ```unmount``` nicht vergessen!


# Neues *JiffyBox*-Profil
Jetzt muss ein neues *JiffyBox*-Profil angelegt werden. Als Name wählt
man beispielsweise *Jiffyarch* aus. Als Kernel muss unbedingt
*Bootmanager 64bit (pvgrub64)* verwendet werden.

Die Festplatten-Zuordnung sieht so aus:

*   ```/dev/xvda``` -> *arch-boot* (0.20, ext4)
*   ```/dev/svdb``` -> *arch-hd* (69.30, ext4)

Als Root-Festplatte wird ```/dev/xvda``` ausgewählt. Dieses Profil
speichert man nun. Im Kundenmenü klickt man dann in der Status-Spalte
auf *Aktivieren*, sodass das neuangelegte Profil auch standardmässig
verwendet wird.


# Starten der *Arch Linux* *JiffyBox*

Sofern das neue Profil angelegt wurde, kann die *JiffyBox* nun gestartet
werden. Zur Sicherheit bzw. für etwaige Fehlermeldungen eignet sich die
Web-Konsole im Kundenmenü hervorragend!


# Quellen

*   [Arch Linux LiveCD Image](https://wiki.archlinux.org/index.php/Install_from_Existing_Linux#Method_2:_Using_the_LiveCD_Image)
*   [Automatisierte Arch Installation (veraltet)](https://www.df.eu/forum/threads/70729-Automatisierte-Arch-Installation-%28Beta%29?highlight=arch)


# Mitarbeit

Bei Problemen etc. kann gerne ein GitHub [Issue](https://github.com/stefan-it/JiffyArch/issues) aufgemacht werden, bzw.
im Supportforum von domainFactory in [diesem](https://www.df.eu/forum/threads/74095-Arch-Linux-Installation) Thread
geantwortet werden.

# Optionale Konfiguration

## Alternative Netzwerkkonfiguration

Anstatt den ```dhcpcd``` Dämonen zu verwenden, kann man auch
```systemd-networkd``` benutzen.

Zunächst wird dieser beim Starten deaktiviert:

```bash
systemctl disable dhcpcd
```

Durch ```ip a``` bekommt u.a. den Namen der verwendeten Netzwerkkarte
heraus - in diesem Beispiel gehen wir von ```eth0``` aus. Jetzt wird die
Datei ```eth0.network``` in ```/etc/systemd/network``` angelegt, die die
Netzwerkkonfiguration enthält:

```bash
[Match]
Name=eth0

[Network]
DNS=109.239.48.251
Address=xx.xxx.xx.183/24
Gateway=xx.xxx.xx.254

Address=2a00:xxxx:3::cf/64
Gateway=fe80::1
```

Alle dafür benötigten Adressen befinden sich beim Punkt "Netzwerk" im
Kundenmenü. Um ```systemd-networkd``` bei jedem Start zu verwenden:

```bash
systemctl enable systemd-networkd
```

Die JiffyBox kann dann neugestartet werden. Mehr Informationen zur
```systemd-networkd```-Syntax gibt es [hier](http://www.freedesktop.org/software/systemd/man/systemd.network.html).


## NTP - Zeitsynchronisierung
In *systemd* (Ab Version 213) kann ein *timesync* Dämon verwendet werden
(in Kombination mit ```systemd-networkd```). Schritte zur Einrichtung
sind folgende:

```bash
systemctl enable systemd-timesyncd.service
sed -i 's/\(^ConditionVirtualization=\).*/\1xen/' /usr/lib/systemd/system/systemd-timesyncd.service
```

Wichtig ist, dass ```ConditionVirtualization``` auf ```xen``` geändert
wird, sonst startet der Dienst nicht.

Die eigentliche NTP Konfiguration kann dann in ```/etc/systemd/timesyncd.conf```
vorgenommen werden - ich verwende als Beispiel folgenden Server:

```bash
[Time]
Servers=ntp1.lrz.de
```

Nun kann der Dienst gestartet werden:

```bash
systemctl daemon-reload
systemctl start systemd-timesyncd.service
```

Status sollte dann soetwas anzeigen:

```bash
systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service; enabled)
   Active: active (running) since Mi 2014-08-20 19:52:35 CEST; 4s ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 669 (systemd-timesyn)
   Status: "Using Time Server [2001:4ca0:0:103::81bb:fe20]:123 (ntp1.lrz.de)."
   CGroup: /system.slice/systemd-timesyncd.service
           └─669 /usr/lib/systemd/systemd-timesyncd

Aug 20 19:52:35 jiffyarch systemd[1]: Started Network Time Synchronization.
Aug 20 19:52:35 jiffyarch systemd-timesyncd[669]: Using NTP server [2001:4ca0:0:103::81bb:fe20]:123 (ntp1.lrz.de).
Aug 20 19:52:35 jiffyarch systemd-timesyncd[669]: interval/delta/delay/jitter/drift 32s/-0.084s/0.020s/0.000s/+0ppm
```

Mehr Informationen zum ```systemd-timesyncd``` gibt es [hier](http://lists.freedesktop.org/archives/systemd-devel/2014-May/019537.html).

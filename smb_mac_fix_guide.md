# SMB-Dauerhafte Automount-, Indexierungsverhinderungs- und Reparaturlösung auf macOS

Diese Anleitung ermöglicht das stabile automatische Mounten von SMB-Freigaben auf macOS, verhindert dauerhaft die Indexierung externer Netzlaufwerke durch Spotlight und bietet Reparaturskripte für bekannte macOS-SMB-Probleme, ohne dass es zu Systemhänger oder Endlosschleifen kommt.

---

## 🔧 Voraussetzungen

* Der Server (z.B. Raspberry Pi, NAS) bietet SMB-Freigabe an
* macOS Ventura oder neuer (getestet auch unter Monterey)
* Terminal-Zugriff mit Adminrechten

---

## 1. 📦 Automatisches Mounten über `launchd`

### ➤ Script erstellen: `~/mount-smb.sh`

```bash
#!/bin/bash
MOUNTPOINT="/Volumes/myshare"
SHARE="//user@host/share"

if ! mount | grep "$MOUNTPOINT" > /dev/null; then
  mkdir -p "$MOUNTPOINT"
  mount_smbfs "$SHARE" "$MOUNTPOINT"
fi
```

### ➤ Ausführbar machen

```bash
chmod +x ~/mount-smb.sh
```

### ➤ LaunchAgent-Plist erstellen: `~/Library/LaunchAgents/com.user.mountsmb.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.mountsmb</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/$(whoami)/mount-smb.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>300</integer> <!-- alle 5 Minuten prüfen -->
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

### ➤ Aktivieren

```bash
launchctl load ~/Library/LaunchAgents/com.user.mountsmb.plist
```

---

## 2. 🛡️ Spotlight und Finder-Indexierung verhindern

### ➤ Für alle SMB-Volumes dauerhaft:

```bash
sudo defaults write /Volumes/.Spotlight-V100/VolumeConfiguration Exclusions -array-add /Volumes
```

Oder je Mountpoint direkt:

```bash
sudo mdutil -i off /Volumes/myshare
sudo mdutil -E /Volumes/myshare
```

### ➤ Alternativ systemweit deaktivieren:

```bash
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist
```

*(nicht empfohlen bei interner Spotlight-Nutzung)*

---

## 3. 🔁 Reparaturskript für fehlerhafte Mounts

### ➤ Skript: `~/repair-smb.sh`

```bash
#!/bin/bash
MOUNTPOINT="/Volumes/myshare"

if mount | grep "$MOUNTPOINT" > /dev/null; then
    echo "Unmounting $MOUNTPOINT"
    sudo umount -f "$MOUNTPOINT"
    sleep 1
    sudo rm -rf "$MOUNTPOINT"
fi

mkdir -p "$MOUNTPOINT"
mount_smbfs //user@host/share "$MOUNTPOINT"
```

### ➤ Ausführbar machen:

```bash
chmod +x ~/repair-smb.sh
```

### ➤ Manuell oder per Menü/App starten

(z.B. als Service mit Automator, kein LaunchDaemon – wegen Stabilität)

---

## 🔄 Optional: SSHFS als Ersatz für SMB (stabiler)

```bash
sshfs user@host:/pfad /Volumes/remote -o reconnect,volname=remote
```

(macFUSE notwendig)

---

## ✅ Hinweise zur Sicherheit & Stabilität

* Keine Endlosschleifen (Intervall-gesteuert)
* `mount_smbfs` wird nur bei Bedarf aufgerufen
* Kein Systemdienst wird dauerhaft blockiert
* `mdutil`-Deaktivierung nur gezielt für Volumes
* `repair-smb.sh` kann bei Bedarf über GUI gestartet werden

---

## Fazit

Mit dieser Lösung ist ein stabiler Zugriff auf SMB-Freigaben unter macOS möglich, ohne dass Spotlight oder das System blockiert werden. Ein automatischer Reconnect sowie manuelle Reparatur sind jederzeit möglich, ohne Neustart oder Dateiverlust.

# SMB-Probleme auf macOS: Ursachen, Abhilfe und Lösungen

## Problemstellung

Unter macOS kommt es bei Verwendung von SMB-Laufwerken (z.B. Raspberry Pi, NAS, Linux-Server) häufig zu Problemen nach Verbindungsunterbrechungen:

- Das Laufwerk wird "tot" (nicht mehr ansprechbar)
- Finder oder Terminal hängt beim Zugriff
- Der Ordner unter `/Volumes/...` lässt sich nicht mehr entfernen oder remounten
- Nur ein kompletter Neustart behebt das Problem zuverlässig

---

## Ursachen

- Instabile Netzwerkverbindungen (z.B. WLAN, IP-Wechsel)
- Samba-Server wird neu gestartet oder geht in Standby
- macOS cached und sperrt SMB-Zugriffe über Kernel-Subsysteme
- Spotlight oder Finder blockieren den Zugriff

---

## Lösungen ohne Neustart

### 1. Verbindung im Terminal trennen
```bash
mount | grep smbfs
# Beispiel: smbfs on /Volumes/myshare (smbfs, nodev, nosuid, mounted by user)

sudo umount -f /Volumes/myshare
```
Falls das nicht funktioniert:
```bash
sudo pkill -f smbfs
```

### 2. Blockierende Prozesse finden
```bash
sudo lsof | grep /Volumes/myshare
# Dann: kill -9 <PID>
```

### 3. Mount-Ordner manuell löschen
```bash
cd /Volumes
sudo rm -rf myshare
```

### 4. Neu mounten
```bash
mkdir -p /Volumes/myshare
mount_smbfs //user@hostname/share /Volumes/myshare
```

---

## Verbesserungen und Workarounds

### Spotlight deaktivieren für SMB-Volumes
```bash
sudo mdutil -i off /Volumes/myshare
```

### SMB mit "nobrowse" mounten (versteckt im Finder)
```bash
mount_smbfs -o nobrowse //user@host/share /Volumes/myshare
```

### Stabileres Mounting per Automounter
- Einrichtung in `/etc/auto_master` und `/etc/auto_smb`
- macOS kann Netzwerk-Mounts robuster verwalten

### Alternative Protokolle nutzen

#### SSHFS (über macFUSE)
```bash
sshfs user@host:/pfad /Volumes/remote -o reconnect,volname=remote
```

#### WebDAV (z.B. mit Nextcloud oder Synology)

---

## Automatisches Remount-Skript

```bash
#!/bin/bash
MOUNTPOINT="/Volumes/myshare"
SHARE="//user@host/share"

if ! mount | grep "$MOUNTPOINT" > /dev/null; then
  mkdir -p "$MOUNTPOINT"
  mount_smbfs "$SHARE" "$MOUNTPOINT"
fi
```

- Speichern als `~/mount-smb.sh`
- Start per Anmeldeobjekt oder `launchd`

---

## Fazit
SMB-Verbindungen unter macOS sind störanfällig, lassen sich aber mit einigen Workarounds zuverlässig betreiben. Bei anhaltenden Problemen lohnt sich ggf. ein Umstieg auf SSHFS oder WebDAV.


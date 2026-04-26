# Homelab: Debian-VM mit gehärtetem SSH

## Ziel
Eine Debian-12-VM in VirtualBox, vom Windows-Host aus per SSH-Key
erreichbar, mit deaktiviertem Passwort- und Root-Login und
Brute-Force-Schutz via fail2ban.

## Setup
- **Host:** Windows 11
- **Virtualisierung:** VirtualBox 7.2, NAT-Adapter mit Port-Forwarding
  (Host:2222 → Guest:22)
- **Guest:** Debian 12 (Bookworm), netinst, ohne Desktop-Umgebung
- **VM-Ressourcen:** 3 GB RAM, 2 vCPU, 20 GB HDD

## Schritte

### 1. Installation
Debian-netinst-ISO gebootet, in der Software-Auswahl alle
Desktop-Umgebungen abgewählt. Aktiviert blieben nur `SSH server`
und `standard system utilities`.

### 2. sudo nachinstalliert
Da ich im Installer ein Root-Passwort gesetzt hatte, wurde sudo nicht
automatisch konfiguriert. Behoben mit:

```
su -
apt install sudo
usermod -aG sudo ilkz
```

Anschließend ausgeloggt und neu eingeloggt, damit die
Gruppenmitgliedschaft wirksam wurde.

### 3. SSH-Schlüssel erzeugt (auf dem Host)

```
ssh-keygen -t ed25519
```

Mit Passphrase für zusätzlichen Schutz des privaten Schlüssels.

### 4. Public Key auf die VM kopiert

```
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 ilkz@127.0.0.1 ^
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"
```

### 5. SSH-Konfiguration gehärtet
In `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Aktiviert mit `sudo systemctl restart ssh`. Test in einer neuen Session,
bevor die alte geschlossen wurde.

### 6. fail2ban installiert

```
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## Verifikation
- Login mit Key funktioniert: `ssh -p 2222 ilkz@127.0.0.1`
- Passwort-Login wird abgelehnt
- Root-Login per SSH wird abgelehnt
- fail2ban-Jail `sshd` ist aktiv

## Stolpersteine
- **Unattended Installation in VirtualBox**: Der Wizard hatte unbemerkt
  eine automatisierte Installation aktiviert (preseed-Datei), wodurch
  Debian inklusive GNOME-Desktop installiert wurde, ohne dass ich
  Optionen auswählen konnte. Lösung: VM neu erstellt mit aktiviertem
  „Skip Unattended Installation".
- **sudo nicht verfügbar**: Wenn im Debian-Installer ein Root-Passwort
  gesetzt wird, installiert Debian sudo nicht automatisch. Hätte ich
  das Root-Passwort leer gelassen, wäre sudo direkt für meinen User
  konfiguriert worden.
- **`usermod` nicht gefunden trotz Root-Rechten**: Ich hatte mit `su`
  (ohne Bindestrich) gewechselt, dadurch wurde nur die User-ID, nicht
  aber das Environment geändert. `/usr/sbin` (wo `usermod` liegt) ist
  nicht im PATH normaler User. Lösung: `su -` für eine vollständige
  Login-Shell als root.
- **`Connection timed out` beim SSH-Login vom Host**: Ich hatte die
  interne VM-IP (10.0.2.15) als Ziel benutzt, die im NAT-Modus vom Host
  aus nicht routbar ist. Lösung: Verbindung an `127.0.0.1` auf den
  weitergeleiteten Port 2222.

## Lessons Learned
- Vor SSH-Konfigurationsänderungen IMMER eine zweite Session offenhalten,
  um sich nicht auszusperren.
- Unterschied zwischen `su` und `su -` praktisch verstanden:
  `su -` lädt die vollständige Login-Umgebung des Ziel-Users.
- VirtualBox-Netzwerkmodi (NAT vs. Bridge) und Port-Forwarding
  durchgespielt — das NAT-Netz ist vom Host nur über
  Port-Weiterleitungsregeln erreichbar.
- Public-Key-Authentifizierung verinnerlicht: privater Schlüssel bleibt
  auf dem Host, öffentlicher Schlüssel wird in `~/.ssh/authorized_keys`
  des Servers hinterlegt.
- Berechtigungen auf `~/.ssh` (700) und `authorized_keys` (600) sind
  zwingend — andernfalls verweigert sshd den Key-Login aus
  Sicherheitsgründen.

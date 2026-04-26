# Debian-VM mit gehärtetem SSH

## Ziel
Eine Debian-12-VM in VirtualBox, vom Windows-Host aus per SSH-Key
erreichbar, mit deaktiviertem Passwort- und Root-Login und
Brute-Force-Schutz.

## Setup
- Host: Windows 11
- VirtualBox 7.x mit NAT + Port-Forwarding (Host:2222 → Guest:22)
- Debian 12 (netinst), minimal ohne Desktop-Umgebung
- VM: 3 GB RAM, 2 vCPU, 20 GB HDD

## Schritte
1. Debian-Installation (Software-Auswahl: nur SSH server + standard utilities)
2. sudo nachinstalliert (war wegen gesetztem Root-Passwort nicht standardmäßig dabei)
3. User in sudo-Gruppe aufgenommen: `usermod -aG sudo ilkz`
4. SSH-Schlüsselpaar (ed25519) auf dem Host erzeugt: `ssh-keygen -t ed25519`
5. Public Key per Pipe in VM kopiert nach ~/.ssh/authorized_keys
6. sshd_config gehärtet:
   - PermitRootLogin no
   - PasswordAuthentication no
   - PubkeyAuthentication yes
7. fail2ban installiert und aktiviert (jail sshd)

## Stolpersteine
- VirtualBox-Wizard hatte unbemerkt Unattended Installation aktiviert,
  dadurch Debian mit GNOME installiert. Lösung: VM neu erstellt mit
  "Skip Unattended Installation".
- sudo war nicht installiert (passiert in Debian, wenn man im Installer
  ein Root-Passwort setzt). Lösung: als root nachinstalliert.
- usermod nicht gefunden: /usr/sbin nicht im PATH normaler User. Lösung:
  Mit `su -` (statt `su`) zur vollen Root-Umgebung wechseln.
- SSH-Verbindung scheiterte mit "Connection timed out", weil ich die
  VM-interne IP (10.0.2.15) statt 127.0.0.1 verwendet hatte – das
  NAT-Netz ist vom Host aus nicht direkt erreichbar.

## Lessons Learned
- Vor SSH-Konfigurationsänderungen IMMER eine zweite Session offenhalten,
  um sich nicht auszusperren.
- Unterschied zwischen `su` und `su -` praktisch verstanden (Login-Shell
  vs. einfache User-Wechsel).
- VirtualBox-Netzwerkmodi (NAT vs. Bridge) und Port-Forwarding praktisch
  durchgespielt.
- Public-Key-Authentifizierung verstanden: privater Key bleibt auf dem
  Host, öffentlicher Key in ~/.ssh/authorized_keys auf dem Server.
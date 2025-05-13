# Nightfall - HackMyVM (Medium)

![Nightfall.png](Nightfall.png)

## Übersicht

*   **VM:** Nightfall
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Nightfall)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 11. November 2022
*   **Original-Writeup:** https://alientec1908.github.io/Nightfall_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Nightfall" zu erlangen. Der initiale Zugriff erfolgte durch das Ausnutzen eines Informationslecks: Ein UDP-Broadcast auf Port 24000 sendete Base64-kodierte, mit OpenSSL verschlüsselte Daten. Nach dem Abfangen, Dekodieren und Knacken des Passworts (`amoajesus`) für die Verschlüsselung wurde ein privater SSH-Schlüssel enthüllt. Mittels Benutzerenumeration via Metasploit wurde der zugehörige Benutzer `abraham` identifiziert, was einen SSH-Login ermöglichte. Die finale Rechteausweitung zu Root gelang durch Ausnutzung der Mitgliedschaft des Benutzers `abraham` in der Gruppe `disk`. Dies erlaubte den direkten Zugriff auf das Blockgerät (`/dev/sda1`) mit `debugfs`, um den privaten SSH-Schlüssel des Root-Benutzers auszulesen.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `lftp`
*   `nc` (netcat)
*   `nano`
*   `base64`
*   `file`
*   `bruteforce-salted-openssl`
*   `openssl`
*   `chmod`
*   `ssh2john`
*   `msfconsole` (Metasploit Framework)
*   `ssh`
*   `debugfs`
*   Standard Linux-Befehle (`cat`, `ls`, `mv`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Nightfall" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.110) mit `arp-scan` identifiziert. Hostname `nightfall.hmv` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 21 (FTP, ProFTPD, anonymer Login erlaubt) und Port 22 (SSH, OpenSSH 7.7).
    *   Über anonymen FTP-Login wurde die Datei `darkness.txt` heruntergeladen. Diese enthielt den Hinweis "In the darkness, there are invisible things happening...".

2.  **Data Discovery & Initial Access (SSH als `abraham`):**
    *   Ein Netcat-Listener auf UDP Port 24000 (`nc -lvnup 24000`) empfing einen Base64-kodierten, mit OpenSSL verschlüsselten Datenblock vom Ziel.
    *   Der Base64-Block wurde dekodiert und die resultierende Binärdatei mit `bruteforce-salted-openssl` und `rockyou.txt` analysiert. Das Passwort `amoajesus` wurde gefunden.
    *   Mittels `openssl enc -aes-256-cbc -d -in out -out decrypt_file` und dem Passwort `amoajesus` wurde die Datei entschlüsselt. Sie enthielt einen privaten SSH-RSA-Schlüssel.
    *   Der Schlüssel war nicht passwortgeschützt (`ssh2john ... has no password!`).
    *   Mittels Metasploit (`auxiliary/scanner/ssh/ssh_enumusers`) wurde der Benutzer `abraham` auf dem SSH-Dienst enumeriert.
    *   Erfolgreicher SSH-Login als `abraham` mit dem extrahierten privaten Schlüssel.
    *   Die User-Flag (`c02546146dfcf2bec351c4fece2dec21`) wurde in `/home/abraham/user.txt` gefunden.

3.  **Privilege Escalation (von `abraham` zu `root` via `debugfs`):**
    *   Als `abraham` zeigte `id`, dass der Benutzer Mitglied der Gruppe `disk` ist.
    *   Mittels `/usr/sbin/debugfs -w /dev/sda1` wurde direkter Zugriff auf das Root-Dateisystem erlangt.
    *   Innerhalb von `debugfs` wurde mit `cat /root/.ssh/id_rsa` der private SSH-Schlüssel des Root-Benutzers ausgelesen.
    *   Der private Root-Schlüssel wurde auf dem Angreifer-System gespeichert (`rootrsa`, `chmod 600`).
    *   Erfolgreicher SSH-Login als `root` mit dem extrahierten privaten Schlüssel.
    *   Die Root-Flag (`3aa982dd7e61dd66d116dea033e10dc7`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Informationsleck über UDP-Broadcast:** Sensible Daten (verschlüsselter privater SSH-Schlüssel) wurden unaufgefordert über einen UDP-Port gesendet.
*   **Passwort-Cracking (OpenSSL & SSH-Key):** Das Passwort für die OpenSSL-verschlüsselten Daten wurde mit `bruteforce-salted-openssl` geknackt. Der SSH-Key selbst war nicht passwortgeschützt.
*   **Unsichere Gruppenmitgliedschaft (Gruppe `disk`):** Die Mitgliedschaft eines normalen Benutzers in der Gruppe `disk` ermöglichte direkten Lese- (und Schreib-)Zugriff auf Blockgeräte und umging somit die normalen Dateisystemberechtigungen. Dies wurde mit `debugfs` ausgenutzt, um den privaten SSH-Schlüssel von Root zu lesen.
*   **Anonymer FTP-Zugang:** Ermöglichte das Herunterladen einer Hinweisdatei.

## Flags

*   **User Flag (`/home/abraham/user.txt`):** `c02546146dfcf2bec351c4fece2dec21`
*   **Root Flag (`/root/root.txt`):** `3aa982dd7e61dd66d116dea033e10dc7`

## Tags

`HackMyVM`, `Nightfall`, `Medium`, `Information Leak`, `UDP`, `OpenSSL Encryption Crack`, `SSH Key Leak`, `debugfs`, `disk group exploit`, `Metasploit`, `Linux`, `Privilege Escalation`, `FTP`

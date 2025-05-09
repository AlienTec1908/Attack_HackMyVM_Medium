# Attack - HackMyVM Writeup

![Attack VM Icon](Attack.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "Attack" (Schwierigkeitsgrad: Medium), erstellt von DarkSpirit. Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** Attack
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Medium
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Attack](https://hackmyvm.eu/machines/machine.php?vm=Attack)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 02. November 2022
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/Attack_HackMyVM_Medium/](https://alientec1908.github.io/Attack_HackMyVM_Medium/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die Attack-Maschine umfasste folgende Schritte:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.100`) mittels `arp-scan`.
    *   Ein `nmap`-Scan offenbarte drei offene Ports: FTP (21, ProFTPD), SSH (22, OpenSSH 7.9p1) und HTTP (80, Nginx 1.14.2).
2.  **Web Enumeration (Port 80) & Capture File Analysis:**
    *   `gobuster` fand nur eine `index.html`-Datei.
    *   Der Inhalt von `index.html` enthielt den Hinweis: "I did a capture with wireshark. The name of the file is "capture" but i dont remember the extension :(".
    *   Die Datei `capture.pcap` wurde erfolgreich vom Webserver heruntergeladen.
    *   Analyse der `capture.pcap` mit `tcpdump -r` offenbarte FTP-Zugangsdaten im Klartext: `teste:simple`.
3.  **FTP Access & SSH Key Upload:**
    *   Erfolgreicher anonymer FTP-Login als Benutzer `teste` mit dem Passwort `simple`.
    *   Im Home-Verzeichnis von `teste` wurden die Dateien `mysecret.png` und `note.txt` ("I need to launch the script to start the attack planned by kratos.") sowie ein `.ssh`-Verzeichnis gefunden.
    *   Es wurde festgestellt, dass der FTP-Benutzer `teste` nicht in sein Home-Verzeichnis gechrootet war und andere Home-Verzeichnisse (`jackob`, `kratos`) auflisten konnte.
    *   Der öffentliche SSH-Schlüssel des Angreifers wurde als `authorized_keys` in das Verzeichnis `/home/teste/.ssh/` hochgeladen.
4.  **SSH Access (teste) & Weitere Enumeration:**
    *   Erfolgreicher SSH-Login als `teste` mittels des hochgeladenen Schlüssels.
    *   Auf dem Webserver wurden weitere Dateien gefunden:
        *   `filexxx.zip`: Enthielt den privaten SSH-Schlüssel für `teste`.
        *   `jackobattack.txt`: Enthielt den privaten SSH-Schlüssel für den Benutzer `jackob`.
5.  **Lateral Movement (zu jackob):**
    *   Der private SSH-Schlüssel für `jackob` wurde verwendet, um sich per SSH als `jackob` anzumelden.
6.  **Privilege Escalation (jackob zu kratos):**
    *   `sudo -l` für `jackob` zeigte: `(kratos) NOPASSWD: /home/jackob/attack.sh`.
    *   Das Skript `/home/jackob/attack.sh` wurde durch eine Version ersetzt, die eine Netcat-Reverse-Shell zum Angreifer startete.
    *   Ausführung von `sudo -u kratos /home/jackob/attack.sh` führte zu einer Shell als Benutzer `kratos`.
7.  **Privilege Escalation (kratos zu root):**
    *   Eine Datei `password_file` wurde mit einem Eintrag für `root` und einem bekannten Passwort-Hash erstellt (`root:$6$...:/root:/usr/bin/bash`).
    *   Der Befehl `sudo /usr/sbin/cppw password_file` wurde ausgeführt (impliziert, dass `kratos` diese `sudo`-Berechtigung hatte). Dies setzte das Root-Passwort auf das dem Hash entsprechende, dem Angreifer bekannte Passwort.
    *   Mit `su` und dem neuen Passwort wurde eine Root-Shell erlangt.
8.  **Flags:**
    *   Die User-Flag wurde als `teste` oder `jackob` gelesen (impliziert, nicht explizit im Log gezeigt, aber im Flag-Abschnitt des Berichts genannt).
    *   Die Root-Flag wurde aus `/root/root.txt` gelesen.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nikto`
*   `tcpdump`
*   `curl`
*   `ftp`
*   `cat`
*   `echo`
*   `ssh`
*   `sudo`
*   `find`
*   `uname`
*   `unzip`
*   `vi`
*   `mv`
*   `chmod`
*   `nc` (netcat)
*   `cd`
*   `pwd`
*   `cppw` (chpasswd - custom password tool)
*   `su`
*   `ls`
*   `id`

## Identifizierte Schwachstellen (Zusammenfassung)

*   **Verfügbarkeit einer Netzwerk-Capture-Datei:** Eine `.pcap`-Datei mit Klartext-FTP-Zugangsdaten war öffentlich zugänglich.
*   **Klartext-FTP-Zugangsdaten:** FTP-Login-Informationen (`teste:simple`) wurden in der Capture-Datei gefunden.
*   **Fehlendes Chroot Jail für FTP-Benutzer:** Der FTP-Benutzer `teste` konnte Verzeichnisse außerhalb seines Home-Verzeichnisses einsehen.
*   **Exponierte private SSH-Schlüssel auf Webserver:** Private Schlüssel für `teste` und `jackob` waren öffentlich zugänglich.
*   **Unsichere `sudo`-Konfiguration (jackob zu kratos):** `jackob` konnte ein von ihm selbst modifizierbares Skript (`/home/jackob/attack.sh`) als Benutzer `kratos` ohne Passwort ausführen.
*   **Unsichere `sudo`-Konfiguration / benutzerdefiniertes Passwort-Tool (kratos zu root):** `kratos` konnte `/usr/sbin/cppw` (vermutlich via `sudo`) ausführen, um das Root-Passwort basierend auf einer Datei zu ändern.

## Flags

*   **User Flag (teste/jackob):** `HMVattackstarted`
*   **Root Flag (`/root/root.txt`):** `HMVattackr00t`

---

**Wichtiger Hinweis:** Dieses Dokument und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Techniken und Werkzeuge sollten nur in legalen und autorisierten Umgebungen (z.B. eigenen Testlaboren, CTFs oder mit expliziter Genehmigung) angewendet werden. Das unbefugte Eindringen in fremde Computersysteme ist eine Straftat und kann rechtliche Konsequenzen nach sich ziehen.

---
*Bericht von Ben C. - Cyber Security Reports*

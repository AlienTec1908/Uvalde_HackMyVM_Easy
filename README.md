# Uvalde - HackMyVM (Easy)
 
![Uvalde.png](Uvalde.png)

## Übersicht

*   **VM:** Uvalde
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Uvalde)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 8. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Uvalde_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Uvalde"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers (Port 80), auf dem eine `login.php` und `create_account.php` gefunden wurden. Eine auf dem FTP-Server (Port 21, anonymer Login erlaubt) gefundene Datei `output` (eine `script`-Mitschrift) enthüllte den Benutzernamen `matthew`. Eine weitere Information, die über eine URL (`success.php?dXNlcm...`) und Base64-Dekodierung gefunden wurde, lieferte die Credentials `darkben:darkben2023@6938`. Das Passwortmuster `[Benutzername]2023@[Zahl]` wurde auf `matthew` angewendet. Mittels `wfuzz` wurde der numerische Teil des Passworts für `matthew` auf der `login.php` gebruteforced (`matthew2023@1554`). Dies ermöglichte den SSH-Login als `matthew`. Die User-Flag wurde in dessen Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel: `matthew` durfte `/bin/bash /opt/superhack` als `root` ohne Passwort ausführen. Da das Verzeichnis `/opt` für `matthew` schreibbar war, wurde `/opt/superhack` durch ein Skript ersetzt, das `chmod u+s /bin/bash` ausführte. Nach Ausführung dieses manipulierten Skripts mit `sudo` hatte `/bin/bash` das SUID-Bit gesetzt, was den Start einer Root-Shell mit `/bin/bash -p` ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `ftp`
*   `nikto`
*   `base64`
*   `wfuzz`
*   `ssh`
*   `sudo`
*   `bash`
*   `echo`
*   `rm`
*   `cat`
*   `toilet`
*   Standard Linux-Befehle (`ls`, `cd`, `id`, `pwd`, `chmod`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Uvalde" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web/FTP Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.123`).
    *   `nmap`-Scan identifizierte offene Ports: 21 (FTP - vsftpd 3.0.3, anonymer Login erlaubt), 22 (SSH - OpenSSH 8.4p1), 80 (HTTP - Apache 2.4.54 "Agency - Start Bootstrap Theme").
    *   `gobuster` auf Port 80 fand u.a. `login.php` und `create_account.php`.
    *   `nikto` auf Port 80 fand eine `package.json` und fehlende Security Header.
    *   Anonymer FTP-Login auf Port 21. Download der Datei `output`.
    *   Analyse von `output` (eine `script`-Mitschrift) enthüllte den Benutzernamen `matthew`.
    *   Entdeckung einer URL `http://192.168.2.123/success.php?dXNlcm5hbWU9ZGFya2JlbiZwYXNzd29yZD1kYXJrYmVuMjAyM0A2OTM4`.
    *   Base64-Dekodierung des Parameters ergab `username=darkben&password=darkben2023@6938`.

2.  **Initial Access (Passwort-Bruteforce & SSH als `matthew`):**
    *   Ableitung eines Passwortmusters (`[Benutzername]2023@[Zahl]`).
    *   Generierung einer Wortliste `numbers.txt` (1000-10000).
    *   `wfuzz -X POST -d "username=matthew&password=matthew2023@FUZZ" -w numbers.txt -u http://uvalde.hmv/login.php --hh 1022` fand das Passwort `matthew2023@1554` für `matthew`.
    *   Erfolgreicher SSH-Login als `matthew` mit dem gefundenen Passwort.
    *   User-Flag `6e4136fbed8f8c691996dbf42697d460` in `/home/matthew/user.txt` gelesen.

3.  **Privilege Escalation (von `matthew` zu `root` via `sudo` und schreibbarem Skript):**
    *   `sudo -l` als `matthew` zeigte: `(ALL : ALL) NOPASSWD: /bin/bash /opt/superhack`.
    *   Das Verzeichnis `/opt` war für `matthew` schreibbar.
    *   Die Originaldatei `/opt/superhack` wurde gelöscht (`rm /opt/superhack`).
    *   Eine neue Datei `/opt/superhack` wurde erstellt mit dem Inhalt `chmod u+s /bin/bash`.
    *   Ausführung des manipulierten Skripts mit `sudo /bin/bash /opt/superhack`.
    *   Dadurch wurde das SUID-Bit auf `/bin/bash` gesetzt.
    *   Starten einer Root-Shell mit `/bin/bash -p`.
    *   Erlangung von Root-Rechten (`euid=0(root)`).
    *   Root-Flag `59ec54537e98a53691f33e81500f56da` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugriff:** Ermöglichte den Download einer Datei (`output`) mit sensiblen Informationen (Benutzername, Systemdetails).
*   **Informationsleck durch Base64 in URL:** Zugangsdaten wurden Base64-kodiert in einem URL-Parameter übergeben.
*   **Passwort-Brute-Force (Web Login):** Ein schwaches, vorhersagbares Passwortmuster erlaubte das Knacken des Passworts für `matthew` mittels `wfuzz`.
*   **Unsichere `sudo`-Konfiguration (schreibbares Skript):** Die Erlaubnis, ein Skript als `root` auszuführen, das in einem für den Benutzer schreibbaren Verzeichnis lag, ermöglichte die Manipulation des Skripts und die Ausführung beliebigen Codes als `root`.
*   **SUID-Bit auf Bash:** Setzen des SUID-Bits auf `/bin/bash` als Methode zur einfachen Erlangung einer Root-Shell.

## Flags

*   **User Flag (`/home/matthew/user.txt`):** `6e4136fbed8f8c691996dbf42697d460`
*   **Root Flag (`/root/root.txt`):** `59ec54537e98a53691f33e81500f56da`

## Tags

`HackMyVM`, `Uvalde`, `Easy`, `FTP`, `Base64`, `wfuzz`, `Password Cracking`, `sudo Exploitation`, `SUID Bash`, `Privilege Escalation`, `Linux`, `Web`

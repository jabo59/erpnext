# Installation ERPNext

ERPNext ist eine **Open-Source-Lösung** für **Enterprise Resource Planning (ERP)**. ERPNext kann in verschiedenen Branchen eingesetzt werden, z. B. in der Fertigung, im Vertrieb, im Einzelhandel, im Handel, im Dienstleistungssektor, im Bildungswesen, im Non-Profit-Bereich und im Gesundheitswesen. Es bietet außerdem Module wie Buchhaltung, CRM, Vertrieb, Einkauf, Website, E-Commerce, Point of Sale, Fertigung, Lager, Projektmanagement, Inventar und Dienstleistungen.

ERPNext ist eine ERP-Plattform für Unternehmen, die unter der **GNU General Public Licence v3** lizenziert ist. Sie ist hauptsächlich in **Python** und **JavaScript** geschrieben und wurde von Frappe Technologies Pvt. Ltd. entwickelt. ERPNext ist eine Anwendung, die unter dem **frappeframework** geschrieben wurde, einem Open-Source-Webframework in Python und JavaScript.

ERPNext wurde als Alternative zu Diensten wie NetSuite von Oracle, QAD, Tython, OpenBrave und Odoo entwickelt. Von der Funktionalität her ist ERPNext ähnlich wie Odoo (früher OpenERP).

Dieses Dokument führt durch die Installation von ERPNext auf einem System mit dem Betriebssystem Debian 12. ERPNext wird mit einem MariaDB-Datenbankserver, Nginx als Reverse Proxy und einem Supervisor-Prozessmanager installiert.

## Voraussetzungen

Um loszulegen, musst du sicherstellen, dass du Zugang zu Folgendem hast

- Ein aktuelles Debian 12 System.
- Einen Nicht-Root-Benutzer erpnext mit sudo-Administrator-Rechten.
- Von ERPNext benötigte Pakete (Python 3, MariaDB Server, Node.js, Yarn Paketmanager, Nginx, Supervisor Prozessmanager und Redis).
- Einen Domainnamen, der auf die IP-Adresse des Servers zeigt.

### Debian 12 System einrichten

In diesem Dokument wird beschrieben, wie ein Debian 12 System als Container in einer Proxmox-VE-Umgebung installiert wird:  
[Einrichtung Debian-Container](https://doku.lubo-it.de/books/root-server-bibel/page/einrichtung-debian-container "Einrichtung Debian-Container")

<p class="callout info">Der Benutzer mit sudo-Administratorrechten sollte nicht wie im Dokument angeben "admin" heißen, sondern "erpnext". Siehe dazu nächster Abschnitt.</p>

### Nicht-Root-Benutzer einrichten

Benutzer einrichten:

```bash
adduser erpnext
```

Das Kommando `adduser`fragt nach mehreren Angaben, u.a. nach dem Kennwort, dass der Benutzer erhalten soll. Dieses Konto ist an sicherer Stelle zu verwahren.

sudo-Rechte einrichten:

```bash
usermod -aG sudo erpnext
```

Anschließend in den Benutzer "erpnext" wechseln.

```
su - erpnext
```

### Benötigte Pakete installieren

```bash
sudo apt install python3-dev \
python3-venv \
nodejs \
yarnpkg \
npm \
redis-server \
mariadb-server \
nginx \
supervisor \
fail2ban \
libffi-dev \
git \
python3-pip \
python3-testresources \
libssl-dev \
gcc \
g++ make
```

**wkhtmltox**

Um PDF-Dateien zu erzeugen, wird wkhtmltopdf benötigt. Zuerst müssen die benötigten Abhängigkeiten installiert werden.

```
sudo apt install xfonts-75dpi xfonts-base
```

Anschließend wird wkhtmltopdf installiert.

```
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.bookworm_amd64.deb
```

```
sudo dpkg -i wkhtmltox_0.12.6.1-3.bookworm_amd64.deb
```

Falls Meldungen, dass Pakete fehlen, muss folgendes Kommando ausgeführt werden:

```
sudo apt --fix-broken install
```

Mit folgendem Kommando kann geprüft werden, ob die Installation geklappt hat:

```
wkhtmltopdf --version
```

Ausgabe:

[![grafik.png](https://doku.lubo-it.de/uploads/images/gallery/2024-08/scaled-1680-/H26grafik.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-08/H26grafik.png)

Mit den folgenden Befehlen kann überprüft werden, ob die benötigten Pakete korrekt installiert wurden.

**MariaDB-Server:**

```
sudo systemctl is-enabled mariadb
```

Erwartete Ausgabe: enabled.

```
sudo systemctl status mariadb
```

Erwartete Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/XQ8image.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/XQ8image.png)

**Nginx:**

```
sudo systemctl is-enabled nginx
```

Erwartete Ausgabe: enabled.

```
sudo systemctl status nginx
```

Erwartete Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/6hRimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/6hRimage.png)

**Supervisor Process Manager:**

```
sudo systemctl is-enabled supervisor
```

Erwartete Ausgabe: enabled.

```
sudo systemctl status supervisor
```

Erwartete Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/jzlimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/jzlimage.png)

**Redis:**

```
sudo systemctl is-enabled redis-server
```

Erwartete Ausgabe: enabled.

```
sudo systemctl status redis-server
```

Erwartete Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/u7gimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/u7gimage.png)

**Node.js und npm:**

```
node --version
```

```
npm --version
```

Erwartete Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/VQZimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/VQZimage.png)

Es sollten mindestens diese Versionsnummer erscheinen.

Jetzt muss noch tarn installiert werden.

```
sudo npm install -g yarn
```

## MariaDB-Server konfigurieren

Nachdem die Abhängigkeiten installiert sind, wird der MariaDB-Server konfiguriert, um sicherzustellen, dass er für die ERPNext-Installation bereit ist. Für ERPNext muss das Barracuda-Format aktiviert und der Standardzeichensatz auf utf8mb4 eingestellt sein. Außerdem muss der MariaDB-Server mit dem Dienstprogramm mariadb-secure-installation abgesichert werden.

Um den MariaDB-Server abzusichern, wird folgendes Kommando ausgeführt:

```
sudo mariadb-secure-installation
```

Die folgenden Eingaben sind erforderlich:

1. Die root-Kennwortabfrage mit der **&lt;Enter&gt;-Taste** bestätigen (kein Kennwort bisher eingerichtet).
2. Lokale Authentifizierung auf unix\_socket umstellen? **"n"** für nein eingeben.
3. MariaDB Root-Passwort einrichten? **"Y"** und dann das neue **MariaDB Root-Passwort** eingeben und wiederholen.
4. Den anonymen Standardbenutzer entfernen? **"Y"** eingeben.
5. Fernanmeldung für den Root-Benutzer deaktivieren? **"Y"** eingeben.
6. Den Standard-Datenbanktest entfernen? **"Y"** eingeben.
7. Tabellenberechtigungen neu laden und Änderungen übernehmen? **"Y"** eingeben.

Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/zGOimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/zGOimage.png)

Im nächsten Schritt muss die MariaDB Server Konfigurationsdatei */etc/mysql/mariadb.conf.d/50-server.cnf* mit dem einem Editor (z. B. nano) angepasst werden.

```
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Folgende Konfiguration im Abschnitt **\[mysqld\]** einfügen, um das Barracuda-Format zu aktivieren und den Standardzeichensatz auf utf8mb4 einzustellen.

```
[mysqld]
innodb-file-format=barracuda
innodb-file-per-table=1
innodb-large-prefix=1
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

Anschließend die Datei speichern und den Editor beenden.

Als nächstes muss die Datei */etc/mysql/mariadb.conf.d/50-mysql-clients.cnf* angepasst werden, um die MariaDB-Client-Verbindung zu konfigurieren.

```
sudo nano /etc/mysql/mariadb.conf.d/50-mysql-clients.cnf
```

Folgende Konfiguration in den Abschnitt **\[mysql\]** einfügen.

```
[mysql]
default-character-set = utf8mb4
```

Nach der Änderung die Datei speichern und den Editor beenden.

Abschließend muss der MariaDB-Server neu gestartet werden, damit die neue Konfiguration angewendet wird.

```
sudo systemctl restart mariadb
```

## Bench-Befehlszeilen-Tool installieren

Bench ist ein Kommandozeilentool zur Verwaltung von Frappe Frameworks, einschließlich Anwendungen und Websites. ERPNext ist eine mit dem Frappe Framework geschriebene Webanwendung, die über Bench installiert werden muss.

Das Paket **frappe-bench** oder bench wird über den Python-Paketmanager pip installiert.

```
sudo pip3 install frappe-bench --break-system-packages
```

Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/ysUimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/ysUimage.png)

Sobald die frappe-bench installiert ist, kann sie mit dem folgenden Befehl überprüft werden. In diesem Beispiel wird bench 5.17 unter */usr/local/bin/bench* installiert.

```
which bench
```

```
bench --version
```

Ausgabe:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/5llimage.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/5llimage.png)

## ERPNext über Bench installieren

In diesem Abschnitt wird ERPNext über die Bench-Befehlszeile installiert. Es wird das Frappe-Framework-Projekt initialisiert, eine neue Site erstellt und dann die ERPNext-Anwendung heruntergeladen und im Frapper-Projekt installiert.

Zunächst den unten stehenden Bench-Befehl ausführen, um Frappe Framework 15 im Verzeichnis frappe-bench zu initialisieren.

```
bench init --python python3.11 --frappe-branch version-15 frappe-bench
```

Ausgabe nach Beginn der Installation:

[![image.png](https://doku.lubo-it.de/uploads/images/gallery/2024-10/scaled-1680-/3p1image.png)](https://doku.lubo-it.de/uploads/images/gallery/2024-10/3p1image.png)

Ausgabe nach Abschluss der Installation:

## Domainnamen konfigurieren

Es wird davon ausgegangen, dass eine Internet-Domäne wie zum Beispiel "example.com" angemietet wurde. Für das ERPNext-System wird angenommen, dass es im Internet unter "erpnext.example.com" erreichbar sein soll. Für example.com ist die tatsächlich angemietete Internet-Domäne zu verwenden.
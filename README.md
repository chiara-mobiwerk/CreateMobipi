# CreateMobipi
# mobivideo

Software um Verkehrs-Videos auf einem Raspberry Pi Zero aufzunehmen.
## Anforderungen
- Raspberry Pi Zero W
- Raspberry Pi Camera Module V2
- [Ribbon Kabel](https://www.berrybase.de/flexkabel-fuer-raspberry-pi-zero-und-kameramodul-laenge-15-cm) um Kamera und Raspberry zu verbinden
- Micro SD Karte (64-128GB)
- Adafruit PiRTC, PCF8523 Real Time Clock
- Knopfzelle Lithium CR1220
- [Powerbank](https://www.berrybase.de/flexkabel-fuer-raspberry-pi-zero-und-kameramodul-laenge-15-cm)
- Wasserdichter Koffer
![Aufbau]()
## Installation
### SD Karte vorbereiten 
Zuerst muss das richtige Betriebssystem auf die SD Karte geflashed werden. Dafür den [Raspberry Pi Imager](https://www.raspberrypi.com/software/) herunterladen und installieren, die SD Karte einlegen und den Imager starten.
![Raspberry Pi Imager](raspi-imager.png)
Raspberry Pi Modell (Raspberry Pi Zero), Betriebssystem (Raspberry PI OS (Legacy, 32-Bit) Lite), und SD Karte auswählen. 
Um die erweiterten Einstellungen zu öffnen <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>X</kbd> drücken.
Hier muss der Hostname eingerichtet werden, die Wi-Fi	Anmeldedaten (vom Netzwerk im Büro) sowie Zeitzone und Tastaturlayout gewählt werden. Als Benutzernamen "pi" beibehalten und Passwort vergeben.

![Raspberry Pi Imager allgemeine Einstellungen](media/imager-allgemein.png)
![Raspberry Pi Imager Dienste Einstellungen](media/imager-dienste.png)

Anschließend <kbd>Weiter</kbd> &rarr; <kbd>Ja</kbd>.
Nachdem der Schreib-Prozess abgeschlossen ist kann die SD Karte in den Raspberry Pi eingesteckt werden.
## Raspberry einrichten
Wichtig: für jede SD-Karte einen anderen Raspberry verwenden, da der SHA256-Hash unter *known_hosts* gespeichert wird und ansonsten keine Verbindung hergestellt werden kann. Alternativ müssen die Einträge aus *known_hosts* nach jedem Einrichten wieder gelöscht werden.
Ist die SD Karte eingesteckt, kann der Raspberry gebootet werden. Dafür ein micro-USB Kabel in den mit "PWR IN"-beschrifteten Port stecken. Der Raspberry sollte sich nun automatisch mit dem im Setup angegebenen WLAN Netzwerk verbinden. Nach etwa 2 min (kann aber auch länger sein) sollte man sich jetzt per SSH mit dem Raspberry verbinden können. Dazu muss man selbst im gleichen WLAN Netzwerk sein.   

Angaben im Terminal:  
```
ping mobipi01
```  
Ip Adresse der ersten Zeile kopieren und nutzen in:  
```
ssh pi@kopierte-ip-adresse
```
(kopierte Sachen fügt man mit Rechtsklick ein)  

Wenn alles korrekt eingerichtet ist, wird man aufgefordert, den Host-Schlüssel zur Liste der bekannten Hosts hinzuzufügen (Antwort: Ja) und sollte dann mit dem Raspberry Pi verbunden sein.  
Falls man darauf hingewiesen wird, dass es den Host schon gibt, dann kann man ihn in der Datei known_hosts.txt löschen.  
Benutzer/.ssh/known_hosts.txt  
#### Update
Raspberry Pi durch apt und reboot updaten.  
```
sudo  apt  update  &&  sudo  apt  upgrade  -y  &&  sudo  reboot
```  
Dieser Prozess kann einige Minuten dauern.

Durch den reboot wird das Gerät ausgeschalten.
#### Repository clonen:
SSH neu starten:
```
ssh pi@mobipi01
```
```bash
sudo apt install python3-pip git -y && git clone https://github.com/MobilitaetswerkGmbH/mobivideo.git
```
### Raspi Config
Verbinde dich erneut mit deinem Raspberry Pi (öffne PowerShell und führe den Befehl `ssh pi@mobipi01` (bzw. andere ID aus) und starte das Raspberry-Konfigurationstool.
```
sudo raspi-config
```
Ändere die folgenden Einstellungen auf die passenden Werte:

-   **System Options → Password**  
    (Falls das nicht bereits mit dem Raspi Imager erledigt wurde, wähle ein neues Passwort aus Sicherheitsgründen.)
-   **Interface Options → I1 Legacy Camera → ja**  
    (Da die neue Kamera-API von picamerax nicht unterstützt wird.)
-   **Advanced Options → GL driver → G1 Legacy**  
    (Das kann eine Weile dauern, spart aber viel Energie.)
    
Beende das Konfigurationstool, indem du "Finish" auswählst, und starte den Raspberry Pi danach neu.

### Energiesparoptionen
Wir deaktivieren Bluetooth, die Kamera und die Onboard-LEDs. Dafür bearbeiten wir die Datei `/boot/config.txt`.
1. Öffne die Datei `/boot/config.txt` mit dem Texteditor `nano`:
	```bash
	sudo nano /boot/config.txt
	```
2. Die Konfigurationsdatei ist ziemlich lang. Scrolle bis zum Ende der Datei und füge folgende Zeilen hinzu, um die gewünschten Funktionen zu deaktivieren:
	```bash
	dtoverlay=disable-bt
	disable_camera_led=1
	dtparam=act_led_trigger=none
	dtparam=act_led_activelow=on
	dtparam=audio=off
	display_auto_detect=0
	dtoverlay=i2c-rtc,pcf8523
	```
Speichere die Datei mit <kbd>Strg</kbd> + <kbd>O</kbd>, bestätige mit <kbd>Enter</kbd>, und schließe den Editor mit <kbd>Strg</kbd> + <kbd>X</kbd>.
 
Um die neuen Einstellungen zu aktivieren muss der Raspberry neu gestartet werden.
```bash
sudo reboot
```

## Hardware Clock
Da der Raspberry Pi keine eigene Hardware-Uhr mitbringt, wird normalerweise davon ausgegangen, dass er über eine Internetverbindung die aktuelle Zeit abrufen kann. Wir wollen allerdings ohne WLAN überall Videos aufnehmen können, daher benötigen wir eine separate Hardware-Uhr (Real-Time Clock oder RTC). Diese enthält eine Backup-Batterie (Knopfzelle), um die Zeit zu speichern. Der Raspberry Pi verwendet dann die RTC-Zeit, um die Systemzeit einzustellen.

1. **I2C aktivieren**
	Um mit der RTC zu kommunizieren, muss I2C aktiviert werden. Du kannst dies entweder mit dem `raspi-config`-Tool machen:

	Navigiere zu:  
	    **Interface Options → I2C → Ja**
 
	Oder benutze die Kommandozeilenversion von `raspi-config`:
	```bash 
	sudo raspi-config nonint do_i2c 0
	```
3. **Überprüfen, ob die RTC funktioniert**
	Installiere i2c-tools und überprüfe, ob die RTC erkannt wird:
	```bash
	sudo apt install i2c-tools -y &&
	sudo i2cdetect -y 1
	```
Du solltest eine Ausgabe mit mehreren Zeilen sehen, in denen `UU` in einer davon erscheint. Das zeigt an, dass die RTC erfolgreich erkannt wurde.

Anschließend muss die fake-hardware Uhr noch deaktiviert werden.
```bash
	sudo apt remove fake-hwclock -y &&
	sudo update-rc.d -f fake-hwclock remove &&
	sudo systemctl disable fake-hwclock
```
Außerdem müssen bestimmte Zeilen in der Datei `/lib/udev/hwclock-set` kommentiert werden, um sicherzustellen, dass die RTC korrekt funktioniert:
- Öffne die Datei:
	`sudo nano /lib/udev/hwclock-set`
- Füge ein `#` am Anfang der folgenden Zeilen hinzu, sodass sie wie unten aussieht:
	```bash
	#!/bin/sh
	# Reset the System Clock to UTC if the hardware clock from which it
	# was copied by the kernel was in localtime.

	dev=$1

	#if [ -e /run/systemd/system ] ; then
	#    exit 0
	#fi

	#/sbin/hwclock --rtc=$dev --systz
	/sbin/hwclock --rtc=$dev --hctosys	 
	```
- Speichere die Datei mit <kbd>Strg</kbd> + <kbd>O</kbd>, bestätige mit <kbd>Enter</kbd>, und schließe den Editor mit <kbd>Strg</kbd> + <kbd>X</kbd>.

Prüfe, ob der Raspberry Pi die Zeit von der RTC abrufen kann:
```bash
sudo hwclock -r
```
Wenn die Uhrzeit korrekt angezeigt wird, funktioniert die RTC.
Falls der Raspberry Pi die Uhrzeit noch nicht automatisch synchronisiert hat, stelle sicher, dass die Zeit korrekt ist und synchronisiere sie manuell:
```bash
date
sudo hwclock -w
```

## WLAN Hotspot
Um den Raspberry Pi in der Praxis zugänglich zu machen, richten wir einen eigenen Wi-Fi-Hotspot ein. Die folgenden Skripte befinden sich alle mit im Repository im Ordner `config`. Dafür müssen wir einige Pakete installieren und konfigurieren.
1. **Benötigte Pakete installieren**

Führe den folgenden Befehl aus, um die notwendigen Pakete zu installieren:
```bash
sudo apt install hostapd dnsmasq dhcpcd -y
```
2. **`hostapd` konfigurieren**
	1.  **Datei `/etc/default/hostapd` bearbeiten:**  
	    Öffne die Datei mit:
    ```bash
    sudo nano /etc/default/hostapd
	```
	    
	2.  **Zeile 13 ändern:**  
Suche die Zeile, die mit `#DAEMON_CONF=` beginnt, und ändere sie so, dass sie auf die Konfigurationsdatei verweist. Sie sollte wie folgt aussehen:
	```bash
	DAEMON_CONF="/etc/hostapd/hostapd.conf"
	```
	    
	3.  **Änderungen speichern:**	    
Speichere die Datei mit <kbd>Strg</kbd> + <kbd>O</kbd>, bestätige mit <kbd>Enter</kbd>, und schließe den Editor mit <kbd>Strg</kbd> + <kbd>X</kbd>.

	Bearbeiten wir nun die Datei hostapd.conf, um unseren Zugangspunkt zu konfigurieren:
	```bash 
	sudo nano /etc/hostapd/hostapd.conf
	```
	Dafür die folgenden Zeilen einfügen den WLAN-Namen anpassen, ein Passwort vergeben und speichern.
	```bash 
	channel=6
	ssid=mobipi01W
	wpa_passphrase=PASSWORT
	interface=uap0
	hw_mode=g
	macaddr_acl=0
	auth_algs=1
	wpa=2
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP
	country_code=DE
	```
	Damit die Kameras einen eigenen Hotspot erzeugen und gleichzeitig noch im Büro-WLAN sein können (z.B. um Dateien im Büro zu übertragen oder um einen Internetzugang für Updates etc. zu erhalten), muss der gleiche WLAN-Kanal wie im Büro-WLAN-Netzwerk verwendet werden (im Moment Kanal 1) da der Raspberry nur eine Antenne hat.

Damit valide ip Adressen vergeben werden müssen die folgenden Zeilen der Datei  `/etc/dhcpcd.conf` angefügt werden:

```bash
    sudo nano /etc/dhcpcd.conf
```

```bash
interface uap0
    static ip_address=10.10.51.1/24
    nohook wpa_supplicant
```
Als nächstes wird `dnsmasq` konfiguriert. Dafür kann einfach die Datei aus dem Repository die alte `/etc/dnsmasq.conf` Datei mit folgendem Befehl ersetzen:
```bash
sudo mv mobivideo/config/dnsmasq.conf /etc/dnsmasq.conf
``` 

Damit alle dazugehörenden Services in der richtigen Reihenfolge ausgeführt werden werden diese erstmal deaktiviert und über ein Skript gesteuert.
```bash 
sudo systemctl unmask hostapd.service && sudo systemctl disable hostapd.service && sudo systemctl disable dhcpcd.service && sudo systemctl disable dnsmasq.service
``` 
Dann kopieren wir das Skript an die richtige Stelle:
```bash
sudo cp mobivideo/config/wifistart.sh /usr/local/bin/wifistart.sh
```

Zuletzt muss noch folgende Zeile (vor exit 0) im Skript `/etc/rc.local` hinzugefügt werden damit das `wifistart.sh` Skript bei boot ausgeführt wird.
```bash
    sudo nano /etc/rc.local
```
```bash
    /bin/bash /usr/local/bin/wifistart.sh
```
```bash
    sudo reboot
```
Anschließend wieder mit dem Raspberry verbinden (`ssh pi@mobipi01`).

## mobivideo einrichten
### Virtuelle Python Umgebung erstellen:
```bash
cd mobivideo
mkdir static
sudo apt-get install python3-venv -y
python3 -m venv venv
```
Requirements installieren:
```bash
sudo chown -R pi:pi /home/pi/mobivideo/venv/ && venv/bin/pip install -r requirements.txt
```
## Webserver einrichten
1. nginx installieren
```
sudo apt install nginx -y
```
2. Webserver-Datei öffnen
```
sudo nano /etc/nginx/sites-available/default
```
3. Anpassen zu:
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /home/pi/mobivideo/templates;
    index index.html;

    server_name mobipi09;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
4. nginx neu starten
```
sudo systemctl restart nginx.service
```
Quelle: OTCamera Setup Webserver for Preview
5. Testen
```
sudo systemctl status nginx
```
👉 Ausgabe muss grün sein.


### Einrichten des `systemd` Services
Damit die mobivideo Software automatisch bei Boot startet muss ein Service erstellt werden. Dafür kann das Skript aus dem Repository verwendet werden, indem wir es an die richtige Stelle verschieben.
```bash
sudo mv /home/pi/mobivideo/config/mobivideo.service /etc/systemd/system/mobivideo.service
```
Nun muss noch eingerichtet werden, dass der Service bei jedem Boot startet.
```bash
sudo systemctl daemon-reload
sudo systemctl start mobivideo.service
sudo systemctl enable mobivideo.service
``` 
Ob der Service richtig läuft kann mit `sudo systemctl status mobivideo.service` überprüft werden. 
Wenn es Probleme gibt, können sich die Protokolle zur Fehlersuche hier angesehen werden: 
```bash
sudo journalctl -u mobicam.service
```
Damit Datum und Uhrzeit geändert sowie WLAN zum Stromsparen in der Oberfläche ausgeschalten werden kann die `sudoers` Datei geändert werden. 
```bash
sudo visudo
```
In dieser Datei am Ende folgende Zeile einfügen und anschließend speichern.
```bash
pi ALL=(ALL) NOPASSWD: /sbin/ifconfig wlan0 down, /usr/bin/sudo ifconfig uap0 down, /usr/sbin/hwclock, /usr/bin/sudo /usr/bin/date
```

Anschließend kann das Interface in einem Browser unter der Adresse `hostname:5000` aufgerufen werden. `hostname` muss dabei durch den im Imager vergebenen ersetzt werden, z.B. `http://mobipi01:5000/`
## Uhrzeit- und Interfacekonfiguration
Aufnahmezeiten anpassen:  
```bash
nano /home/pi/mobivideo/app.py
```
```bash
'start_time': '00:00',
'stop_time': '23:59'
```

Titel in Interface anpassen:
```bash
nano /home/pi/mobivideo/templates/index.html
```
Titelzeile suchen und ändern in:
```bash
<>Schedule Recording MobipiXX<>
```

Zuletzt:
```bash
sudo reboot
```

## Erklärung Video Software
Struktur:
 ```md
.
└── home/pi/
    ├── Videos/
    └── mobivideo/
        ├── app.py
        ├── record.py
        ├── templates/
        │   └── index.html
        └── static/
            └── preview.jpg
```

### `app.py`
Funktionalitäten:
* Vorschaubild aufnehmen
* Aufnahmen planen
* Raspberry Pi Uhrzeit an Geräte Uhr angleichen (falls RTC nicht funktioniert)
* WLAN deaktivieren (um Akku zu sparen)

Die Transfer Data Funktionalität ist nicht vollständig implementiert

### `record.py`
Aufnahmen werden stündlich beendet und neu gestartet, um mögliche Fehler vorzubeugen und korrupte Video Dateien auf maximal 1h zu reduzieren. Diese Zeit kann über die Variable `SEGMENT_TIME` angepasst werden.
Ist die Kamera anders verbaut, kann ihre Rotation angepasst werden: Zeile 36: `camera.rotation = 180` 
`check_stop_flag()` wird verwendet, um Aufnahmen bei Fehlern neu zu starten, bei manueller Beendigung über das Web-Interface aber abzubrechen.
Videos werden im Format `<hostname>_FR<fps>_<YYYY-MM-DD_HH-SS>.h264` benannt, somit müssen FPS und Anfangszeit bei Konvertierung mit OTVision nicht mehr manuell angegeben werden.

## Nutzung
### Checkliste bevor es losgeht
* Akkus geladen?
* Geräte testen, ob im Ordner Videos schon was vorhanden ist.

Packliste:
* 20 Geräte in 20 Kästen
* 20 Akkus
* 20 Befestigungen
* Kreide und Maßband
  
### Am Standort
 1. Koffer am Standort öffnen und Akku anstecken. 
 2. Koffer verschließen und aufhängen.
 3. Im WLAN-Netzwerk des Geräts anmelden.
 4. Im Browser `http://<hostname>:5000` oder `http://10.42.0.1:5000` aufrufen.
 5. Mit "Capture Preview" Sichtfeld der Kamera überprüfen und eventuell Ausrichtung mit Stativkopf anpassen.
 6. Mit Kreide zwei Markierungen quer über die Straße zeichnen, möglichst weit voneinander entfernt (um Geschwindigkeit messen zu können), Abstand zwischen den Linien messen.
 7. Solange nicht der Standard Plan von 5:00-23:00 Uhr verwendet werden soll diesen anpassen. Mit "Set Schedule" bestätigen.
 8. "Disable Wi-Fi and Bluetooth" um Akku zu sparen.

### Zurück im Büro
Die aufgenommenen Videos können von den Geräten über WLAN mit `scp` heruntergeladen werden. Dafür muss ein Gerät angeschalten werden, sich mit dem WLAN Netzwerk verbunden werden und der folgende Befehl in der Konsole eingeben werden. Dabei wird erst die remote und dann die local-destination angegeben. 
```bash 
scp pi@mobipi01:~/Videos/*.h264 C:\Users\DEIN_BENUTZERNAME\Desktop\Videos\Mobipi01Test\
```
z.B.
```bash 
scp pi@mobipi01:/home/pi/Videos/*.h264 .\Desktop\Videos\Dresden
```
Daraufhin wird das Passwort des Mobipis abgefragt.
Falls einzelne Videos nicht komplett heruntergeladen wurden, können diese nocheinmal einzeln heruntergeladen werden. Dafür den Namen des Videos kopieren und damit das * (Sternchen) ersetzen. 
Anschließend können die Video Dateien auf dem Gerät gelöscht werden. Dafür mit Gerät per SSH verbinden und 
```bash
	rm Videos/*
```
eingeben.
Nun den Vorgang mit den restlichen Geräten wiederholen.

### Daten auswerten

Die h264-Dateien auf den Standcomputer laden und in einen Ordner packen. Die Pythondatei OThelper öffnen und den Pfad zu dem Ordner mit den Videos angeben. Anschließend die Länge der einzelnen Videos (im Normalfall eine Stunde -> 3600 s) angeben; dieses Feld ist auf 3600 voreingestellt.

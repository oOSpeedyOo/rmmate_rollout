# Installation und Betrieb mit Lizenz

Dieses Repository enthält die Compose-Dateien und diese Anleitung für den eigenen Betrieb der Software mit einer Lizenz des
Lizenzgebers. Quellcode ist nicht enthalten: Das Server-Image kommt fertig aus der Registry `ghcr.io`, die Agents für Windows und Linux
liefert der Server selbst aus. Den Namen, unter dem die Software bei Ihnen und Ihren Kunden erscheint, legen Sie selbst fest
(Ersteinrichtung bzw. *Einstellungen, Branding*).

**Version und Lizenz.** Lizenzschlüssel prüft der Server ab der ersten Version mit eingebautem Schlüssel des Lizenzgebers. Erst ab
dieser Version zeigt die Ersteinrichtung den Schritt „Lizenzschlüssel“ und wertet der Server `RMM_LICENSE` aus; frühere Versionen laufen
ohne Lizenz und ohne diesen Schritt. Welche Version das ist, nennt der Lizenzgeber. `RMM_IMAGE_TAG` deshalb nie auf eine ältere Version
setzen.

| Datei | Wann |
|---|---|
| `docker-compose.yml` | Standard. Der Server ist auf dem Host über `RMM_HTTP_PORT` erreichbar; der Reverse-Proxy leitet an diesen Port weiter. |
| `docker-compose.proxy.yml` | Der Reverse-Proxy läuft selbst als Container in einem eigenen Docker-Netz. Der Server hängt zusätzlich in diesem Netz, ohne Port auf dem Host; die Datenbank nur im internen Netz des Stacks. |
| `docker-compose.macvlan.yml` | Der Server bekommt eine eigene IP-Adresse im MacVLAN. |
| `.env.example` | Vorlage für die Variablen des Stacks. |

Jede Datei enthält drei Dienste: `server` (Oberfläche, REST-Schnittstelle und Verbindung zu den Agents, im Container Port 8080), `db`
(PostgreSQL, nur im Netz des Stacks) und `backup` (Datensicherung der Datenbank mit dem Schlüsselring).

## Unterlagen vom Lizenzgeber

| Unterlage | Wofür |
|---|---|
| Adresse dieses Repositorys | Stack in Portainer. |
| Name des Rollout-Benutzers und ein Zugriffstoken | Registry `ghcr.io` in Portainer. Das Token erlaubt nur das Herunterladen des Images, es ist persönlich für Ihr Haus und lässt sich einzeln widerrufen. Nicht weitergeben. |
| Lizenzschlüssel | Ersteinrichtung. Wie ein Kennwort behandeln: Wer ihn hat, kann eine Installation auf Ihre Lizenz aktivieren. |
| Kontakt | Für Fragen zur Lizenz; er steht auch in allen Hinweisen zur Lizenz in der Oberfläche. |

## Voraussetzungen

| Was | Anforderung |
|---|---|
| Docker-Host | Linux mit Docker Engine und Compose-Plugin, empfohlen mit Portainer, auf x86_64 oder ARM64 (64 Bit). 2 CPU-Kerne, 2 GB RAM und 20 GB Speicher reichen für einige hundert Geräte. |
| Adresse und TLS | Ein DNS-Name, etwa `rmm.example.de`, und ein Reverse-Proxy mit Zertifikat davor (Caddy, Traefik, nginx, Nginx Proxy Manager). Der Proxy reicht WebSockets durch, setzt `X-Forwarded-For` und `X-Forwarded-Proto` und lässt Verbindungen mindestens 120 Sekunden ohne Daten offen. Diese Angaben übernimmt der Server ab Werk nur von einem Proxy mit privater Adresse oder auf demselben Host (siehe „Reverse-Proxy und TLS“). |
| Ausgehende Verbindungen | Der Server verbindet sich über HTTPS (Port 443) mit der Lizenz-Instanz, deren Adresse im Lizenzschlüssel steht, und für Web-Push mit den Push-Diensten der Browser. Ein Proxy für ausgehende Verbindungen gilt über `HTTPS_PROXY` und `NO_PROXY`. Ohne Verbindung zur Lizenz-Instanz gibt es den Offline-Weg (siehe „Lizenz“). |
| Uhrzeit | Korrekt über NTP. Die Lizenz-Instanz nimmt Anfragen nur an, wenn die Uhr höchstens 10 Minuten abweicht; Zwei-Faktor-Codes hängen ebenfalls davon ab. |

## Installation in Portainer

1. **Registry hinterlegen** (einmal je Portainer): *Registries*, *Add registry*, *Custom registry*. Name frei, etwa `ghcr`,
   *Registry URL* `ghcr.io`, *Authentication* an, *Username* der Rollout-Benutzer, *Password* das Token.
2. **Stack anlegen:** *Stacks*, *Add stack*, Name `rmm` (die Beispiele unten nutzen diesen Namen), Methode *Repository*. Die Adresse
   dieses Repositorys, Referenz `refs/heads/main` und als Compose-Pfad `docker-compose.yml` bzw. eine der Varianten. Eine Anmeldung für
   das Repository ist nicht nötig.
3. **Variablen** unter *Environment variables* eintragen oder mit *Load variables from .env file* aus der `.env.example` laden. Pflicht
   ist nur `POSTGRES_PASSWORD` (lang und zufällig, nach dem ersten Start nicht mehr ändern), dazu bei den Varianten `RMM_PROXY_NETWORK`
   bzw. `RMM_IPV4_ADDRESS`.
4. **Bereitstellen:** *Deploy the stack*. Beim ersten Start legt der Server das Datenbankschema an; das dauert etwa eine Minute.
   Bereit ist er, wenn `http://<host>:8080/healthz/ready` mit `200` antwortet (MacVLAN: die eigene Adresse; Proxy-Variante: erst nach
   Schritt 5 über die öffentliche Adresse). `/healthz` antwortet schon vorher mit `200`; dort zeigt `"database":true` die bereite
   Datenbank.
5. **Reverse-Proxy mit TLS** davor setzen (siehe unten) und `https://<adresse>/setup` öffnen.

Ohne Portainer geht es mit Docker Compose: die Datei `.env` aus der `.env.example` anlegen, einmal `docker login ghcr.io` mit dem
Rollout-Benutzer und dem Token, dann `docker compose -p rmm -f docker-compose.yml --env-file .env up -d`.

Die Bezeichnungen unterscheiden sich je nach Version von Portainer etwas: Ältere Versionen fragen *Repository URL* und *Authentication*
direkt im Stack ab, neuere unter *Source*.

**Variablen später ändern:** den Wert in den Variablen des Stacks anpassen und den Stack neu bereitstellen (beim Speichern *Redeploy*
wählen). Erst dann startet der Server mit dem neuen Wert.

## Reverse-Proxy und TLS

Der Server spricht im Container nur HTTP auf Port 8080. Beispiel mit Caddy, das Zertifikat kommt automatisch von Let's Encrypt:

```caddy
rmm.example.de {
    reverse_proxy 192.168.1.10:8080
}
```

Hängt der Proxy im selben Docker-Netz (`docker-compose.proxy.yml`), ist das Ziel der Containername `rmm-server-1` mit Port 8080 (bei
einem anderen Stack-Namen `<stack>-server-1`). Den Dienstnamen `server` nur nehmen, wenn kein anderer Container im Netz des Proxys so
heißt. Den Namen des Netzes zeigt `docker network ls` auf dem Host oder Portainer beim Container des Proxys unter *Connected networks*.

Beispiel mit nginx:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 300s;
    client_max_body_size 300m;
}
```

**Vertrauenswürdige Proxys.** Die Adresse des Browsers und die Angabe https übernimmt der Server nur von vertrauenswürdigen Proxys, ab
Werk Loopback und private Netze (`127.0.0.0/8`, `::1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `fc00::/7`). Ein Proxy auf
demselben Host, im selben Docker-Netz oder im lokalen Netz ist damit abgedeckt. Von anderen Absendern verwirft er diese Angaben, damit
niemand mit direktem Zugriff auf Port 8080 eine fremde Adresse oder https vortäuschen kann. Erreicht Ihr Proxy den Server über eine
öffentliche Adresse, meldet die Selbstüberwachung das; unter *Einstellungen, Server & Verbindung, Reverse-Proxy* übernimmt „Als Proxy
eintragen“ die Adresse nach einem Warn-Dialog, wirksam ohne Neustart. Nur eintragen, wenn die Adresse sicher zu Ihrem Proxy gehört: Von
dort ließen sich danach Adressen und https vortäuschen. Im Notfall gibt `RMM_TRUSTED_PROXIES` die Liste vor (siehe „Alle Variablen“). Port 8080
nie direkt ins Internet geben.

## Ersteinrichtung

Solange die Einrichtung offen ist, leitet jede Seite auf `/setup` um. Die REST-Schnittstelle antwortet mit 503, und neue Geräte lassen
sich noch nicht anmelden.

1. **Einrichtungscode:** Der Server schreibt ihn beim Start als deutliche Warnung ins Container-Log, in der Zeile
   `Einrichtung offen, Code XXXX-XXXX-XXXX`: in Portainer unter *Containers*, `rmm-server-1`, *Logs*, auf dem Host mit
   `docker logs rmm-server-1` (bei einem anderen Stack-Namen `<stack>-server-1`). Zwölf Zeichen in drei Vierergruppen, etwa `ABCD-EFGH-JKMN`. Er entsteht bei jedem Start neu; nach einem
   Neustart des Containers gilt also ein neuer Code. Nach fünf Fehlversuchen von einer Adresse (bei IPv6 aus einem /64-Netz) ist die
   Eingabe dort 15 Minuten gesperrt. Für alle gesperrt wird nie, damit Dritte die Einrichtung nicht verhindern können; viele
   Fehlversuche verlangsamen nur jede Eingabe auf einige Sekunden und stehen als Warnung im Log. Ein Neustart des Containers erzeugt
   einen neuen Code und hebt alle Sperren auf.
2. **Lizenzschlüssel** (ab der ersten Version mit eingebautem Schlüssel des Lizenzgebers) einfügen und „Schlüssel prüfen“. Der Server prüft ihn sofort und zeigt Lizenznehmer, Paket, Grenzen und Laufzeit,
   danach „Jetzt online aktivieren“. Klappt das nicht, geht es mit „Später aktivieren“ weiter (der Schlüssel gilt dann 14 Tage ohne
   Aktivierung) oder mit „Offline aktivieren“ (siehe „Lizenz“).
3. **Öffentliche Adresse und Zeitzone**, vorbelegt aus der Anfrage. Die Adresse ist auch die Server-Adresse für die Agents.
4. **Erster Administrator:** E-Mail, Name und ein eigenes Kennwort. Für dieses Konto ist die Zwei-Faktor-Anmeldung Pflicht; nach der
   ersten Anmeldung richten Sie sie ein und sichern die Wiederherstellungscodes.
5. **Branding** (überspringbar): Produktname und Akzentfarbe. Logos danach unter *Einstellungen, Branding*.
6. **Abschluss:** Zusammenfassung und „Einrichtung abschließen“, danach geht es mit der Anmeldung weiter.

Anschließend in dieser Reihenfolge: *Einstellungen, Sicherheit* (Zwei-Faktor-Pflicht für alle), *Benutzer und Rollen* (persönliche Konten),
*Benachrichtigungen* (Kanäle und Regeln), *Kunden* (ersten Kunden anlegen), dann Agents über *Installations-Tokens* bzw. die Aktion
„Agent installieren“ beim Kunden ausrollen.

**Unbeaufsichtigt (ab der ersten Version mit eingebautem Schlüssel):** Mit `RMM_LICENSE` trägt der Server den Lizenzschlüssel beim Start selbst ein und aktiviert ihn kurz danach; der
Schritt entfällt. Ein ungültiger Schlüssel erscheint als Betriebswarnung, und die Ersteinrichtung fragt ihn ab. Den ersten Administrator
legt immer die Ersteinrichtung an.

## Aktualisieren

Im Stack *Pull and redeploy* wählen und *Re-pull image and redeploy* einschalten. So kommen eine geänderte Compose-Datei aus diesem
Repository und das neue Image zusammen. Beim Start bringt der Server das Datenbankschema selbst auf den neuen Stand; zurück geht es
nicht, deshalb vor größeren Sprüngen eine Sicherung anlegen (*Einstellungen, Datenbank, Sicherungen des Servers*, „Jetzt sichern“).
Die Agents aktualisieren sich danach selbst, gestaffelt in Ringen (*Einstellungen, Agent-Updates*).

Automatisch geht es mit *GitOps updates* im Stack (Mechanismus *Polling*, dazu *Re-pull image*); dann folgt die Instanz jeder Änderung
dieses Repositorys. Mit `RMM_IMAGE_TAG` auf einer Version wie `0.87.0` bleibt die Instanz auf diesem Stand, bis der Wert geändert wird.

## Lizenz

**Karte „Lizenz“.** Unter *Info*, Reiter *Instanz und Pakete*, sehen Administratoren die Stufe der Lizenz mit Grund und nächster Stufe,
Lizenznehmer, Paket und Grenzen, Vertragsende, Ablauf des Lizenzbescheids, letzte Erneuerung und den Kontakt. Dort stehen „Jetzt
prüfen“, „Lizenzschlüssel ändern“ und „Offline erneuern“. Lehnt die Lizenz-Instanz ab, nennt die Karte den Grund, etwa „gelöst“ nach
einem Umzug oder „an eine andere Installation gebunden“. Per REST: `GET /api/v1/license`.

**Erneuerung.** Nach der Aktivierung holt der Server alle 12 Stunden einen neuen Lizenzbescheid, der 30 Tage gilt. Ist die Lizenz-Instanz
einmal nicht erreichbar, läuft alles weiter; nach 3 Tagen ohne Erneuerung sehen Administratoren einen Hinweis. Änderungen an Ihrer Lizenz
(Paket, Module, Grenzen, Laufzeit) wirken mit der nächsten Erneuerung, sofort mit „Jetzt prüfen“.

**Was übertragen wird.** Lizenz-ID, Installations-ID, Version des Servers, öffentliche Adresse und Zahlen: Geräte (gesamt, Server,
Workstations), Kunden und aktive Benutzer mit den Höchstständen des laufenden Monats. Keine Namen von Kunden, Geräten oder Benutzern und
keine Adressen von Geräten.

**Stufen.** Ohne gültigen Lizenzbescheid, am Vertragsende oder nach einer Ankündigung des Lizenzgebers durchläuft die Instanz Stufen:

| Stufe | Wann (Vorgaben) | Wirkung |
|---|---|---|
| Hinweis | 3 Tage ohne Erneuerung, 30 Tage vor dem Vertragsende | Hinweis nur für Administratoren |
| Warnung | Lizenzbescheid abgelaufen (30 Tage ohne Erneuerung), Vertragsende erreicht oder vom Lizenzgeber angekündigt | Hinweis für alle Benutzer mit Datum der nächsten Stufe und Kontakt. Alles funktioniert. |
| Eingeschränkt | 14 Tage nach Beginn der Warnung | Keine neuen Geräte und Benutzer, keine Fernzugriffe (Terminal, Dateien, Registry, Dienste und Prozesse samt ihren Live-Listen, Web-Zugriff, Fernwartung), kein Steuern von Hand (Neustart und Herunterfahren, VMs, Änderungen im Active Directory), keine neuen Jobs (auch keine Software-Vorgänge von Hand), keine Änderungen an Einstellungen außer der Lizenz; schreibende REST-Aufrufe antworten mit 423. Lesen bleibt möglich, auch die Live-Ereignisanzeige, die Leistungsmessung, die angemeldeten Benutzer, die VM-Liste, die Ansicht des Active Directory und die Software-Listen. |
| Gesperrt | 30 Tage nach Beginn der Einschränkung | Die Oberfläche zeigt nur noch die Seite *Lizenz*, die REST-Schnittstelle antwortet mit 423 (außer `/api/v1/license`). |

In jeder Stufe laufen Agent-Verbindung, Checks, Alarme, Benachrichtigungen, Patch-Automatik, Agent-Updates, Datensicherung, Anmeldung
und Notfallweg weiter. Auch im Zustand Gesperrt lassen sich die Isolation von Geräten aufheben und Agents deinstallieren. Die Sperre
löscht nie Daten und zeigt den Benutzern Ihrer Kunden nichts.

**Gerätegrenze.** Ist an Ihrer Lizenz eine Grenze gesetzt, lassen sich ab der Grenze plus 10 % keine neuen Geräte mehr anmelden;
vorhandene Geräte bleiben verwaltet.

**Ohne Verbindung zur Lizenz-Instanz (offline).** Auf der Karte *Lizenz* „Offline erneuern“ wählen (in der Ersteinrichtung „Offline
aktivieren“), den Anforderungscode zwischen `-----BEGIN LICENSE REQUEST-----` und `-----END LICENSE REQUEST-----` vollständig an den
Lizenzgeber schicken und seine Offline-Antwort im selben Dialog einfügen. Eine Offline-Antwort gilt so lange, wie Ihre Lizenz es vorsieht
(Vorgabe 180 Tage). Derselbe Weg hilft, wenn die Online-Erneuerung klemmt.

**Umzug.** Mit Sicherung und Wiederherstellung (Datenbank und Schlüsselring, siehe unten) bleibt die Installation dieselbe. Eine neue
Installation ohne Sicherung aktiviert sich mit demselben Lizenzschlüssel; die Lizenz zieht automatisch um, wenn sich die alte Installation
seit 7 Tagen nicht mehr gemeldet hat, sonst gibt der Lizenzgeber sie frei. Bis dahin zeigt die neue „an eine andere Installation
gebunden“, die alte nach dem Umzug „gelöst“. Ohne Schlüsselring entsteht ein neuer Installationsschlüssel; die Installation aktiviert
sich dann bei der nächsten Erneuerung selbst neu.

## Datensicherung und Wiederherstellung

Der Dienst `backup` sichert täglich um 02:00 Uhr (Zeitzone der Instanz) die Datenbank mit `pg_dump` und den Schlüsselring als eigene Datei
und behält 14 Sicherungen; alles einstellbar unter *Einstellungen, Datenbank, Datensicherung*. Ziel ist der Datenträger `rmm-backup`
bzw. `RMM_BACKUP_PATH`, beschreibbar für UID 1654. Wer das Ziel lesen kann, kann mit dem Schlüsselring die gespeicherten Geheimnisse
entschlüsseln; das Ziel deshalb schützen (`chmod 700`) und eine Kopie außerhalb des Hosts anlegen. Ohne Schlüsselring sind Geheimnisse
in einer wiederhergestellten Datenbank unlesbar.

Wiederherstellung (Stack `rmm`, benannte Datenträger; nur in dieselbe oder eine neuere Version):

```bash
docker stop rmm-server-1
docker exec rmm-backup-1 ls -l /backup
docker exec rmm-backup-1 pg_restore --clean --if-exists --create -d postgres /backup/rmm-20261011T000000Z.dump
docker run --rm -v rmm_rmm-backup:/backup:ro -v rmm_rmm-server-keys:/keys alpine sh -c "rm -rf /keys/* && tar -xzf /backup/rmm-keys-20261011T000000Z.tar.gz -C /keys && chown -R 1654:1654 /keys"
docker start rmm-server-1
```

Datenbank und Schlüsselring immer mit demselben Zeitstempel zurückspielen. Auf einem neuen Host zuerst den Stack anlegen, die Sicherungen
in das Ziel legen, den Server stoppen und dann wie oben vorgehen.

## Notfallweg

Ist der einzige Administrator ausgesperrt (Telefon und Wiederherstellungscodes verloren, Kennwort vergessen): `RMM_RECOVER_ADMIN` auf
seine E-Mail setzen und den Stack neu bereitstellen. Beim Start setzt der Server dessen Zwei-Faktor-Anmeldung zurück und schreibt ein
neues Startkennwort einmal ins Container-Log. Danach anmelden, ein eigenes Kennwort festlegen, die Variable wieder leeren und neu
bereitstellen. Der Notfallweg funktioniert in jeder Stufe der Lizenz.

## Alle Variablen

| Variable | Pflicht | Bedeutung |
|---|---|---|
| `POSTGRES_PASSWORD` | ja | Kennwort der Datenbank. Nach dem ersten Start nicht mehr ändern. |
| `RMM_IMAGE_TAG` | nein | Version des Server-Images, Standard `latest`. Mit einer Version wie `0.87.0` bleibt die Instanz auf diesem Stand. |
| `RMM_HTTP_PORT` | nein | Port auf dem Host, Standard `8080`. Nur `docker-compose.yml`. |
| `RMM_RECOVER_ADMIN` | nein | Notfallweg (siehe oben). Leer = aus. |
| `RMM_PRODUCT_NAME` | nein | Vorgabe für den Produktnamen, solange die Ersteinrichtung bzw. die Oberfläche keinen setzt. |
| `RMM_AGENT_AUTOUPDATE` | nein | Vorgabe für automatische Agent-Updates (`true` oder `false`); die Einstellung in der Oberfläche gewinnt. |
| `RMM_LICENSE` | nein | Lizenzschlüssel für eine unbeaufsichtigte Installation, ausgewertet ab der ersten Version mit eingebautem Schlüssel des Lizenzgebers. Im Normalfall leer lassen und den Schlüssel in der Ersteinrichtung eintragen. |
| `RMM_DEV_TOOLS` | nein | `true` schaltet Testwerkzeuge ein (Neustart-Abfrage testen, Automatik-Lauf sofort). Im Alltag aus lassen. |
| `HTTPS_PROXY`, `NO_PROXY` | nein | Proxy für ausgehende Verbindungen des Servers und Ausnahmen davon. Leer = direkt. |
| `RMM_TRUSTED_PROXIES` | nein | Nur im Notfall: vertrauenswürdige Reverse-Proxys statt der Einstellung *Einstellungen, Server & Verbindung, Reverse-Proxy*. Netzliste wie `172.20.0.0/16`, `any` (jeder Absender; Adressen und https lassen sich dann fälschen, der Server warnt) oder `none`. Gesetzt gewinnt sie vor der Oberfläche. Leer = Einstellung der Oberfläche. |
| `RMM_DB_PATH`, `RMM_KEYS_PATH` | nein | Pfade auf dem Host statt benannter Datenträger; der Schlüsselpfad beschreibbar für UID 1654. |
| `RMM_BACKUP_PATH` | nein | Ziel der Datensicherung auf dem Host, etwa eine NAS-Freigabe; beschreibbar für UID 1654. |
| `RMM_IPV4_ADDRESS`, `RMM_MACVLAN_NETWORK` | nur MacVLAN | Eigene IP-Adresse und Name des vorhandenen MacVLAN-Netzes (Standard `MacVLAN`). |
| `RMM_PROXY_NETWORK` | nur Proxy-Variante | Name des vorhandenen Docker-Netzes des Reverse-Proxys. |

Alles andere (Produktname, Farben, Checks, Fristen, Patchmanagement, Fernwartung und so weiter) steht in der Oberfläche unter
*Einstellungen* und braucht keinen Neustart.

## Fehlerbilder

| Anzeichen | Ursache und Abhilfe |
|---|---|
| Pull des Images scheitert mit „unauthorized“ oder „denied“ | Registry fehlt, Benutzername oder Token falsch, Token abgelaufen oder widerrufen. Unter *Registries* den Eintrag für `ghcr.io` prüfen; ein neues Token gibt es beim Lizenzgeber. |
| Kein Einrichtungscode im Log | Der Code steht nur beim Start im Log, solange die Einrichtung offen ist. Container neu starten. |
| Eingabe des Codes gesperrt oder langsam | Fünf Fehlversuche von dieser Adresse sperren die Eingabe dort 15 Minuten; viele Fehlversuche von anderen Adressen verlangsamen sie nur. Container neu starten: neuer Code, alle Sperren aufgehoben. Den Server bis zum Abschluss der Einrichtung nur aus dem eigenen Netz erreichbar machen. |
| „Das Dokument ist mit einem Schlüssel signiert, den diese Version nicht kennt“ | Die Version des Servers ist zu alt für Ihren Lizenzschlüssel. Aktualisieren, sonst beim Lizenzgeber nachfragen. |
| „Die Signatur ist ungültig“ oder „unvollständig oder beschädigt“ | Der Lizenzschlüssel wurde beim Kopieren verändert oder gekürzt. Vollständig und unverändert einfügen. |
| Aktivierung oder Erneuerung scheitert | Ausgehende Verbindung zur Lizenz-Instanz prüfen (Firewall, `HTTPS_PROXY`). Die Karte *Lizenz* nennt den letzten Fehler. Bis zur Klärung „Später aktivieren“ bzw. den Offline-Weg nutzen. |
| „Die Uhrzeit dieses Servers weicht um mehr als 10 Minuten von der Lizenz-Instanz ab“ | Die Uhr des Hosts weicht ab. NTP einrichten. |
| „Der Lizenzgeber hat diese Installation von der Lizenz gelöst“ | Die Lizenz ist auf eine andere Installation umgezogen oder wurde für einen Umzug freigegeben. Der letzte Bescheid gilt bis zu seinem Ablauf; mit dem Lizenzgeber klären. |
| „Die Lizenz ist an eine andere Installation gebunden“ | Eine andere Installation nutzt die Lizenz. Die alte abschalten (nach 7 Tagen zieht die Lizenz um) oder den Lizenzgeber um die Freigabe bitten; bis dahin „Später aktivieren“. |
| Agents bleiben „offline“ | Der Reverse-Proxy reicht keine WebSockets durch oder trennt sie zu früh. |
| Anmeldung springt zurück auf die Anmeldeseite | `X-Forwarded-Proto` fehlt, oder der Proxy steht nicht unter *Einstellungen, Server & Verbindung, Reverse-Proxy* (die Selbstüberwachung meldet das); der Server hält die Anfrage dann für HTTP. |
| Server startet nicht, Log nennt „password authentication failed“ | `POSTGRES_PASSWORD` wurde nach dem ersten Start geändert. Alten Wert wieder eintragen. |
| „exec format error“ beim Start | Das Image passt nicht zur Architektur des Hosts. Image neu ziehen; 32-Bit-Systeme werden nicht unterstützt. |

Protokoll des Servers: `docker logs --tail 200 rmm-server-1` (Stack `rmm`).

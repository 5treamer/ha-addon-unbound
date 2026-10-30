# Unbound (recursive)

*Deutsch unten / English below.*

---

## Deutsch

### Was diese App macht

Unbound löst Namen **selbst rekursiv** auf: Root-Server → TLD-Server →
autoritative Server, mit **DNSSEC-Validierung**. Es gibt **keine
Weiterleitung** an öffentliche Resolver (1.1.1.1, 8.8.8.8 …). Forwarding
existiert nur als Notfalloption, standardmäßig aus und im Log deutlich
markiert.

Externe Verbindungen der App:

- DNS-Anfragen (UDP/TCP Port 53) an Root-, TLD- und autoritative Server.
- Optional beim Start ein HTTPS-Abruf von
  `https://www.internic.net/domain/named.root` (Option
  „Root-Hints beim Start aktualisieren“). Für diesen einen Abruf nutzt der
  Container den System-DNS von Home Assistant. Ohne diesen Abruf nutzt
  Unbound die eingebaute Root-Server-Liste.

Keine Telemetrie, keine weiteren Verbindungen.

### Installation (lokale App)

Der zuverlässige Weg, ohne Zugangsdaten im Supervisor:

1. App „Advanced SSH & Web Terminal“ oder „Samba share“ installieren.
2. Den Ordner `unbound/` aus diesem Repository in den Ordner für lokale Apps
   kopieren: `/local_apps/unbound/` (ab Supervisor 2026.07; in älteren
   Installationen bzw. älteren SSH-/Samba-Versionen heißt die Freigabe
   `addons`, also `/addons/unbound/`).
   Per SSH z. B.:
   `git clone <repo-url> /tmp/ha-addon-unbound && cp -r /tmp/ha-addon-unbound/unbound /local_apps/`
3. **Einstellungen → Apps → App-Store → ⋮ → Nach Updates suchen / Neu laden.**
4. Unter „Lokale Apps“ erscheint **Unbound (recursive)** → Installieren.
   Der Build läuft auf dem Pi (mit `make -j2`, einige Minuten).
5. Optionen prüfen (insbesondere „Erlaubte Netze“), dann starten.

**Update:** Repository aktualisieren (`git pull`), Ordner erneut nach
`/local_apps/unbound/` kopieren, `version` in `config.yaml` erhöhen, dann im
App-Store neu laden und „Aktualisieren“ bzw. „Neu bauen“ wählen.

**Privates Repo direkt als App-Repository (UNGETESTET):** Der Supervisor
reicht die Repository-URL an `git clone` weiter. Eine URL der Form
`https://<TOKEN>@github.com/<owner>/ha-addon-unbound` mit einem
*fine-grained* Token (nur dieses Repo, nur „Contents: Read“, kurze Laufzeit)
sollte deshalb funktionieren. Das ist nicht getestet. Achtung: Das Token
steht dann im Klartext in der HA-Konfiguration und in Backups. Wird das
Token gewechselt, ändert sich die URL und damit der Hash im Ordnernamen
`/app_configs/<hash>_unbound_recursive`.

### Konfigurationsreiter

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Erlaubte Netze (`access_control`) | 127.0.0.0/8, 10/8, 172.16/12, 192.168/16 | Wer anfragen darf; alles andere: `refuse`. Auf das eigene Netz verengen. 172.30.32.0/23 (HA-Docker-Netz) behalten, wenn AdGuard über 127.0.0.1 fragt. |
| `do_ip4` / `do_ip6` | an / **aus** | IPv6 nur einschalten, wenn die LTE-Verbindung IPv6 sicher kann. |
| `do_udp` / `do_tcp` | an / an | TCP an lassen (Fallback für große Antworten). |
| `num_threads` | 2 | Threads; Cache-Slabs werden automatisch passend gesetzt. |
| `msg_cache_size_mb` / `rrset_cache_size_mb` | 32 / 64 | Cache-Größen. |
| `cache_min_ttl` / `cache_max_ttl` | 0 / 86400 | Grenzen der Cache-Dauer. |
| `prefetch` | an | Beliebte Einträge vor Ablauf erneuern. |
| `serve_expired` | an | RFC 8767; Antwort aus altem Cache, wenn die Upstream-Server nach 1,8 s nicht antworten. |
| `qname_minimisation` | an | Datenschutz. |
| `aggressive_nsec` | an | RFC 8198. |
| `harden_dnssec_stripped` / `harden_glue` / `harden_below_nxdomain` | an | Härtung. |
| `harden_referral_path` | aus | Experimentell, mehr Anfragen. |
| `edns_buffer_size` | 1232 | Gegen Fragmentierung im Mobilfunk. |
| `ratelimit` | 1000 | Anfragen/s pro Zone. |
| `ip_ratelimit` | 0 (aus) | Pro Client-IP. AdGuard ist meist der einzige Client! |
| `private_address_protection` | an | DNS-Rebinding-Schutz. |
| `private_domains` | leer | Ausnahmen davon, z. B. `home.arpa`. |
| `local_zones` / `local_data` | leer | Eigene Zonen/Einträge, z. B. `nas.home.arpa. IN A 192.0.2.10`. |
| `verbosity` | 1 | 0–2. |
| `log_queries` | **aus** | Datenschutz: keine Abfrageprotokolle. |
| `log_servfail` | aus | Für die Fehlersuche einschalten. |
| `update_root_hints_on_start` | an | Siehe oben. |
| `reset_config` | aus | Einmaliges Zurücksetzen, schaltet sich selbst ab. |
| `forwarding_enabled` / `forward_addresses` | **aus** / leer | Notfalloption, siehe unten. |

**Port:** Im Container immer 5335. Den Host-Port änderst du im Abschnitt
**Netzwerk** des Konfigurationsreiters (Standard 5335). Die App lauscht nicht
auf Port 53.

**Experten-Ergänzungen:** Die Datei `custom.conf` im privaten App-Ordner
(siehe unten) wird eingebunden, falls vorhanden. Sie muss mit einer eigenen
Sektion beginnen, z. B.:

```text
server:
    log-local-actions: yes
```

Die App verändert diese Datei nie. Sie wird nur gelesen und als Kopie
(`/data/conf/custom.conf`) eingebunden. Fehler darin verhindern den Start mit
klarer Meldung im Log.

### AdGuard Home einrichten

AdGuard läuft im Host-Netz. Unter **Einstellungen → DNS-Einstellungen →
Upstream-DNS-Server** eintragen:

```text
<pi-ip>:5335
```

(alternativ `127.0.0.1:5335`; dann muss 172.30.32.0/23 in „Erlaubte Netze“
stehen, das ist im Standard enthalten). Andere Upstreams entfernen, sonst
fragt AdGuard parallel öffentliche Resolver. **Bootstrap-DNS** ist davon
unabhängig und wird nur für DoH/DoT-Upstreams benötigt.

### Wo liegen die Dateien?

| Pfad im Container | Pfad in HAOS | Inhalt |
| --- | --- | --- |
| `/data/conf/unbound.conf` | privat (Supervisor) | wird bei jedem Start neu erzeugt |
| `/data/state/root.key` | privat (Supervisor) | DNSSEC-Trust-Anchor (RFC 5011) |
| `/data/state/root.hints` | privat (Supervisor) | optional geladene Root-Hints |
| `/config/custom.conf` | `/app_configs/<repo>_unbound_recursive/custom.conf` | optional, gehört dir |

Bei lokaler Installation ist `<repo>` = `local`. Die App schreibt **nicht** in
den HA-Konfigurationsordner, `/share`, `/ssl` oder `/backup`.

**Deinstallation:** Beim Deinstallieren „Konfigurationsordner/App-Daten
löschen“ wählen. `/data` wird immer entfernt, mit der Option auch
`/app_configs/<repo>_unbound_recursive`. Danach im Terminal prüfen:

```text
ls /app_configs/ | grep unbound      # sollte leer sein
ls /homeassistant/unbound 2>/dev/null # sollte nicht existieren
```

### Migration vom alten Unbound-Addon

1. Altes Addon stoppen und deinstallieren.
2. Alte Konfiguration nicht sofort löschen, sondern beiseitelegen:
   `mv /homeassistant/unbound /homeassistant/unbound.old` (früher `/config/unbound`).
3. Diese App installieren und testen. Erst wenn alles läuft:
   `rm -r /homeassistant/unbound.old`.

### Tests nach der Installation

```text
dig @<pi-ip> -p 5335 example.com +time=3 +tries=1          # NOERROR
dig @<pi-ip> -p 5335 dnssec-failed.org +time=3 +tries=1    # SERVFAIL (DNSSEC kaputt -> blockiert)
dig @<pi-ip> -p 5335 sigok.verteiltesysteme.net +dnssec    # NOERROR, Flag "ad"
dig @<pi-ip> -p 5335 example.com +tcp                      # TCP funktioniert
dig @<pi-ip> -p 5335 version.bind CH TXT                   # REFUSED (Version versteckt)
```

Von einem Rechner außerhalb der erlaubten Netze muss jede Anfrage `REFUSED`
liefern.

### Diagnose: direkte Auflösung über LTE funktioniert nicht

Ausführen im Terminal der App (`docker exec`) oder auf einem Rechner im
gleichen Netz:

1. **Root-Server per UDP erreichbar?**
   `dig @198.41.0.4 . NS +norec +time=3 +tries=1` → `NOERROR` mit 13 NS-Einträgen.
   Ein Timeout bedeutet, dass ausgehender UDP-Port 53 gefiltert wird.
2. **Per TCP?** `dig @198.41.0.4 . NS +norec +tcp` → Timeout bedeutet: TCP/53 gefiltert.
3. **Manipulation?** `dig @198.41.0.4 example.com +norec` muss eine
   *Referral* (AUTHORITY-Sektion mit `.com`-Servern) liefern, **keine**
   direkte Antwort und kein `NXDOMAIN`. Eine direkte Antwort bedeutet, dass
   der Anbieter DNS abfängt (transparenter Proxy).
4. **Ganzer Pfad:** `dig +trace example.com`.
5. **Fragmentierung:** `dig @198.41.0.4 . DNSKEY +dnssec +bufsize=1232` und
   `+bufsize=4096` vergleichen. Wenn 4096 einen Timeout bringt und 1232
   funktioniert, fehlen fragmentierte Pakete. Dann `edns_buffer_size` bei
   1232 lassen oder kleiner einstellen.
6. **DNSSEC:** `log_servfail` einschalten, `verbosity` 2, Log prüfen.
7. **IPv6:** Nur aktivieren, wenn `dig -6 @2001:503:ba3e::2:30 . NS +norec` funktioniert.

Funktioniert die direkte Auflösung dauerhaft nicht, ist `forwarding_enabled`
eine **bewusste, manuelle** Entscheidung. Die App aktiviert das nie
automatisch. Im Log erscheint dann ein deutlicher Warnblock.

### Unbound aktualisieren (Sicherheitsupdates manuell!)

Diese App aktualisiert Unbound **nicht** selbst. Bei einer neuen
Unbound-Version (Changelog und Sicherheitsmeldungen auf nlnetlabs.nl
beobachten):

1. In `Dockerfile` `UNBOUND_VERSION` und `UNBOUND_COMMIT` (Commit des Tags
   `release-X.Y.Z`, z. B. per `git ls-remote https://github.com/NLnetLabs/unbound refs/tags/release-X.Y.Z^{}`) anpassen.
2. Optional das Basis-Image (`BASE_IMAGE`, Tag **und** Digest) anheben.
3. `version` in `config.yaml` erhöhen, `CHANGELOG.md` ergänzen.
4. Neu bauen (siehe Update oben).

**Lieferkette:** Der Quellcode wird per Git vom offiziellen NLnet-Labs-
Repository geholt und auf den gepinnten Commit-Hash geprüft. Die
SHA256-Prüfsumme und PGP-Signatur des Release-Tarballs werden auf Wunsch des
Repo-Eigentümers **nicht** geprüft. Die Vertrauensbasis ist damit GitHub
plus Commit-Hash (Git verwendet SHA-1 mit Kollisionserkennung).

---

## English

### What it does

Unbound resolves names **recursively by itself** (root → TLD →
authoritative) with **DNSSEC validation**. There is **no forwarding** to public
resolvers. Forwarding exists only as an emergency option, off by default and
clearly marked in the log. The only external connections are DNS traffic and an
optional HTTPS download of `named.root` from internic.net at start.

### Installation

Copy the `unbound/` folder to `/local_apps/unbound/` (older systems:
`/addons/unbound/`) via the SSH or Samba app. Then go to **Settings → Apps →
App store → ⋮ → Check for updates**, install "Unbound (recursive)" from
"Local apps", review the options and start it. To update: `git pull`, copy
the folder again, bump `version` in `config.yaml`, reload and rebuild.
Installing the private repository via a token URL is possible in principle,
but **untested**. The token would be stored in plain text in the HA config.

### Configuration

All options are described in the configuration tab (German and English
translations). Defaults work out of the box. Narrow "Allowed networks" to your
own LAN, but keep `172.30.32.0/23` if AdGuard uses `127.0.0.1:5335`. The
container port is always 5335; change the host port in the **Network**
section. Optional expert settings go into `custom.conf` in the app config
folder (`/app_configs/<repo>_unbound_recursive/`). It must start with its
own clause (`server:`) and is never modified by the app.

### AdGuard Home

Set the upstream DNS server to `<pi-ip>:5335` and remove other upstreams.
Bootstrap DNS is independent of this.

### Files and cleanup

Generated files live in the private `/data` folder, which is removed on
uninstall. `custom.conf` lives in `/app_configs/<repo>_unbound_recursive/`,
which is removed if you select the option to delete the app's configuration
when uninstalling. Nothing is written to the HA config folder, `/share`,
`/ssl` or `/backup`. When migrating from an old add-on, move
`/homeassistant/unbound` aside with `mv`, and delete it only after this app
works.

### Tests and diagnostics

See the `dig` commands in the German section (identical). Expected results:
`dnssec-failed.org` → SERVFAIL, `sigok.verteiltesysteme.net` → NOERROR with
the `ad` flag.

### Updating Unbound

Security updates are **manual**. Update `UNBOUND_VERSION` and
`UNBOUND_COMMIT` in the `Dockerfile`, bump `version`, rebuild. The source is
fetched via git and checked against the pinned commit hash. Tarball
checksum and PGP signature are not verified (owner's decision).

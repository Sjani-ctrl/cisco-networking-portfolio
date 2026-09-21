---
tags: [ccna, modul2-network-access, capstone, projekt, portfolio, etherchannel, stp, wlc]
modul: "2 - Network Access (Capstone-Projekt)"
thema: "NetCorp Capstone-Erweiterung - EtherChannel, STP Root Bridge, WLC, Marketing-WLAN"
---

# 🏢 Capstone-Erweiterung: NetCorp GmbH (Modul 2 komplett)

> Erweiterung des ursprünglichen NetCorp-Projekts um EtherChannel, bewusste STP-Root-Bridge-Wahl, WLC/CAPWAP und eine 5. Abteilung (Marketing) mit WLAN-Zugang.

## Was neu hinzugekommen ist

| Bereich | Vorher | Jetzt |
|---|---|---|
| SW-A ↔ SW-CORE | 1 Trunk-Link | EtherChannel (Po1, LACP active) |
| SW-B ↔ SW-CORE | 1 Trunk-Link | EtherChannel (Po2, LACP active) |
| Root Bridge | zufällig (MAC-basiert) | Bewusst auf SW-CORE (Priorität 4096) |
| Abteilungen | 4 (kabelgebunden) | + Marketing (VLAN 50, WLAN) |
| Neue Geräte | – | WLC-2504, Lightweight AP, Autonomous AP, 2 Laptops |

## Neues VLAN

| Abteilung | VLAN-ID | Subnetz | Gateway |
|---|---|---|---|
| Marketing (WLAN) | 50 | 192.168.50.0/24 | 192.168.50.1 |
| Wireless-Mgmt | 100 | 192.168.1.0/28 | 192.168.1.1 |

---

## Konfigurationsschritte (Kernbefehle)

### EtherChannel SW-A/SW-B ↔ SW-CORE

```
interface range gigabitEthernet 0/1-2
speed 100 (oder 1000, je nach Hardware)
duplex full
switchport trunk encapsulation dot1q
switchport trunk native vlan 999
switchport mode trunk
switchport nonegotiate
channel-group 1 mode active
exit
interface port-channel 1
switchport trunk native vlan 999
switchport mode trunk
```

### STP Root Bridge (SW-CORE)

```
spanning-tree vlan 10,20,30,40,999 priority 4096
```

### DHCP für Wireless-Mgmt (R1)

```
interface gigabitEthernet 0/0.100
encapsulation dot1q 100
ip address 192.168.1.1 255.255.255.240
exit
ip dhcp excluded-address 192.168.1.1 192.168.1.3
ip dhcp pool VLAN100-POOL
network 192.168.1.0 255.255.255.240
default-router 192.168.1.1
```

### Marketing-Subinterface (R1)

```
interface gigabitEthernet 0/0.50
encapsulation dot1q 50
ip address 192.168.50.1 255.255.255.0
```

---

## ✅ Testplan

| Test | Ergebnis |
|---|---|
| `show etherchannel summary` (beide Bündel) | ✅ Beide Po1/Po2 mit "P" (bundled) |
| `show spanning-tree vlan 10` | ✅ SW-CORE "This bridge is the root" |
| Lightweight AP → WLC Registrierung | ✅ Nach DHCP-Fix erfolgreich |
| Marketing-Laptop → Marketing-Laptop (WLAN) | ✅ Erfolgreich (via Autonomous AP) |
| Marketing → andere Abteilungen / Internet | ✅ |

---

## 🐛 Troubleshooting-Fälle (5 reale Probleme gelöst)

### Fall 1: EtherChannel Duplex-Mismatch
**Symptom:** `%EC-5-CANNOT_BUNDLE2: Gig0/1 is not compatible with Gig0/2 ... duplex of Gig0/1 is half, Gig0/2 is full`
**Ursache:** Auto-Negotiation hat auf beiden Interfaces unterschiedliche Duplex-Werte ausgehandelt.
**Lösung:** `duplex full` und `speed` explizit auf beiden Interfaces gleichzeitig gesetzt (`interface range`).
**Lernpunkt:** EtherChannel verlangt exakt identische Einstellungen auf allen Mitgliedsports — nie auf Auto-Negotiation verlassen.

### Fall 2: Hardware-Port-Limit (keine freien Gigabit-Ports)
**Symptom:** SW-CORE hatte keine freien GigabitEthernet-Ports mehr für die zweite EtherChannel-Verbindung.
**Ursache:** 2960-Switches haben nur 2 Gigabit-Ports; beide waren schon für SW-A belegt.
**Lösung:** FastEthernet-Ports auf SW-CORE-Seite genutzt, Speed auf beiden Enden explizit auf `100` gesetzt (Gigabit-Port kann sich auf 100 Mbit/s herunterhandeln).
**Lernpunkt:** Port-Typ-Namen müssen auf beiden Kabelenden nicht identisch sein — nur Speed/Duplex müssen übereinstimmen.

### Fall 3: Native VLAN Mismatch nach Neukonfiguration
**Symptom:** `%CDP-4-NATIVE_VLAN_MISMATCH` zwischen SW-B und SW-CORE trotz vermeintlich korrekter Config.
**Ursache:** Native-VLAN-Einstellung war nur auf einem der beiden Mitglieds-Interfaces angekommen, nicht auf beiden.
**Lösung:** Komplette Neukonfiguration mit `shutdown` → `no channel-group` → alle Einstellungen explizit neu mit `interface range` gesetzt → `no shutdown`.
**Lernpunkt:** Bei hartnäckigen EtherChannel-Fehlern hilft ein sauberer Reset (shutdown, Channel-Group entfernen, alles neu) zuverlässiger als einzelne Korrekturen.

### Fall 4: Lightweight AP registriert sich nicht beim WLC (0 APs)
**Symptom:** Alle Switch-Interfaces up/up, AP-Konfiguration korrekt, trotzdem "Number of APs: 0" auf dem WLC.
**Ursache:** Lightweight AP hat kein manuelles IP-Eingabefeld — er bezieht seine Management-IP zwingend per DHCP. Es gab keinen DHCP-Server im Wireless-Mgmt-VLAN.
**Lösung:** DHCP-Pool auf R1 für das Subnetz 192.168.1.0/28 eingerichtet.
**Lernpunkt:** Bei Geräten ohne manuelles IP-Feld (Lightweight APs, viele IoT-Geräte) zuerst prüfen, ob überhaupt ein DHCP-Server im jeweiligen VLAN existiert.

### Fall 5: WLAN-Client verbindet sich nicht (Frequenzband-Mismatch)
**Symptom:** Laptop-WLAN-Adapter zeigte dauerhaft "Adapter is Inactive", keine Netzwerke in der Suche sichtbar.
**Ursache 1:** Autonomous-AP-Konfiguration wurde zunächst auf "Port 1" vorgenommen, was sich als 5-GHz-Funkmodul herausstellte — der Laptop-Adapter (WPC300N) unterstützt nur 2,4 GHz.
**Ursache 2:** Ein weiteres AP-Modell hatte gar kein 2,4-GHz-Funkmodul (Port 0 war dort nur der kabelgebundene Ethernet-Port).
**Lösung:** Wechsel zu einem AP-Modell mit passendem 2,4-GHz-Funkmodul (AccessPoint-PT-AC), dort SSID/Security korrekt konfiguriert.
**Lernpunkt:** Frequenzband-Kompatibilität zwischen AP und Client-Adapter ist eine Grundvoraussetzung, die vor jeder Security-/SSID-Fehlersuche geprüft werden sollte.

## ⚠️ Dokumentierte Simulationseinschränkung

Die vollständige Client-WLAN-Verbindung über einen **Lightweight AP + WLC** (Split-MAC-Architektur) konnte in Cisco Packet Tracer trotz erfolgreicher CAPWAP-Registrierung des APs nicht hergestellt werden — mehrere WLC-Konfigurationsseiten (z.B. 802.11b/g/n Network Enable) zeigen explizit "This feature is not supported in Packet Tracer". Dies ist eine bekannte, dokumentierte Grenze des Simulators, keine Fehlkonfiguration. Für den funktionierenden End-to-End-Test wurde daher ein zusätzlicher **Autonomous AP** eingesetzt, dessen Client-Konnektivität vollständig simulierbar ist.

---

## Fachbegriffe (Englisch – Albanisch)

| Englisch | Albanisch |
|---|---|
| Duplex Mismatch | Papërputhje Duplex |
| Native VLAN Mismatch | Papërputhje e VLAN-it vendas |
| DHCP Pool | Grupi DHCP |
| Frequency Band Mismatch | Papërputhje e brezit të frekuencës |
| Split-MAC | MAC e Ndarë |

---

## 🔗 Verlinkung
- Basiert auf: [[Modul2-Projekt-NetCorp-Firmennetzwerk]]
- Lektionen: [[Modul2-Lektion5-EtherChannel]], [[Modul2-Lektion4-Spanning-Tree-Protocol]], [[Modul2-Lektion6-Wireless-Architekturen]], [[Modul2-Lektion7-WLC-Konfiguration]]
- Modul-Übersicht: [[Modul2-Network-Access]]

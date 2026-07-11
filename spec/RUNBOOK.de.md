# Laden zu Hause — Kurzanleitung

*(Für den Alltag. Technik-Details stehen in den englischen Docs daneben.)*

## App öffnen

**Einfachster Weg (überall, auch unterwegs):** Lesezeichen „Laden" auf dem Homescreen
öffnen — fertig. *(Adresse: evcc-Fernzugriff-URL, wird bei der Einrichtung eingetragen —
TODO)*

**Alternative für Liam (Admin):**
1. Zu Hause (WLAN): Browser → `http://<pi-ip>:7070`.
2. Unterwegs: erst WireGuard-App öffnen → Tunnel einschalten → dann dieselbe Adresse.

## Auto laden

Auto einstecken — evcc erkennt automatisch, welches Auto es ist (steht oben in der App).

Dann einen Modus wählen (Knöpfe unten):

| Knopf | Bedeutung |
|---|---|
| **Schnell** | Sofort mit voller Leistung laden — egal was der Strom kostet. Für „ich muss gleich los". |
| **Solar** | Nur mit Sonnenstrom-Überschuss laden. Günstigster Modus, kann dauern. |
| **Min+Solar** | Lädt langsam durchgehend, nutzt Sonne wenn da. |
| **Aus** | Nicht laden. |

## „Bis morgen früh voll"

1. In der App auf das Auto tippen → **Ladeplan**.
2. Abfahrtszeit und Ziel-Prozent einstellen (z. B. 80 % bis 07:00).
3. evcc sucht selbst die günstigsten Stunden (Sonne + Börsenpreis). Fertig.

## Wenn etwas nicht geht

- **App lädt nicht (unterwegs):** Kurz warten und neu laden; sonst Liam fragen. (Bei der
  WireGuard-Variante: Tunnel an? Sonst an/aus schalten.)
- **Falsches Auto angezeigt:** In der App auf den Autonamen tippen und das richtige wählen.
- **Lädt nicht im Solar-Modus:** Zu wenig Sonne ist normal — Modus **Schnell** wählen, wenn
  es eilig ist.
- **Gar nichts geht:** Steckdose/Sicherung vom Raspberry Pi prüfen, einmal aus- und
  einstecken, 2 Minuten warten.

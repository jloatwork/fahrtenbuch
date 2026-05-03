# waterdrop·drive – Fahrtenbuch

Fahrtenbuch & Reisekostenabrechnung für Janos Lobos (WDP).  
Gebaut als Single-Page-App, gehostet auf GitHub Pages, Datenbank auf Supabase.

---

## App aufrufen

**URL:** https://jloatwork.github.io/fahrtenbuch/

Funktioniert im Browser auf Handy und Desktop. Auf iOS/Android: im Browser öffnen → „Zum Homescreen hinzufügen" → verhält sich wie eine native App.

---

## Architektur

```
Browser (index.html)
    │
    ├── Login/Auth     →  Supabase Auth (Email + Passwort)
    ├── Daten lesen    →  Supabase Postgres (REST API)
    ├── Daten schreiben → Supabase Postgres (REST API)
    └── Offline-Cache  →  localStorage (automatisch)
```

| Schicht | Service | Details |
|---|---|---|
| Frontend | GitHub Pages | `main` Branch, `index.html` im Root |
| Datenbank | Supabase Postgres | Projekt: `bdunrvkypzwadtthrfmi` |
| Auth | Supabase Auth | Email + Passwort |
| GPS-Adressen | Nominatim (OpenStreetMap) | Reverse Geocoding |
| Excel-Export | SheetJS (CDN) | `.xlsx` Download |

---

## Zugänge

| System | URL | Login |
|---|---|---|
| App | https://jloatwork.github.io/fahrtenbuch/ | Email + Passwort (in der App) |
| GitHub Repo | https://github.com/jloatwork/fahrtenbuch | GitHub Account `jloatwork` |
| Supabase Dashboard | https://bdunrvkypzwadtthrfmi.supabase.co | Supabase Account (GitHub SSO) |

---

## Datenbankstruktur (Supabase)

### Tabelle `trips` – Fahrten

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | uuid | Primärschlüssel (auto) |
| `user_id` | uuid | Verknüpfung mit Auth-User |
| `date` | text | Format: `YYYY-MM-DD` |
| `time` | text | Abfahrtszeit `HH:MM` |
| `time_arrival` | text | Ankunftszeit `HH:MM` |
| `from_place` | text | Abfahrtsort |
| `to_place` | text | Ankunftsort |
| `vehicle` | text | Fahrzeugkennzeichen |
| `purpose` | text | Zweck der Reise |
| `km_start` | numeric | KM-Stand bei Abfahrt |
| `km_end` | numeric | KM-Stand bei Ankunft |
| `km` | numeric | Gefahrene Kilometer |
| `created_at` | timestamptz | Erstellungsdatum (auto) |

### Tabelle `kw_days` – Verpflegungstage pro KW

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | uuid | Primärschlüssel (auto) |
| `user_id` | uuid | Verknüpfung mit Auth-User |
| `kw_key` | text | Format: `YYYY-WW` (z.B. `2026-13`) |
| `days` | integer | Anzahl Verpflegungstage |

**Row Level Security (RLS)** ist aktiviert – jeder User sieht nur seine eigenen Daten.

---

## Abrechnungslogik

| Posten | Satz | Basis |
|---|---|---|
| Kilometergeld | € 0,50 / km | netto |
| Verpflegungspauschale | € 45,00 / Tag | pro Verpflegungstag (manuell in KW-Übersicht eingetragen) |

---

## Code-Änderungen deployen

1. `index.html` lokal bearbeiten
2. Commit & Push auf `main`:
   ```bash
   git add index.html
   git commit -m "Beschreibung der Änderung"
   git push origin main
   ```
3. GitHub Pages deployed automatisch – nach ~1 Minute ist die Änderung live.

---

## Daten manuell importieren / reparieren

Falls Daten fehlen oder korrigiert werden müssen: Supabase Dashboard → **SQL Editor**.

**Alle Fahrten ersetzen:**
```sql
DELETE FROM trips WHERE user_id = (SELECT id FROM auth.users LIMIT 1);

INSERT INTO trips (id, user_id, date, time, time_arrival, from_place, to_place, vehicle, purpose, km_start, km_end, km)
VALUES (gen_random_uuid(), (SELECT id FROM auth.users LIMIT 1), 'YYYY-MM-DD', 'HH:MM', 'HH:MM', 'Von', 'Nach', 'FORD Focus S-147XX', 'Zweck', 0, 0, 0);
```

**Verpflegungstage setzen:**
```sql
DELETE FROM kw_days WHERE user_id = (SELECT id FROM auth.users LIMIT 1);

INSERT INTO kw_days (user_id, kw_key, days) VALUES
  ((SELECT id FROM auth.users LIMIT 1), '2026-13', 3),
  ((SELECT id FROM auth.users LIMIT 1), '2026-14', 2);
```

Nach SQL-Änderungen in der App auf **Sync** drücken.

---

## Passwort zurücksetzen

Supabase Dashboard → **Authentication → Users** → User auswählen → „Send password recovery".

---

## Bekannte Einschränkungen

- **SheetJS** für Excel-Export kommt von einem externen CDN. Falls nicht erreichbar, funktioniert nur der Excel-Export nicht – die App selbst läuft weiter.
- **Nominatim** (GPS-Adressen) hat ein Rate-Limit. Bei blockierter IP einfach Adresse manuell eintippen.
- **Offline:** Neue Fahrten werden sofort lokal in `localStorage` gespeichert und beim nächsten Online-Sync zu Supabase hochgeladen.

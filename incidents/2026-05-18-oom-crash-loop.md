# Lärdom #001 — OOM-kaskad från obegränsad crash-loop

**Datum:** 2026-05-18  
**Påverkan:** Hetzner VPS otillgänglig ~20:08–07:36 CEST. Alla user-services dödades av OOM-killern, inkl. dbus.  
**Projekt:** AJ-bevakning, Mälartåg (maltag), generell VPS-infrastruktur

---

## Vad hände

1. `aj-bevakning-dashboard.service` försökte starta men port 8095 var redan upptagen av en zombie-instans (troligen en manuell restart utan att gamla processen dog ordentligt).
2. Varje försök misslyckades med `[Errno 98] address already in use` och avslutade med exit-kod 1.
3. Systemd tolkade det som `on-failure` → omstart var 5 sekunder.
4. **Ingen `StartLimitBurst` var satt** → systemd omstartade i princip obegränsat, timme efter timme.
5. Varje Python/uvicorn-process tog ~2 MB vid uppstart — hundratals processer per timme tömde RAM-bufferten.
6. OOM-killern slog till ~21:32 och tog sedan `maltag`, `aj-bevakning-fetch` och till slut `dbus` i kaskad.
7. Ingen swap fanns → noll buffert, direkt hård kill utan varning.

Hög CPU-användning var ett **symptom** (massor av korta processer + OOM-kerner-arbete), inte orsaken. Hetzner cost-optimised CPU-throttling var **inte** inblandat.

---

## Rotkorsak

```ini
# Vad service-filen hade (fel):
Restart=on-failure
RestartSec=5
# Ingen StartLimitBurst → obegränsad crash-loop
```

---

## Åtgärd (genomförd 2026-05-19)

Alla user-services på Hetzner-VPS:en fick:

```ini
Restart=on-failure   # eller always
RestartSec=10
StartLimitBurst=5
StartLimitIntervalSec=120
```

Innebär: max 5 omstarter per 2-minutersperiod, sedan ger systemd upp och tjänsten hamnar i `failed`-state tills manuell `systemctl --user reset-failed && systemctl --user start <tjänst>`.

---

## Checklista för alla framtida systemd user-services

Kopiera detta skelett. Fyll i det som faktiskt behövs, ta inte bort raden med `StartLimitBurst`.

```ini
[Unit]
Description=<beskrivning>
After=network-online.target

[Service]
Type=simple
WorkingDirectory=<sökväg>
ExecStart=<kommando>
Restart=on-failure
RestartSec=10
StartLimitBurst=5
StartLimitIntervalSec=120

[Install]
WantedBy=default.target
```

---

## Öppna risker

| Risk | Status |
|---|---|
| Ingen swap på VPS (3,7 GB RAM, 0 swap) | Öppen — lägg till 1 GB swapfile |
| Vad skapade zombie på port 8095? | Oklart — troligen `systemctl --user restart` utan att vänta på clean shutdown |
| Övriga services utan `StartLimitBurst`? | Kontrollera med `grep -rL StartLimitBurst ~/.config/systemd/user/` |

---

## Falsifieringsvillkor

Lärdomen är inaktuell om:
- Systemd ändrar default för `StartLimitBurst` i user-läge till ett ändligt värde.
- VPS:en får tillräckligt med RAM (>16 GB) att en crash-loop aldrig kan tömma minnet innan den självbegränsas.

---

## Kommandon att komma ihåg

```bash
# Se om tjänst loopat och hamnat i failed
systemctl --user status <tjänst>

# Återstarta manuellt efter StartLimit-stopp
systemctl --user reset-failed <tjänst>
systemctl --user start <tjänst>

# Hitta services som saknar StartLimitBurst
grep -rL StartLimitBurst ~/.config/systemd/user/*.service

# Snabb OOM-historik
journalctl --user --since "yesterday" -p warning --no-pager | grep -i "oom\|kill\|failed"

# Kolla vad som lyssnar på en port
ss -tlnp | grep 8095
```

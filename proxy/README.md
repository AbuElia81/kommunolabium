# Vermittler einrichten

Die beiden Instrumentenseiten liegen als statisches HTML auf GitHub Pages und
können die Anthropic-API nicht direkt aufrufen: Ein Schlüssel im Quelltext wäre
für jeden Besucher lesbar, und der Browser blockt Direktaufrufe ohnehin per CORS.

`worker.js` ist ein kleiner Vermittler, der den Schlüssel serverseitig hält. Er
kennt genau zwei Anfragen — Wort klassifizieren, Wort deuten — und trägt die
Prompts selbst. Wer die URL findet, kann das Instrument benutzen, aber keine
beliebigen Anfragen auf deine Rechnung an die API schicken.

Auf diesem Rechner ist kein Node und kein npm installiert. Die folgende
Anleitung braucht beides nicht — sie läuft komplett im Browser.

---

## 1. Worker anlegen (Cloudflare, kostenlos)

1. Auf [dash.cloudflare.com](https://dash.cloudflare.com) anmelden (oder ein
   kostenloses Konto anlegen — das musst du selbst tun, ich kann keine Konten
   erstellen).
2. **Compute (Workers)** → **Create** → **Start from Hello World!** → **Deploy**.
3. Namen vergeben, z. B. `kommunolabium-proxy`. Die URL lautet danach
   `https://kommunolabium-proxy.<dein-name>.workers.dev`.
4. **Edit code** öffnen, den gesamten Inhalt von `worker.js` aus diesem Ordner
   hineinkopieren (den vorhandenen Beispielcode ersetzen), **Deploy**.

## 2. Schlüssel hinterlegen

Im Worker: **Settings** → **Variables and Secrets** → **Add**

| Feld  | Wert                                             |
| ----- | ------------------------------------------------ |
| Typ   | **Secret** (nicht *Text* — sonst steht er im Klartext) |
| Name  | `ANTHROPIC_API_KEY`                              |
| Wert  | dein Schlüssel aus der [Anthropic Console](https://console.anthropic.com/settings/keys) |

Danach **Deploy**. Den Schlüssel bitte selbst eintragen und mir nicht schicken.

## 3. URL in die beiden Seiten eintragen

In `kommunikationslabium.html` und `astrolabium.html` steht jeweils oben im
Skript:

```js
const PROXY="PROXY_URL_HIER_EINSETZEN";
```

Dort die Worker-URL einsetzen, **ohne Schrägstrich am Ende**:

```js
const PROXY="https://kommunolabium-proxy.dein-name.workers.dev";
```

Solange der Platzhalter steht, zeigen die Seiten den Hinweis
„Kein Vermittler eingetragen“ statt eines Netzwerkfehlers.

## 4. Prüfen

Lokal auf Port 8917 oder live auf abuelia81.github.io — ein Wort eingeben, das
nicht im Lexikon steht (z. B. *Heimweh*): Die Scheibe muss sich drehen **und**
ein Text im Antwortfeld erscheinen.

---

## Was der Vermittler prüft

- **Herkunft**: Nur `abuelia81.github.io` und `localhost:8917` werden bedient.
  Weitere Adressen in `ERLAUBTE_HERKUNFT` in `worker.js` ergänzen.
- **Länge**: Eingaben über 80 Zeichen werden abgewiesen — das Instrument
  verarbeitet Wörter, keine Texte.
- **Domäne**: Muss aus der Tabelle des jeweiligen Instruments stammen.
- **Fremde Anweisungen**: Der Systemprompt sagt ausdrücklich, dass das
  eingegebene Wort Gegenstand der Deutung ist und keine Anweisung.

Die Herkunftsprüfung hält Browser fern, keine Skripte — ein `Origin`-Header
lässt sich fälschen. Wenn das Instrument öffentlich läuft, lohnt zusätzlich eine
**Rate limiting**-Regel in den Worker-Einstellungen, z. B. 20 Anfragen pro
Minute und IP.

## Modell und Kosten

Der Vermittler nutzt `claude-opus-5` mit `effort: "low"` für beide Schritte.
Pro unbekanntem Wort fallen zwei Aufrufe an (klassifizieren + deuten), pro
bekanntem Wort einer. Wenn dir das zu teuer wird, sind zwei Stellschrauben in
`worker.js` sinnvoll:

- `MODELL` auf `claude-sonnet-5` setzen — deutlich günstiger, für diese Aufgabe
  gut geeignet.
- Für die Klassifikation ein eigenes, kleineres Modell verwenden
  (`claude-haiku-4-5`); sie ist nur eine Zuordnung zu acht Kennungen.

## Domänentabelle synchron halten

`worker.js` führt die Domänen beider Instrumente noch einmal, damit die Prompts
serverseitig entstehen. Wenn du in einer HTML-Datei eine Domäne, ein Bildschema
oder eine Beschreibung änderst, muss dieselbe Änderung in `worker.js` nachgezogen
werden — sonst deutet das Instrument gegen eine veraltete Beschreibung.

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

## Schale und Pol

`/deute` nimmt zwei weitere Felder entgegen, beide freiwillig:

| Feld     | Erlaubte Werte                              | Voreinstellung |
| -------- | ------------------------------------------- | -------------- |
| `schale` | `Inhalt`, `Epistemisch`, `Sprechakt`        | `Inhalt`       |
| `pol`    | `Darstellung`, `Ausdruck`, `Appell`         | `Darstellung`  |

Die Sprachkugel schickt beide mit; die beiden älteren Seiten kennen sie nicht und
bekommen die Voreinstellung — ihre Antworten bleiben also unverändert.

Wichtig für die Sicherheit: Der Aufrufer schickt **nur eine Kennung aus dieser
Liste**, kein Stück Prompt. Die zugehörigen Anweisungen stehen in `worker.js`.
Ein unbekannter Wert wird mit 400 abgewiesen, statt stillschweigend auf die
Voreinstellung zurückzufallen — sonst könnte ein Tippfehler unbemerkt die falsche
Ebene deuten.

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

## Domänentabelle

Die Domänen stehen nur an einer Stelle: in `domaenen.json` im Wurzelverzeichnis
des Repos. Beide HTML-Seiten laden sie beim Start, der Worker holt sie von

```
https://abuelia81.github.io/kommunolabium/domaenen.json
```

und hält sie zehn Minuten lang zwischengespeichert. Eine Änderung an der Tabelle
braucht also **kein** erneutes Einspielen des Workers — pushen genügt, spätestens
zehn Minuten später deutet er gegen die neue Fassung. Fällt die Datei einmal aus,
arbeitet der Worker mit der zuletzt geladenen Fassung weiter; nur wenn er noch nie
eine hatte, meldet er einen Fehler.

Liegt das Repo einmal woanders, ist `DOMAENEN_URL` oben in `worker.js` die einzige
anzupassende Zeile.

Das Lexikon steht ebenso nur einmal, in `lexikon.json`. Der Worker braucht es nicht —
nachgeschlagen wird im Browser; er sieht ein Wort erst, wenn das Lexikon es nicht kennt.

## Instrumente

Drei Seiten benutzen den Vermittler und schicken ihre Kennung im Feld `instrument` mit:
`kommunikationslabium`, `astrolabium` und `sprachkugel`. Jede hat im Worker ihre eigene
Anrede; die Domänen teilen sie sich. Eine vierte Seite braucht nur einen weiteren
Eintrag in `INSTRUMENTE`.

# Plugin-System für XBVR – Analyse & Plan

Stand: 2026-10-01, Basis upstream `master` (0.4.40, `dc8c9f0`). Nur Analyse, noch kein Code.

## Ziel

Externe Web-Anwendungen („Plugins“) sollen in die XBVR-Oberfläche eingebunden werden können – als eigener
Menüpunkt und als Tab in der Szenen-Detailansicht –, ohne dass ihr Code in XBVR landet. Ein Plugin ist ein
eigenständiger HTTP-Dienst (eigener Prozess/Container), XBVR kennt nur seine URL.

## Empfohlener Ansatz

1. XBVR hält eine Liste registrierter Plugins (`id`, `name`, `url`, `enabled`).
2. Jedes Plugin liefert ein Manifest `GET <url>/xbvr-plugin.json`.
3. XBVR leitet `/plugins/<id>/…` per Reverse Proxy an das Plugin weiter (gleiche Origin wie XBVR).
4. Die UI zeigt das Plugin als iframe: Navbar-Eintrag + optionaler Tab im Szenen-Detail.
5. Eine `postMessage`-Brücke erlaubt dem Plugin, XBVR zu steuern (Szene öffnen, Szene neu laden, Toast).

Warum iframe statt echter UI-Plugins: Die UI ist Vue 2.7 + Buefy 0.9, mit vue-cli/webpack gebaut und per
`go:embed` fest eingebettet (`ui/fs.go`). Fremde Vue-Komponenten lassen sich zur Laufzeit nicht nachladen
(kein Module Federation o. Ä.). Ein iframe entkoppelt Technik, Version und Release-Zyklus komplett.

Warum Proxy statt iframe direkt auf den Plugin-Port:
- gleiche Origin → `postMessage`-Brücke und `frame-ancestors 'self'` funktionieren einfach,
- das Plugin muss nicht selbst im LAN offen sein (VR-Headset/andere Geräte erreichen nur XBVR),
- die Basic-Auth von XBVR schützt das Plugin mit.

XBVR soll Plugins **nicht** selbst starten (kein Prozess-Manager). „Mitstarten“ übernimmt docker compose /
der Dienst-Manager des Systems; XBVR zeigt nur „nicht erreichbar“, wenn ein Plugin fehlt.

## Manifest (Vorschlag)

```json
{
  "id": "example",
  "name": "Example",
  "version": "1.0.0",
  "nav": { "label": "Example", "path": "/" },
  "sceneTab": { "label": "Example", "path": "/xbvr/scene?scene_id={scene_id}" },
  "options": { "path": "/settings" },
  "healthPath": "/health"
}
```

Alle Pfade relativ zur Plugin-Basis; `{scene_id}` wird von XBVR ersetzt.

## Ist-Zustand in XBVR (relevante Stellen)

| Bereich | Stand | Datei |
|---|---|---|
| Routing | `go-restful` für `/api/*`, `gorilla/mux` für `/img`, `/imghm`, `/download`, `/myfiles`, Rest fällt auf `http.DefaultServeMux` | `pkg/server/server.go:161` |
| UI-Auslieferung | `/ui/` über `authHandle` (Basic Auth nur hier) | `pkg/server/server.go:41`, `:141` |
| CORS | `cors.Default()` um **alle** Routen | `pkg/server/server.go:164` |
| WebSocket | WAMP unter `/ws/`, Realm `default`, anonym, `AllowOrigins("*")` | `pkg/server/server.go:186` |
| Events | nur `lock.change`, `remote.state`, `state.change.optionsStorage`, `options.previews.previewReady` | `common.PublishWS` |
| Config | `ObjectConfig` als JSON im KV-Eintrag `config` | `pkg/config/config.go:23` |
| UI-Router | Hash-Modus, 4 feste Routen | `ui/src/router.js` |
| Navbar | statische Links | `ui/src/Navbar.vue` |
| Szenen-Detail | `b-tabs` Files / Cuepoints / Watch history / Description / Search fields | `ui/src/views/scenes/Details.vue:229–377` |
| Options | Menü + Sektionen | `ui/src/views/options/Options.vue` |

## Nötige Änderungen in XBVR

### Backend (Go, ca. 250–350 Zeilen)

1. **Config** (`pkg/config/config.go`): Block `Plugins []struct{ ID, Name, URL string; Enabled bool }`.
   Zusätzlich Env-Variable `XBVR_PLUGINS` (z. B. `id=url,id2=url2`) für Container-Setups, damit nichts per UI
   eingetragen werden muss.
2. **Registry** (`pkg/plugins/`, neu): Manifest abrufen und cachen, Health-Check, Status (erreichbar/Fehler).
3. **API** (`pkg/api/plugins.go`, neu): `GET /api/plugins` (Liste + Manifest + Status),
   `PUT /api/options/plugins` (speichern). In `server.go` per `restful.Add` registrieren.
4. **Proxy** (`pkg/server/server.go`): `/plugins/{id}/` über `httputil.ReverseProxy`:
   - Prefix abschneiden, `X-Forwarded-Prefix: /plugins/<id>` und `X-Forwarded-*` setzen,
   - `req.Host` auf das Ziel setzen (Plugins mit Host-Header-Prüfung lehnen sonst ab),
   - `FlushInterval: -1` (SSE/Streaming),
   - Basic Auth wie bei `/ui/` (`authHandle`),
   - **außerhalb** des `cors.Default()`-Wrappers einhängen (siehe Schwachstellen).
5. **Optional – Events:** neues WAMP-Topic `scene.changed {scene_id}` in `editScene`, `toggleList`,
   `rateScene`, `matchFile`, `unmatchFile`, `createCustomScene`, `deleteScene`. Wenige Zeilen, erspart Plugins
   das Pollen von `/api/scene/list`.

### UI (Vue, ca. 250 Zeilen)

1. `store/plugins.js` – lädt `/api/plugins`.
2. `views/plugins/PluginFrame.vue` – iframe über volle Höhe unter der Navbar; Route `/plugins/:id/:path*`.
3. `Navbar.vue` – Plugin-Einträge per `v-for`.
4. `Details.vue` – Plugin-Tab(s) **am Ende** der `b-tabs`, iframe erst bei Tab-Aktivierung laden.
5. `Options.vue` + `sections/Plugins.vue` – hinzufügen, aktivieren, Status anzeigen.
6. **postMessage-Brücke** (nur Nachrichten der eigenen Origin akzeptieren):
   `openScene(scene_id)`, `sceneUpdated(scene_id)`, `navigate(path)`, `toast(msg, type)`.
7. Optional: Badge/Button auf `SceneCard.vue` je Plugin – nur über Batch-Abfrage, nie pro Karte einzeln.

## Anforderungen an ein Plugin

- `GET /xbvr-plugin.json` liefern.
- iframe erlauben: `Content-Security-Policy: frame-ancestors 'self'` (hinter dem Proxy ist XBVR „self“);
  kein `X-Frame-Options: DENY`.
- **Unter einem Unterpfad lauffähig sein:** keine absoluten Pfade (`/api/...`, `/static/...`) in HTML/JS.
  Empfehlung: `<base href="${X-Forwarded-Prefix}/">` in jede Seite einsetzen und alle URLs relativ schreiben.
  Direkter Aufruf ohne Proxy funktioniert dann weiterhin.
- Für den Szenen-Tab eine Seite, die `scene_id` (String-Key wie `slr-12345`) entgegennimmt.

## Schwachstellen / Risiken (Flaws)

1. **CORS öffnet Plugins für CSRF.** `cors.Default()` erlaubt jeder Origin GET/POST/HEAD inklusive
   `Content-Type: application/json`. Plugins, die sich darauf verlassen, dass ein JSON-POST einen
   (nie erlaubten) Preflight auslöst, wären hinter dem Proxy von jeder fremden Webseite aus beschreibbar.
   → Proxy-Route unbedingt ohne CORS-Wrapper ausliefern; zusätzlich `Origin`-Prüfung im Proxy.
2. **Die XBVR-API selbst ist unauthentifiziert** (Basic Auth nur auf `/ui/`), und durch `cors.Default()`
   ebenfalls von fremden Seiten aus ansprechbar. Ein Plugin im iframe (gleiche Origin) hat vollen API-Zugriff.
   Das ist bei selbst betriebenen Plugins gewollt, macht aber jedes Plugin zu einer vertrauenswürdigen
   Komponente → nur eigene/geprüfte Plugins registrieren; iframe ggf. mit `sandbox`-Attribut.
3. **WebSocket mit `AllowOrigins("*")`** und anonymem Realm: jede Seite kann Logs/Events mitlesen.
   Unverändert, aber relevant, falls Plugins Events darüber bekommen.
4. **Fest verdrahtete Tab-Indizes in `Details.vue`:** an mehreren Stellen wird `activeTab == 1` (Cuepoints)
   bzw. `!= 1` geprüft (z. B. Zeile 147, 163, 180). Neue Tabs vor Cuepoints verschieben die Indizes →
   Plugin-Tabs nur ans Ende hängen oder die Prüfungen auf Namen umstellen.
5. **Keine Szenen-Events:** Änderungen an Szenen (Edit, Listen, Match) erzeugen kein WAMP-Event;
   Plugins müssen ohne Punkt 5 der Backend-Änderungen pollen.
6. **Unterpfad-Problem liegt beim Plugin:** Plugins mit absoluten Pfaden funktionieren hinter dem Proxy
   nicht. HTML-Umschreiben im Proxy wäre zu fragil und ist nicht vorgesehen.
7. **Host-Header-Prüfung** in Plugins: ohne `req.Host`-Umschreibung im Proxy werden Anfragen abgelehnt.
8. **Fork-Pflege:** Die Änderungen treffen viel geänderte upstream-Dateien (`server.go`, `Details.vue`,
   `config.go`, `Navbar.vue`). Logik deshalb in neue Dateien legen, Eingriffe in bestehende minimal halten;
   ein generisches „External Apps“-Feature ggf. als PR upstream anbieten.
9. **Build:** Eigener Build nötig (Go 1.25 + cgo für sqlite, yarn-UI-Build; unter Windows mingw). Setups,
   die heute Release-Binaries ziehen, müssen auf den Fork-Build umgestellt werden. Ein Update von älteren
   Versionen bringt die upstream-Migrationen mit.

## Umsetzung in Stufen

1. **Stufe 1:** Config + `XBVR_PLUGINS`, Registry, `GET /api/plugins`, Proxy (ohne CORS, mit Auth),
   Navbar-Eintrag, `PluginFrame.vue`. Ergebnis: Plugin voll in XBVR bedienbar.
2. **Stufe 2:** Szenen-Tab + postMessage-Brücke (Szene öffnen / neu laden).
3. **Stufe 3:** WAMP-Event `scene.changed`, Options-Sektion „Plugins“ mit Status, optional Karten-Badges.

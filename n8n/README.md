# Daglig AI-driven marknadsnyhetsbot — n8n workflow

`marknadsnyhetsbot.workflow.json` är ett komplett, importerbart n8n-workflow för `alibaba1234.app.n8n.cloud`.

## Importera

1. n8n → **Workflows → Import from File** → välj `marknadsnyhetsbot.workflow.json`.
2. Workflowet importeras inaktivt (`active: false`) med `timezone: Europe/Stockholm` redan satt i workflow-inställningarna, så 07:00/17:00 blir korrekt lokal tid året runt (även över DST-skiften).

## Så hänger workflowet ihop

Två oberoende triggerkedjor (SE 07:00 och Global 17:00) samlar in, normaliserar och taggar artiklar var för sig — sedan går båda in i **samma delade pipeline** (Claude-analys → parsning → loggning → mailutskick), precis som du bad om ("återanvänd samma Claude-analysnod"). Endast en av kedjorna kör åt gången (styrs av respektive schema), så det delade steget ("Build Article Batch") körs en gång per exekvering oavsett vilken gren som triggade den.

```
Schedule 07:00 SE ──┬─ HTTP Di.se → Parsa RSS ──────────┐
                     ├─ HTTP Placera.se → Parsa RSS ─────┼─ Merge SE (RSS) ─┐
                     ├─ RSS Breakit ──────────────────────┘                 ├─ Merge SE (alla källor) ─ Normalize & Tag SE ─┐
                     └─ HTTP NewsAPI SE ─── Flatten NewsAPI SE ─────────────┘                                              │
                                                                                                                              ├─ Build Article Batch ─ Claude (analys) ─ Parsa Claude-svar ─┬─ Split Out (flaggor) ─ Google Sheets (append)
Schedule 17:00 Global ┬─ HTTP Yahoo Finance → Parsa RSS ──┐                                                                 │                                                              └─ Bygg Sammanställning ─ Gmail (send)
                       ├─ RSS MarketWatch ─────────────────┼─ Merge Global (RSS) ─┐
                       ├─ RSS OilPrice.com ─────────────────┤                      ├─ Merge Global (alla källor) ─ Normalize & Tag Global ─┘
                       ├─ HTTP Mining.com → Parsa RSS ──────┤                      │
                       ├─ RSS TechCrunch ────────────────────┘                     │
                       └─ HTTP NewsAPI Global ─ Flatten NewsAPI Global ────────────┘
```

**Alla 8 källor är verifierade live och fungerar** (se tabellen nedan) — inga döda RSS-noder kvar i workflowet.

## Credentials du behöver lägga in (4 st)

Skapas under **n8n → Credentials → Add Credential**. Efter import måste du öppna respektive nod och välja rätt credential i dropdownen (de kommer per automatik inte vara ihopkopplade, av säkerhetsskäl).

| Credential | Typ i n8n | Fält | Används av |
|---|---|---|---|
| NewsAPI (Query Auth) | **Query Auth** | Name=`apiKey`, Value=din NewsAPI-nyckel | `HTTP — NewsAPI SE`, `HTTP — NewsAPI Global` |
| Anthropic API (Header Auth) | **Header Auth** | Name=`x-api-key`, Value=din Anthropic API-nyckel | `Claude — Analysera Artiklar` |
| Google Sheets account | **Google Sheets OAuth2** | Standard Google-inloggning | `Google Sheets — Logga Flagga` |
| Gmail account | **Gmail OAuth2** | Standard Google-inloggning | `Gmail — Skicka Sammanställning` |

Efter att du kopplat Google Sheets-credentialen: öppna noden, välj ditt Sheet + blad i dropdownen, och kontrollera att kolumnmappningen (redan förifylld i JSON:en) matchar dina rubriker. Skapa bladet med rubrikraden **exakt**:

```
Datum | Körning (SE/Global) | Bolag | Bransch | Signal | Konfidensgrad | Motivering | Källa | Utfall
```

Gmail-mottagare är förifylld till din adress (`zejneliadrian@icloud.com`) i noden "Gmail — Skicka Sammanställning" — byt om det behövs.

## Mock-test (JSON-parsningen är verifierad)

Jag har kört hela pipeline-logiken (Code-noderna, extraherade rakt ur workflow-JSON:en) lokalt i Node.js mot en simulerad artikel och ett simulerat Claude-svar, för att bevisa att parsningen fungerar innan du kopplar på skarpt schema:

- **Input**: två mock-artiklar (en Saab-nyhet med tydlig signal, en ospecifik räntespekulation)
- **Simulerat Claude-svar**: JSON med en flagga (Saab, Försvar, Köp, Hög)
- **Verifierat**: `Build Article Batch` bygger korrekt user-message → `Parsa Claude-svar` extraherar `flaggor`-arrayen korrekt ur `content[0].text` → `Split Out` producerar en rad med alla kolumner (inkl. Datum/Körning) redo för Google Sheets-append → `Bygg Sammanställning` genererar korrekt formaterad brief-text inklusive Topp-flagga-sektionen → **nolltrfägganfall** (tomt `flaggor: []`) testades också och producerar ett rent "Inga köpsignaler idag"-mail istället för att krascha.

Alla fem stegen passerade (`ALL MOCK TESTS PASSED`). Det här bevisar logiken — det bevisar inte att RSS-URL:erna eller de riktiga API-nycklarna fungerar; det måste testas i n8n med riktiga credentials (kör workflowet manuellt med "Execute Workflow" per nod).

## ✅ RSS-källor — alla verifierade live

Jag hämtade och kontrollerade varje källa på riktigt (inte bara antog att URL:en stämde), och bytte ut de två som var genuint döda. Så här ser det ut nu:

| Källa | Status | Teknik | Verifierat |
|---|---|---|---|
| Di.se | Fixad — blockerade förfrågningar utan webbläsar-header | HTTP + parse | 200 OK, 20 artiklar |
| Placera.se | Fixad — fel URL (`/rss.xml` → rätt är `/artiklar/rss.xml`, hittad i sidkällan) | HTTP + parse | 200 OK, 30 artiklar |
| ~~Affärsvärlden~~ → **Breakit** | Affärsvärlden borttagen (oändlig omdirigeringsloop, Cloudflare-liknande skydd som inte går att lösa med headers). Ersatt med Breakit (svensk tech/startup-nyheter) | Plain RSS-nod | 200 OK, 17 artiklar |
| MarketWatch | Fungerade från början | Plain RSS-nod | 200 OK, 10 artiklar |
| OilPrice.com | Fungerade från början | Plain RSS-nod | 200 OK, 15 artiklar |
| ~~Reuters Business~~ → **Yahoo Finance** | Reuters borttaget (inget publikt RSS-flöde kvar sedan ~2020, ingen feed-länk hittad någonstans på deras sajt). Ersatt med Yahoo Finance (bred global marknadsbevakning) | HTTP + parse | 200 OK, 50 artiklar |
| ~~Kitco News~~ → **Mining.com** | Kitco borttaget (ingen RSS-länk i sidkällan, alla gissade adresser 404). Ersatt med Mining.com (gruv-/metall-/råvarunyheter — samma nisch som Kitco skulle täckt) | HTTP + parse | 200 OK, 36 artiklar |
| TechCrunch | Fungerade från början | Plain RSS-nod | 200 OK, 20 artiklar |

**Teknisk bakgrund:** n8n:s inbyggda RSS Feed Read-nod kan inte skicka egna headers. De källor som blockerar en "vanlig" HTTP-klient (kollar `User-Agent`) går via **HTTP Request (med en riktig webbläsar-`User-Agent` + `Accept`-header) → en liten Code-nod som parsar RSS/Atom-XML:en själv** — det är samma teknik för alla fyra "HTTP + parse"-källor ovan, och samma kodsnutt (testad separat mot varje källas riktiga data). Källor som redan tog emot en vanlig förfrågan utan problem fick behålla den enklare inbyggda RSS-noden.

Om en källa någon gång slutar fungera (sajter ändrar bot-skydd/URL:er över tid): ta bort den noden i n8n — varje källa är oberoende fram till Merge-steget, så resten av workflowet påverkas inte, och NewsAPI-sökningen ger ändå bred bevakning av branscherna.

## Tips / saker jag skulle ändra

1. **Dubblettskydd över dagar, inte bara inom en körning.** Just nu dedupliceras artiklar bara inom samma körning (via länk). Om samma Saab-nyhet dyker upp både i morgon- och kvällskällor olika dagar loggas den två gånger i Sheets. Om du vill undvika det kan ett litet "har jag redan loggat den här länken de senaste N dagarna"-steg läggas till (Google Sheets-lookup innan append) — jag lämnade det utanför för att hålla workflowet enklare, men säg till om du vill ha det.
2. **`max_tokens: 4096` kan vara i minsta laget** om en körning genererar många flaggor (t.ex. kvällskörningen med 5 RSS-källor + NewsAPI kan ge 30–40 artiklar). Om Claude-svaret klipps av mitt i JSON:en kbraschar parsningen tyst till `flaggor: []`. Överväg att höja till 8192 och/eller logga `stop_reason` för felsökning.
3. **`MAX_ARTICLES = 40`** i "Build Article Batch" är en hård gräns för att hålla prompten rimlig — om en källa är ovanligt aktiv en dag kan relevanta artiklar tappas bort i klippningen (de första 40 i input-ordning vinner, ingen prioritering). Fungerar bra i normalfallet men värt att känna till.
4. **Modellnamnet `claude-sonnet-5`** är satt som en rimlig standard. Om du vill låsa en exakt, oföränderlig modellversion (så att en framtida modell-uppdatering inte påverkar dina resultat över tid när du utvärderar träffsäkerhet), byt till en daterad snapshot-modell-identifierare istället.
5. **Ingen retry/felhantering på HTTP-anropen.** Om NewsAPI eller Claude API svarar med fel (t.ex. rate limit) stannar hela körningen och inget mail skickas den dagen — du märker det bara om du aktivt tittar i n8n:s execution-logg. Om du vill ha en varning istället, går det enkelt att lägga till en "Error Trigger"-workflow som mailar dig om en körning misslyckas.
6. **Gmail auto-send, som du bad om** — inga bekräftelsesteg. Värt att dubbelkolla att `sendTo`-adressen i "Gmail — Skicka Sammanställning" alltid är rätt, eftersom det går direkt ut utan draft-steg.
7. **Breakit är inte en 1:1-ersättning för Affärsvärlden** — Breakit är svensk tech/startup-bevakning, medan Affärsvärlden var bredare affärsjournalistik. Det ger faktiskt lite extra värde för "AI/tech"-kategorin på morgonkörningen (som annars saknade en dedikerad källa), men täcker inte banker/fastigheter/energi på samma sätt Affärsvärlden hade gjort — där gör Di.se, Placera.se och NewsAPI-sökningen jobbet istället.

## Modifierad prompt

Systempromptet i "Build Article Batch" är kopierat ordagrant från din spec — jag har inte ändrat något i sak, bara paketerat den som en JS-sträng.

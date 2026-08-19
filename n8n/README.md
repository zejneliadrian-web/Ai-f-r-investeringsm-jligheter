# Daglig AI-driven marknadsnyhetsbot — n8n workflow

`marknadsnyhetsbot.workflow.json` är ett komplett, importerbart n8n-workflow för `alibaba1234.app.n8n.cloud`.

## Importera

1. n8n → **Workflows → Import from File** → välj `marknadsnyhetsbot.workflow.json`.
2. Workflowet importeras inaktivt (`active: false`) med `timezone: Europe/Stockholm` redan satt i workflow-inställningarna, så 07:00/17:00 blir korrekt lokal tid året runt (även över DST-skiften).

## Så hänger workflowet ihop

Två oberoende triggerkedjor (SE 07:00 och Global 17:00) samlar in, normaliserar och taggar artiklar var för sig — sedan går båda in i **samma delade pipeline** (Claude-analys → parsning → loggning → mailutskick), precis som du bad om ("återanvänd samma Claude-analysnod"). Endast en av kedjorna kör åt gången (styrs av respektive schema), så det delade steget ("Build Article Batch") körs en gång per exekvering oavsett vilken gren som triggade den.

```
Schedule 07:00 SE ────── [8 SE-källor, se tabell] ─── Merge SE (RSS) ─┐
                     └── HTTP NewsAPI SE → Flatten ────────────────────┼─ Merge SE (alla källor) ─ Normalize & Tag SE ─┐
                                                                                                                          │
                                                                                                                          ├─ Build Article Batch ─ Claude (analys) ─ Parsa Claude-svar ─┬─ Split Out (flaggor) ─ Google Sheets (append)
Schedule 17:00 Global ── [5 Global-källor, se tabell] ─ Merge Global (RSS) ─┐                                           │                                                              └─ Bygg Sammanställning ─ Gmail (send)
                     └── HTTP NewsAPI Global → Flatten ─────────────────────┼─ Merge Global (alla källor) ─ Normalize & Tag Global ─┘
                                                                             ┘
```

**Alla 13 källor (8 SE + 5 Global) är verifierade live och fungerar** — inga döda RSS-noder kvar i workflowet. Källorna för respektive marknad listas i tabellerna nedan.

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

Jag hämtade och kontrollerade varje källa på riktigt (inte bara antog att URL:en stämde). Två döda källor byttes ut, och SE-listan utökades på begäran från 3 till 8 källor för bredare, mer pålitlig bevakning av den svenska marknaden.

**Morgonbrief SE (07:00) — 8 källor:**

| Källa | Status | Teknik | Verifierat |
|---|---|---|---|
| Di.se | Fixad — blockerade förfrågningar utan webbläsar-header | HTTP + parse | 200 OK, 20 artiklar |
| Placera.se | Fixad — fel URL (`/rss.xml` → rätt är `/artiklar/rss.xml`, hittad i sidkällan) | HTTP + parse | 200 OK, 30 artiklar |
| ~~Affärsvärlden~~ → **Breakit** | Affärsvärlden borttagen (oändlig omdirigeringsloop, Cloudflare-liknande skydd). Ersatt med Breakit (svensk tech/startup) | Plain RSS-nod | 200 OK, 17 artiklar |
| **DN Ekonomi** (ny) | Dagens Nyheters ekonomisektion — brett och stort flöde | Plain RSS-nod | 200 OK, 103 artiklar |
| **SVT Ekonomi** (ny) | Public service, mycket pålitlig | Plain RSS-nod | 200 OK, 100 artiklar |
| **Sveriges Radio Ekot** (ny) | Public service-nyhetsradions ekonomibevakning | Plain RSS-nod | 200 OK, 20 artiklar |
| **Second Opinion** (ny) | Nischad energisektor-bevakning — matchar "Energi"-kategorin extra bra | Plain RSS-nod | 200 OK, 4 artiklar |
| **Dagens PS** (ny) | Svenska närings-/entreprenörsnyheter | Plain RSS-nod | 200 OK, 10 artiklar |

**Kandidater jag testade och förkastade** (så att du slipper testa dem igen): MFN.se (bolagspressmeddelanden — sidan är en JavaScript-app, ingen riktig feed bakom URL:en trots namnet), Fastighetsvärlden (RSS-länken kräver inloggning via SSO), Realtid.se (403 Forbidden på alla varianter), Aktiespararna (ingen fungerande RSS hittad).

**Kvällsbrief Global (17:00) — 5 källor:**

| Källa | Status | Teknik | Verifierat |
|---|---|---|---|
| MarketWatch | Fungerade från början | Plain RSS-nod | 200 OK, 10 artiklar |
| OilPrice.com | Fungerade från början | Plain RSS-nod | 200 OK, 15 artiklar |
| TechCrunch | Fungerade från början | Plain RSS-nod | 200 OK, 20 artiklar |
| ~~Reuters Business~~ → **Yahoo Finance** | Reuters borttaget (inget publikt RSS-flöde kvar sedan ~2020). Ersatt med Yahoo Finance (bred global marknadsbevakning) | HTTP + parse | 200 OK, 50 artiklar |
| ~~Kitco News~~ → **Mining.com** | Kitco borttaget (ingen RSS-länk i sidkällan). Ersatt med Mining.com (gruv-/metall-/råvarunyheter — samma nisch) | HTTP + parse | 200 OK, 36 artiklar |

**Teknisk bakgrund:** n8n:s inbyggda RSS Feed Read-nod kan inte skicka egna headers. De källor som blockerar en "vanlig" HTTP-klient (kollar `User-Agent`) går via **HTTP Request (med en riktig webbläsar-`User-Agent` + `Accept`-header) → en liten Code-nod som parsar RSS/Atom-XML:en själv** — samma teknik och samma kodsnutt för alla "HTTP + parse"-källor, testad separat mot varje källas riktiga data. Källor som redan tog emot en vanlig förfrågan utan problem fick behålla den enklare inbyggda RSS-noden.

Om en källa någon gång slutar fungera (sajter ändrar bot-skydd/URL:er över tid): ta bort den noden i n8n — varje källa är oberoende fram till Merge-steget, så resten av workflowet påverkas inte, och NewsAPI-sökningen ger ändå bred bevakning av branscherna.

## Tips / saker jag skulle ändra

1. **Dubblettskydd över dagar, inte bara inom en körning.** Just nu dedupliceras artiklar bara inom samma körning (via länk). Om samma Saab-nyhet dyker upp både i morgon- och kvällskällor olika dagar loggas den två gånger i Sheets. Om du vill undvika det kan ett litet "har jag redan loggat den här länken de senaste N dagarna"-steg läggas till (Google Sheets-lookup innan append) — jag lämnade det utanför för att hålla workflowet enklare, men säg till om du vill ha det.
2. **`max_tokens: 4096` kan vara i minsta laget** om en körning genererar många flaggor. Om Claude-svaret klipps av mitt i JSON:en kraschar parsningen tyst till `flaggor: []`. Överväg att höja till 8192 och/eller logga `stop_reason` för felsökning.
3. **Urval per källa, inte bara en rak gräns.** Morgonkörningen har nu 8 SE-källor med väldigt olika volym (DN Ekonomi ~103 artiklar/dag, Second Opinion ~4). Ett naivt "ta de första 40" hade i praktiken helt uteslutit de mindre källorna eftersom de stora fyller upp gränsen innan de ens nås. "Build Article Batch" gör därför **rättvist urval**: max 8 artiklar per källa, totalt tak 60 — så alla 8-9 källor (inkl. NewsAPI) alltid är representerade i det som skickas till Claude, oavsett hur många artiklar en enskild källa råkar ha den dagen. Testat med riktiga volymer (324 artiklar från 9 källor → alla källor representerade, ingen utesluten).
4. **Modellnamnet `claude-sonnet-5`** är satt som en rimlig standard. Om du vill låsa en exakt, oföränderlig modellversion (så att en framtida modell-uppdatering inte påverkar dina resultat över tid när du utvärderar träffsäkerhet), byt till en daterad snapshot-modell-identifierare istället.
5. **Ingen retry/felhantering på HTTP-anropen.** Om NewsAPI eller Claude API svarar med fel (t.ex. rate limit) stannar hela körningen och inget mail skickas den dagen — du märker det bara om du aktivt tittar i n8n:s execution-logg. Om du vill ha en varning istället, går det enkelt att lägga till en "Error Trigger"-workflow som mailar dig om en körning misslyckas.
6. **Gmail auto-send, som du bad om** — inga bekräftelsesteg. Värt att dubbelkolla att `sendTo`-adressen i "Gmail — Skicka Sammanställning" alltid är rätt, eftersom det går direkt ut utan draft-steg.
7. **Breakit är inte en 1:1-ersättning för Affärsvärlden** — Breakit är svensk tech/startup-bevakning, medan Affärsvärlden var bredare affärsjournalistik. Det ger faktiskt lite extra värde för "AI/tech"-kategorin på morgonkörningen (som annars saknade en dedikerad källa), men täcker inte banker/fastigheter/energi på samma sätt Affärsvärlden hade gjort — där gör Di.se, Placera.se och NewsAPI-sökningen jobbet istället.

## Modifierad prompt

Systempromptet i "Build Article Batch" är kopierat ordagrant från din spec — jag har inte ändrat något i sak, bara paketerat den som en JS-sträng.

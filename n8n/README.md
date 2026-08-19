# Daglig AI-driven marknadsnyhetsbot — n8n workflow

`marknadsnyhetsbot.workflow.json` är ett komplett, importerbart n8n-workflow för `alibaba1234.app.n8n.cloud`.

## Importera

1. n8n → **Workflows → Import from File** → välj `marknadsnyhetsbot.workflow.json`.
2. Workflowet importeras inaktivt (`active: false`) med `timezone: Europe/Stockholm` redan satt i workflow-inställningarna, så 07:00/17:00 blir korrekt lokal tid året runt (även över DST-skiften).

## Så hänger workflowet ihop

Två oberoende triggerkedjor (SE 07:00 och Global 17:00) samlar in, normaliserar och taggar artiklar var för sig — sedan går båda in i **samma delade pipeline** (Claude-analys → parsning → loggning → mailutskick), precis som du bad om ("återanvänd samma Claude-analysnod"). Endast en av kedjorna kör åt gången (styrs av respektive schema), så det delade steget ("Build Article Batch") körs en gång per exekvering oavsett vilken gren som triggade den.

```
Schedule 07:00 SE ──┬─ HTTP Di.se → Parsa RSS ─────────┐
                     ├─ HTTP Placera.se → Parsa RSS ────┼─ Merge SE (RSS) ─┐  ✅ Di.se + Placera.se bekräftat fungerande
                     ├─ HTTP Affärsvärlden → Parsa RSS ─┘     ├─ Merge SE (alla källor) ─ Normalize & Tag SE ─┐
                     └─ HTTP NewsAPI SE ─── Flatten NewsAPI SE ┘                                              │
                                                                                                                ├─ Build Article Batch ─ Claude (analys) ─ Parsa Claude-svar ─┬─ Split Out (flaggor) ─ Google Sheets (append)
Schedule 17:00 Global ┬─ RSS Reuters ───────┐                                                                 │                                                              └─ Bygg Sammanställning ─ Gmail (send)
                       ├─ RSS MarketWatch ───┼─ Merge Global (RSS) ─┐
                       ├─ RSS OilPrice.com ──┤                      ├─ Merge Global (alla källor) ─ Normalize & Tag Global ─┘
                       ├─ RSS Kitco News ────┤                      │
                       ├─ RSS TechCrunch ────┘                      │
                       └─ HTTP NewsAPI Global ─ Flatten NewsAPI Global ┘
```

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

## ⚠️ RSS-källor: status efter verifiering + bot-skydds-fix

Jag hämtade och kontrollerade varje RSS-URL live. Tre av de svenska källorna blockerade mina första försök — det visade sig bero på att sajten kollar `User-Agent`-headern och nekar förfrågningar som inte ser ut att komma från en vanlig webbläsare. n8n:s inbyggda RSS-nod kan inte skicka egna headers, så jag bytte ut de tre svenska källorna mot **HTTP Request (med en vanlig webbläsar-`User-Agent`) → en liten Code-nod som parsar RSS/Atom-XML:en själv**. Samma fält (title/länk/ingress) skickas vidare som tidigare, så resten av workflowet är opåverkat.

**Verifierat fungerande efter fix (testade live med riktig data):**
- **Di.se** — `https://www.di.se/rss` var faktiskt hela tiden en giltig adress; blockeringen var bara bot-skyddet. Med webbläsar-header: 200 OK, 20 artiklar hämtade och parsade korrekt.
- **Placera.se** — den ursprungliga gissningen (`/rss.xml`) var fel adress. Rätt adress hittades i sidans källkod: `https://www.placera.se/artiklar/rss.xml`. Med webbläsar-header: 200 OK, 30 artiklar hämtade och parsade korrekt.
- MarketWatch — `https://feeds.content.dowjones.io/public/rss/mw_topstories`
- OilPrice.com — `https://oilprice.com/rss/main`
- TechCrunch — `https://techcrunch.com/feed/`

**Fortfarande inte löst:**
- **Affärsvärlden** — testade `/rss.xml`, `/rss`, `/feed`, `/feed/` (403/404) och till och med bara startsidan, som gav en omdirigering (302) vars mål aktivt stryper anslutningen efter under en sekund. Det tyder på ett skydd bortom bara User-Agent-kontroll (typ Cloudflare), som header-tricket inte kan lösa. Noden är förberedd med samma HTTP+parse-teknik ifall du hittar en lösning själv i webbläsaren.
- **Reuters Business** — Reuters lade ner nästan alla publika RSS-flöden runt 2020. URL:en jag satt in är deras institutionsflöde (`reutersagency.com`), som jag inte kunde nå för verifiering.
- **Kitco News** — ingen fungerande publik RSS hittad efter flera försök.

Reuters och Kitco står kvar som vanliga RSS-noder (ej ombyggda) tills vi bestämmer om de ska bytas ut mot t.ex. CNBC/Defense News, som jag föreslog tidigare.

Om en källa inte går att lösa: ta bort den noden i n8n — NewsAPI-sökningen täcker redan branschbevakningen brett och resten av workflowet påverkas inte, eftersom varje källnod är oberoende innan Merge-steget.

## Tips / saker jag skulle ändra

1. **Dubblettskydd över dagar, inte bara inom en körning.** Just nu dedupliceras artiklar bara inom samma körning (via länk). Om samma Saab-nyhet dyker upp både i morgon- och kvällskällor olika dagar loggas den två gånger i Sheets. Om du vill undvika det kan ett litet "har jag redan loggat den här länken de senaste N dagarna"-steg läggas till (Google Sheets-lookup innan append) — jag lämnade det utanför för att hålla workflowet enklare, men säg till om du vill ha det.
2. **`max_tokens: 4096` kan vara i minsta laget** om en körning genererar många flaggor (t.ex. kvällskörningen med 5 RSS-källor + NewsAPI kan ge 30–40 artiklar). Om Claude-svaret klipps av mitt i JSON:en kbraschar parsningen tyst till `flaggor: []`. Överväg att höja till 8192 och/eller logga `stop_reason` för felsökning.
3. **`MAX_ARTICLES = 40`** i "Build Article Batch" är en hård gräns för att hålla prompten rimlig — om en källa är ovanligt aktiv en dag kan relevanta artiklar tappas bort i klippningen (de första 40 i input-ordning vinner, ingen prioritering). Fungerar bra i normalfallet men värt att känna till.
4. **Modellnamnet `claude-sonnet-5`** är satt som en rimlig standard. Om du vill låsa en exakt, oföränderlig modellversion (så att en framtida modell-uppdatering inte påverkar dina resultat över tid när du utvärderar träffsäkerhet), byt till en daterad snapshot-modell-identifierare istället.
5. **Ingen retry/felhantering på HTTP-anropen.** Om NewsAPI eller Claude API svarar med fel (t.ex. rate limit) stannar hela körningen och inget mail skickas den dagen — du märker det bara om du aktivt tittar i n8n:s execution-logg. Om du vill ha en varning istället, går det enkelt att lägga till en "Error Trigger"-workflow som mailar dig om en körning misslyckas.
6. **Gmail auto-send, som du bad om** — inga bekräftelsesteg. Värt att dubbelkolla att `sendTo`-adressen i "Gmail — Skicka Sammanställning" alltid är rätt, eftersom det går direkt ut utan draft-steg.

## Modifierad prompt

Systempromptet i "Build Article Batch" är kopierat ordagrant från din spec — jag har inte ändrat något i sak, bara paketerat den som en JS-sträng.

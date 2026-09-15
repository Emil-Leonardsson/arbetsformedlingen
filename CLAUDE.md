# Arbetsförmedlingen – aktivitetsrapportering

Detta är Emils underlag för Arbetsförmedlingens aktivitetsrapport (sökta jobb, jobbintervjuer och kurser/utbildningar).

## Den officiella källan: Supabase (riktig Postgres-databas)

Sedan 2026-09-15 ligger all data i en riktig Supabase-databas, inte längre i Claude Artifacts inbyggda databas.

- **Projekt**: `kpaolyovfybxedgaigvp` (org "Emil-Leonardsson's Project", eu-west-1)
- **Schema**: `arbetsformedlingen`, tabeller `entries` och `meta_period`
- **Migrations/schema-historik**: [Emil-Leonardsson/shared-db](https://github.com/Emil-Leonardsson/shared-db) — databasen delas med andra små personliga appar (t.ex. gym-tracker), varje app har sitt eget Postgres-schema
- **Åtkomst**: den officiella Supabase MCP-connectorn (kopplad till Emils claude.ai-konto). Från Claude Code: verktygen `mcp__<uuid>__execute_sql` / `apply_migration` / `list_tables` osv (hitta uuid via `ToolSearch` om de inte redan är laddade, eller fråga Emil att aktivera connectorn i chatten om den inte syns).
- **Aktivitetsloggen** (sidan): https://claude.ai/code/artifact/60593adf-e013-4b1d-81a6-99ecf164e402 — samma URL som tidigare, men sidans kod pratar nu med Supabase via `claude.use("mcp")` istället för `claude.use("db")`. Källkoden ligger i den här mappen: `aktivitetslogg.html`.

`Arbetsformedlingen_aktivitetsrapport.md` i den här mappen är en ännu äldre historisk snapshot (markdown-eran, före både Artifact-databasen och Supabase), rör den inte.

### Varför bytet skedde

Claude Artifacts databas fick en bugg där `write_db`/uppdateringar av befintliga poster krävde ett `if_version`-fält som varken agent-verktyget eller (i praktiken) var enkelt att köra runt. Snarare än att fortsätta patcha kring det flyttades datan till en riktig Postgres-databas (Supabase), som redan fanns tillgänglig via Bolt/team-health-check-kontot.

### Viktig teknisk detalj: hur sidan pratar med Supabase

Artifact-sidor kan **inte** göra vanliga `fetch()`-anrop till Supabase (CSP blockerar utgående nätverksanrop till icke-godkända domäner). Istället används **`mcp`-capabilityn**, som låter sidan anropa viewarens egna anslutna claude.ai-connectors:

```js
capabilities: { mcp: { servers: [{ server: "Supabase", tools: ["execute_sql"] }] } }
```
```js
const mcp = await claude.use("mcp");
const res = await mcp.callTool("Supabase", "execute_sql", { project_id: "kpaolyovfybxedgaigvp", query: "..." });
```

`execute_sql`-verktyget svarar med en text inlindad i en `<untrusted-data-XXXX>...</untrusted-data-XXXX>`-boundary (samma format oavsett om anropet görs från en agent-session eller från sidans egen `mcp`-capability, bekräftat genom att observera ett riktigt anrop). **Obs:** samma tagg-sträng nämns även i prosan precis före den riktiga öppnande taggen ("...boundaries.\n\n<untrusted-data-XXXX>\n[...]"), så en naiv regex som matchar första `<untrusted-data-...>` fångar för mycket. `aktivitetslogg.html` löser det genom att kräva att den infångade texten börjar med `[` eller `{` direkt efter taggen. Om Supabase-verktygets svarsformat någonsin ändras, uppdatera `runSql()`-funktionen i `aktivitetslogg.html` och verifiera **live i webbläsaren** (inte bara via ett agent-tool-anrop, formatet kan skilja sig) innan du litar på tolkningen.

Eftersom det inte finns någon `watchTool`-baserad live-uppdatering (execute_sql är inte deklarerad read-only, så `watchTool` skulle avvisa den), laddas listan om explicit efter varje skrivning (`loadEntries()`), det är alltså inte realtid mellan flera samtidigt öppna flikar, bara efter en egen ändring.

## Uppdatera löpande

Så fort Emil klistrar in eller berättar om ett jobb han sökt, eller en kurs/utbildning, i vilken session som helst: skriv en post till `arbetsformedlingen.entries` direkt via Supabase-connectorns `execute_sql`/`apply_migration` – vänta inte tills han ber om en sammanfattning. Om något fält saknas för att posten ska vara komplett, **fråga Emil om just det fältet** istället för att gissa.

## Datamodell (tabell `arbetsformedlingen.entries`, en rad per post)

```
id: text primary key (valfri identifierare, t.ex. "jobb-<bolag>-<roll>" i kebab-case)
kind: "jobb" | "intervju" | "kurs"
title: text        -- yrkesroll (jobb/intervju) eller kursnamn (kurs)
org: text          -- arbetsgivare (jobb/intervju) eller anordnare (kurs)
status: "new" | "done" | "skip"
note: text         -- frivillig kontext, t.ex. varför status är "skip", eller vad som antogs

-- jobb only:
omfattning: text   -- fritext, t.ex. "Heltid", "Deltid", "~75%"
ort: text
annons: "ja" | "nej"
sokdatum: date

-- intervju only:
ort: text
intervjudatum: date

-- kurs only:
omfattning: text
start: date
slut: date

-- jobb/intervju only, sätts när ett avslag kommer in — se "Avslag" nedan:
avslag: boolean
avslagsdatum: date
avslagsinfo: text

created_at / updated_at: timestamptz
```

Periodinfo ligger i `arbetsformedlingen.meta_period` (en rad, `id=1`): `period_label`, `submit_window`, `earliest_date`.

## Avslag

När ett jobb eller en intervju får avslag: **skriv INTE in det i `note`** (t.ex. "AVSLAG mottaget ..."), det gör det otydligt och blandar ihop utfallet med kontexten om själva ansökan. Sätt istället de dedikerade kolumnerna på samma rad:

```
avslag = true
avslagsdatum = 'ÅÅÅÅ-MM-DD'
avslagsinfo = 'T.ex. Från Frank Brunell: gick vidare med andra kandidater'
```

`note` ska bara innehålla kontext om själva ansökan/intervjun (hur den kom till, vem som förmedlade den, etc.), aldrig utfallet. Sidan visar avslaget som en egen tydlig markering (röd "Avslag"-chip + rad) separat från `note` och från rapporteringsstatusen. Ett avslag ändrar INTE `status` automatiskt, jobbet kan redan vara `done` (rapporterat till Arbetsförmedlingen som sökt jobb, vilket det ska vara oavsett utfall) eller `new`.

## Statusvärden

- `new` = ej ännu rapporterad till Arbetsförmedlingen (standard för nya poster)
- `done` = ifylld i Arbetsförmedlingens aktivitetsrapport
- `skip` = ligger utanför öppen rapporteringsperiod eller behöver av annan anledning inte rapporteras (ange varför i `note`)

## Viktigt om Arbetsförmedlingens formulär (lärdomar)

- Aktivitetsrapportens öppna period har ett tidigaste datum som systemet accepterar (var 26 augusti 2026 för perioden augusti–september 2026). Datum före det avvisas med felmeddelande. Jämför mot `arbetsformedlingen.meta_period.earliest_date`, och uppdatera det värdet där när en ny period öppnar.
- Fältet "Yrkesroll" på arbetsformedlingen.se är en typeahead mot en fast lista. Skriv rollen och se om förslag dyker upp; matchar inget förslag, klicka "Hittar inte yrkesrollen" och skriv fritext i fältet som dyker upp istället.
- "Sök och välj ort" är fritext, ingen dropdown krävs.
- Endast en av Heltid/Deltid/Timmar vid behov kan väljas i formuläret – finns ingen exakt match (t.ex. ~75%) välj närmaste och notera det i `note`.

## Vid själva rapporteringen

När Emil ber om att fylla i formuläret på arbetsformedlingen.se: läs rader där `status != 'done'` via Supabase-connectorns `execute_sql`, fyll i formuläret för de som ligger inom perioden, och sätt `status = 'done'` (eller `'skip'` med anledning i `note`) via `execute_sql` direkt efteråt – inte bara i chatten.

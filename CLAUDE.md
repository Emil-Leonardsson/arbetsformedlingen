# Arbetsförmedlingen – aktivitetsrapportering

Detta är Emils underlag för Arbetsförmedlingens aktivitetsrapport (sökta jobb, jobbintervjuer och kurser/utbildningar).

## Den officiella källan: Supabase (riktig Postgres-databas)

Sedan 2026-09-15 ligger all data i en riktig Supabase-databas, inte längre i Claude Artifacts inbyggda databas.

- **Projekt**: `kpaolyovfybxedgaigvp` (org "Emil-Leonardsson's Project", eu-west-1)
- **Schema**: `arbetsformedlingen`, tabeller `entries` och `meta_period`
- **Migrations/schema-historik**: [Emil-Leonardsson/shared-db](https://github.com/Emil-Leonardsson/shared-db) — databasen delas med andra små personliga appar (t.ex. gym-tracker), varje app har sitt eget Postgres-schema
- **Åtkomst från Claude Code/agent-sessioner**: den officiella Supabase MCP-connectorn (kopplad till Emils claude.ai-konto). Verktygen `mcp__<uuid>__execute_sql` / `apply_migration` / `list_tables` osv (hitta uuid via `ToolSearch` om de inte redan är laddade, eller fråga Emil att aktivera connectorn i chatten om den inte syns).
- **Aktivitetsloggen** (sidan Emil faktiskt använder): https://emil-leonardsson.github.io/arbetsformedlingen/ — en fristående statisk sida (`index.html` i det här repot), **inte** en Claude Artifact längre (den gamla Artifact-versionen är borttagen, se nedan). Sidan pratar direkt med Supabase via `@supabase/supabase-js` (CDN, ingen build), skyddad av inloggning (Supabase Auth, magic link) och RLS låst till `emil.leonardsson@hotmail.com`. Repot är publikt (krävs för gratis GitHub Pages), men det avslöjar bara källkod + den publika anon-nyckeln, aldrig faktisk data, det är RLS som skyddar den.

`Arbetsformedlingen_aktivitetsrapport.md` i den här mappen är en ännu äldre historisk snapshot (markdown-eran, före både Artifact-databasen och Supabase), rör den inte.

### Historik: två tidigare arkitekturer, båda övergivna

1. **Markdown-fil** → för mycket manuellt kopierande mellan chattar.
2. **Claude Artifact med inbyggd databas** (`claude.use("db")`) → `write_db`/uppdateringar av befintliga poster fick en bugg som krävde ett `if_version`-fält som varken agent-verktyget eller sidans egen kod kunde tillgodose. Sidan låg dessutom bakom ett Claude-konto, inget stabilt publikt/CV-länkbart alternativ.

Nuvarande lösning (Supabase + GitHub Pages, sedan 2026-09-15/16) löser båda: en riktig databas utan versionskrångel, och en stabil publik URL med riktig inloggning istället för att vara låst till Claude Artifacts.

### Hur Claude Code/agent-sessioner pratar med Supabase

Direkt via Supabase MCP-connectorns `execute_sql`/`apply_migration` (se ovan). **Obs** en detalj värd att känna till om du någon gång behöver tolka `execute_sql`s textsvar från agent-sidan: det kommer inlindat i en `<untrusted-data-XXXX>...</untrusted-data-XXXX>`-boundary, och samma tagg-sträng nämns även en gång i prosan precis före den riktiga öppnande taggen, så en naiv regex som matchar första `<untrusted-data-...>` fångar fel ställe. `index.html` behöver inte hantera detta alls, den använder `supabase-js` direkt (`db.from('entries').select()` etc), som svarar med vanliga JS-objekt, ingen textparsning.

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

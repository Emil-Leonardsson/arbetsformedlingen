# Arbetsförmedlingen – aktivitetsrapportering

Detta är Emils underlag för Arbetsförmedlingens aktivitetsrapport (sökta jobb, jobbintervjuer och kurser/utbildningar).

## Den officiella källan: Aktivitetsloggen (Artifact)

**Aktivitetsloggen** är en publicerad Artifact med en delad databas: https://claude.ai/code/artifact/60593adf-e013-4b1d-81a6-99ecf164e402

Det är den enda sanningen. `Arbetsformedlingen_aktivitetsrapport.md` i den här mappen är en historisk snapshot från innan Artifacten fanns och uppdateras inte längre, låt den ligga kvar orörd som referens.

Jobbsökningen sker i den vanliga claude.ai-chatten (webb/app), som **inte** har filåtkomst men **kan** skriva till Artifact-databasen om den har länken (be Emil dela länken i den chatten första gången, eller när en ny sådan chatt startas). Både jobbsöknings-chatten och Claude Code-sessioner läser/skriver alltså samma databas via Artifact-verktygets `read_db`/`write_db` (Claude Code) eller `claude.use("db")` (sidans egen kod, som jobbsöknings-chatten triggar genom att be Emil öppna länken och interagera, eller genom att själv anropa write_db om den har Artifact-verktyget).

## Uppdatera löpande

Så fort Emil klistrar in eller berättar om ett jobb han sökt, eller en kurs/utbildning, i vilken session som helst: skriv en post till `entries`-collectionen i Artifact-databasen direkt (via `write_db` på artifact-url ovan) – vänta inte tills han ber om en sammanfattning. Om något fält saknas för att posten ska vara komplett, **fråga Emil om just det fältet** istället för att gissa.

## Datamodell (collection `entries`, ett dokument per post)

```
kind: "jobb" | "intervju" | "kurs"
title: string       // yrkesroll (jobb/intervju) eller kursnamn (kurs)
org: string          // arbetsgivare (jobb/intervju) eller anordnare (kurs)
status: "new" | "done" | "skip"
note: string          // frivillig kontext, t.ex. varför status är "skip", eller vad som antogs

// jobb only:
omfattning: string   // fritext, t.ex. "Heltid", "Deltid", "~75%"
ort: string
annons: "ja" | "nej"
sokdatum: "ÅÅÅÅ-MM-DD"

// intervju only:
ort: string
intervjudatum: "ÅÅÅÅ-MM-DD"

// kurs only:
omfattning: string
start: "ÅÅÅÅ-MM-DD"
slut: "ÅÅÅÅ-MM-DD"

// jobb/intervju only, sätts när ett avslag kommer in — se "Avslag" nedan:
avslag: boolean
avslagsdatum: "ÅÅÅÅ-MM-DD"
avslagsinfo: string

createdAt / updatedAt: ISO-tidsstämpel
```

## Avslag

När ett jobb eller en intervju får avslag: **skriv INTE in det i `note`** (t.ex. "AVSLAG mottaget ..."), det gör det otydligt och blandar ihop utfallet med kontexten om själva ansökan. Sätt istället de dedikerade fälten på samma dokument:

```
avslag: true
avslagsdatum: "ÅÅÅÅ-MM-DD"
avslagsinfo: string   // t.ex. "Från Frank Brunell: gick vidare med andra kandidater"
```

`note` ska bara innehålla kontext om själva ansökan/intervjun (hur den kom till, vem som förmedlade den, etc.), aldrig utfallet. Sidan visar avslaget som en egen tydlig markering (röd "Avslag"-chip + rad) separat från `note` och från rapporteringsstatusen. Ett avslag ändrar INTE `status` automatiskt, jobbet kan redan vara `done` (rapporterat till Arbetsförmedlingen som sökt jobb, vilket det ska vara oavsett utfall) eller `new`.

Periodinfo ligger i `meta/period`: `{ periodLabel, submitWindow, earliestDate }`.

## Statusvärden

- `new` = ej ännu rapporterad till Arbetsförmedlingen (standard för nya poster)
- `done` = ifylld i Arbetsförmedlingens aktivitetsrapport
- `skip` = ligger utanför öppen rapporteringsperiod eller behöver av annan anledning inte rapporteras (ange varför i `note`)

## Viktigt om Arbetsförmedlingens formulär (lärdomar)

- Aktivitetsrapportens öppna period har ett tidigaste datum som systemet accepterar (var 26 augusti 2026 för perioden augusti–september 2026). Datum före det avvisas med felmeddelande. Jämför mot `meta/period.earliestDate` i databasen, och uppdatera det värdet där när en ny period öppnar.
- Fältet "Yrkesroll" på arbetsformedlingen.se är en typeahead mot en fast lista. Skriv rollen och se om förslag dyker upp; matchar inget förslag, klicka "Hittar inte yrkesrollen" och skriv fritext i fältet som dyker upp istället.
- "Sök och välj ort" är fritext, ingen dropdown krävs.
- Endast en av Heltid/Deltid/Timmar vid behov kan väljas i formuläret – finns ingen exakt match (t.ex. ~75%) välj närmaste och notera det i `note`.

## Vid själva rapporteringen

När Emil ber om att fylla i formuläret på arbetsformedlingen.se: läs `entries` där `status != "done"` via `read_db`, fyll i formuläret för de som ligger inom perioden, och sätt `status: "done"` (eller `"skip"` med anledning i `note`) via `write_db` direkt efteråt – inte bara i chatten.

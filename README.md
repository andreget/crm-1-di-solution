# Crm

En liten webapp for kundebehandling (CRM). Dette er løsningsforslaget til DI-oppgaven (1C) i Modul 1. API-et gjør det samme som i `crm-1-di-start`, men alle endepunktene får nå tjenestene sine fra DI-containeren. Kunder og aktiviteter kan lagres i JSON, XML eller i minnet, og du bytter lagring ved å endre én linje i `Program.cs`.

## Hva den gjør

- **Kunder:** liste med søk og statusfilter, detaljside, opprett, endre og slett.
- **Aktiviteter:** registrer møter, samtaler, e-poster og oppgaver for en kunde, merk dem som ferdige, og se alle åpne aktiviteter i én oversikt. Forfalte aktiviteter er markert.
- **API (JSON):** alle endepunktene under `/api`, som beskrevet under.
- **Dependency injection (DI):** `ICustomerService` og `IActivityService` er registrert, og alle endepunktene får dem som parameter. Ingen endepunkter lager tjenestene selv med `new`.

## Krav

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- Visual Studio (en nyere versjon, som støtter `.http`-filer), eller en annen editor sammen med kommandolinjeverktøyet `dotnet`

Sjekk at installasjonen fungerer:

```
dotnet --version
```

## Lag din egen kopi

1. Klikk **Use this template** øverst på denne siden, og deretter **Create a new repository**.
2. Klone det nye repoet ditt:

```
git clone <url-to-your-repository>
```

## Kjør appen

**Visual Studio:** åpne `Crm.slnx` (eller `Crm.csproj`) og trykk `F5`. Nettleseren åpner <http://localhost:5080>.

**Kommandolinjen:** i mappen som inneholder `Crm.csproj`:

```
dotnet restore
dotnet run
```

Åpne deretter <http://localhost:5080>.

> **Start alltid appen fra prosjektmappen.** Maler og datafiler leses med relative stier, så sidene feiler hvis appen startes fra en annen mappe.

## API-et

Endepunktene under `/api` returnerer data som JSON, ikke HTML.

### Kunder

| Endepunkt | Beskrivelse | Svar |
|---|---|---|
| `GET /api/customers?search=&status=` | Alle kunder, eventuelt filtrert på tekst og status | 200 |
| `GET /api/customers/{id}` | Én kunde | 200, 404 |
| `POST /api/customers` | Opprett kunde | 201, 400 |
| `PUT /api/customers/{id}` | Oppdater kunde | 200, 404, 400 |
| `DELETE /api/customers/{id}` | Slett kunde og kundens aktiviteter | 204, 404 |

### Aktiviteter

| Endepunkt | Beskrivelse | Svar |
|---|---|---|
| `GET /api/customers/{id}/activities` | Aktivitetene til en kunde, nyeste først | 200, 404 |
| `POST /api/customers/{id}/activities` | Opprett aktivitet for en kunde | 201, 404, 400 |
| `GET /api/activities?open=true` | Alle aktiviteter, eller bare åpne med `open=true` | 200 |
| `PUT /api/activities/{id}/done` | Merk som ferdig eller åpen, med `{ "done": true }` | 204, 404 |
| `DELETE /api/activities/{id}` | Slett aktivitet | 204, 404 |

### Oppslagslister

| Endepunkt | Beskrivelse | Svar |
|---|---|---|
| `GET /api/statuses` | Gyldige kundestatuser | 200 |
| `GET /api/activity-types` | Gyldige aktivitetstyper | 200 |

### Valg i løsningen

- **Validering av kunde:** `name` kan ikke være tom, `email` må inneholde `@`, og `status` må være `Lead`, `Active` eller `Inactive`.
- **Validering av aktivitet:** `title` kan ikke være tom, og `type` må være `Meeting`, `Call`, `Email` eller `Task`. Mangler `date`, brukes nåværende tidspunkt.
- **Feilsvar:** Alle 400-svar har formatet `{ "error": "melding" }`.
- **201 Created:** Svaret fra `POST /api/customers` har en `Location`-header med adressen til den nye kunden. Svaret fra `POST /api/customers/{id}/activities` har ingen, siden API-et ikke har et endepunkt for å hente én aktivitet.
- **Sletting av kunde:** Aktivitetene til kunden slettes sammen med kunden.
- **Sortering:** `GET /api/customers` sorterer på navn.
- **JSON-navn:** Egenskapene i JSON starter med liten bokstav (`name`, `email`).

### Test med `.http`-filer

- `Crm.http` inneholder de tre første kallene fra API-videoen.
- `Assignment.http` inneholder alle kallene som tester løsningen, med forventet statuskode i kommentaren over hvert kall.

Åpne filen i Visual Studio, start appen, og trykk **Send request** over et kall. Kjør kallene ovenfra og ned. Noen kall bruker variabler øverst i filen (`@newCustomerId`, `@activityId` og `@tempCustomerId`). Sett dem til id-en i svaret fra opprett-kallet før du kjører de neste. Hvis appen kjører på en annen port enn 5080, endrer du `@host`.

Alle kallene skal gi de samme statuskodene og svarene uansett hvilken implementasjon som er registrert.

## DI i dette repoet

### Endret siden `crm-1-di-start`

- `CustomerService` har fått nytt navn: `JsonCustomerService`.
- `ActivityService` har fått nytt navn: `JsonActivityService`, og implementerer det nye interfacet `IActivityService`.
- Nye implementasjoner: `InMemoryCustomerService`, `XmlCustomerService`, `InMemoryActivityService` og `XmlActivityService`.
- Alle endepunktene i `Program.cs`, både HTML-sidene og API-et, får `ICustomerService` og `IActivityService` som parameter i stedet for å lage tjenestene med `new`.
- `activities.xml` følges ikke av Git.

### Tre implementasjoner bak hvert interface

| Interface | Implementasjon | Lagring | Registrering |
|---|---|---|---|
| `ICustomerService` | `JsonCustomerService` | `customers.json` | `AddScoped` |
| | `XmlCustomerService` | `customers.xml` | `AddScoped` |
| | `InMemoryCustomerService` | Liste i minnet, med de fem eksempelkundene | `AddSingleton` |
| `IActivityService` | `JsonActivityService` | `activities.json` | `AddScoped` |
| | `XmlActivityService` | `activities.xml` | `AddScoped` |
| | `InMemoryActivityService` | Liste i minnet, starter tom | `AddSingleton` |

### Bytt implementasjon

Øverst i `Program.cs` står én aktiv linje per interface. Flytt kommentaren `//` for å bytte:

```csharp
builder.Services.AddScoped<ICustomerService, JsonCustomerService>();
//builder.Services.AddScoped<ICustomerService, XmlCustomerService>();
//builder.Services.AddSingleton<ICustomerService, InMemoryCustomerService>();
```

Kunder og aktiviteter byttes hver for seg. Resten av koden er uendret, fordi endepunktene bare kjenner interfacet.

### Hvorfor `AddSingleton` for versjonene i minnet?

Med `AddScoped` lages en ny instans for hvert kall. Listen i minnet starter da på nytt hver gang, og en kunde du nettopp opprettet er borte i neste kall. Med `AddSingleton` lever instansen like lenge som appen. Fil-versjonene leser filen for hvert kall og virker med begge.

## Prosjektstruktur

```
Program.cs          Endepunkter, hjelpefunksjoner og registrering av tjenester (DI)
Models/             Customer og Activity
Services/           ICustomerService og IActivityService, med JSON-, XML- og minneimplementasjoner
Templates/          HTML-maler med {{placeholders}}
wwwroot/            CSS og JavaScript
customers.json      Eksempelkunder i JSON-format (appen endrer denne filen)
customers.xml       Eksempelkunder i XML-format (appen endrer denne filen)
Crm.http            De tre første kallene fra API-videoen
Assignment.http     Alle kallene som tester API-et. Skal gi samme svar med alle implementasjonene
Log.md              Logg og designnotater
TestResults.md      Resultatene fra kallene i Assignment.http
```

## Data

- `customers.json` og `customers.xml` inneholder fem eksempelkunder. Appen skriver til filen som hører til den registrerte implementasjonen, så den vises som endret i Git. Slik gjenoppretter du eksempeldataene:

```
git checkout customers.json
git checkout customers.xml
```

- `activities.json` og `activities.xml` opprettes første gang du legger til en aktivitet med JSON- eller XML-versjonen. De følges ikke av Git. Slett filen for å fjerne alle aktiviteter.
- Versjonene i minnet nullstilles hver gang appen startes på nytt.

Nullstill dataene før du kjører hele `Assignment.http` på nytt. Standardverdiene for variablene øverst i filen stemmer bare med ferske data.

## Feilsøking

| Problem | Hva du bør sjekke |
|---|---|
| Feilen `NETSDK1004` (assets file not found) | Kjør `dotnet restore` i mappen med `Crm.csproj`, og bygg på nytt. |
| Feil eller blank side rett etter oppstart | Appen er startet fra feil mappe. Start den fra mappen med `Crm.csproj`. |
| Port 5080 er opptatt | Endre `applicationUrl` i `Properties/launchSettings.json`, og `@host` i `.http`-filene. |
| `Send request` vises ikke over kallene i `.http`-filene | Filen må ha endelsen `.http`, og Visual Studio må være en nyere versjon. |
| Dataene ser rare ut etter eksperimentering | Gjenopprett `customers.json` og `customers.xml`, og slett `activities.json` og `activities.xml`. |
| Appen feiler med at en tjeneste ikke kan løses | Interfacet er ikke registrert med `builder.Services` i `Program.cs`, eller ingen linje for det interfacet er aktiv. |
| En kunde eller aktivitet forsvinner rett etter at den er opprettet | Versjonen i minnet er registrert med `AddScoped`. Bruk `AddSingleton`. |
| Et kall i `Assignment.http` gir 404 for en id som skulle finnes | Variablene øverst i filen peker på feil id, eller dataene er ikke nullstilt. |
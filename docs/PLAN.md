# SSPW Zwembadcalculator — Plan: van demo naar echt systeem

> Status: voorstel / planfase. Doel: de huidige losse HTML-calculator omvormen
> tot een volwaardig systeem met een beheerdashboard, login en in de database
> beheerbare prijzen, ingebouwd op sspw.nl.

## 1. Wat er nu is

- **Calculator**: één zelfstandig HTML-bestand (`index.html`) met de prijzen
  hardcoded in een `CONFIG`-blok.
- **Supabase**: leads worden al opgeslagen in de tabel `calculator_leads`.
- **GitHub**: repo `Legacymodels123/SSPW---Offerte-calc-`, branch `main`.
- **Vercel**: nog te koppelen (optioneel) voor hosting.

## 2. Wat we willen toevoegen

1. **Prijzen beheerbaar maken** zonder code (in de database).
2. **Beheerdashboard** voor SSPW met **login**:
   - prijzen aanpassen (maten, pakketonderdelen, opties, btw, CTA-links);
   - leads bekijken en beheren (status, notities, zoeken/filteren, export).
3. **Inbouwen op sspw.nl**.
4. **Afwerking**: e-mailnotificatie bij nieuwe lead, statistieken, export.

## 3. Architectuur

Drie onderdelen, bovenop de bestaande Supabase + GitHub + Vercel-stack:

```
  Bezoeker (sspw.nl)                SSPW-eigenaar
        │                                 │
        ▼                                 ▼
  ┌───────────────┐               ┌──────────────────┐
  │  Calculator   │               │  Admin-dashboard │
  │  (HTML embed) │               │  (Next.js/Vercel)│
  └──────┬────────┘               └─────────┬────────┘
         │ leest prijzen / schrijft lead     │ login + CRUD
         ▼                                   ▼
  ┌──────────────────────────────────────────────────┐
  │                    SUPABASE                        │
  │  Postgres (prijzen + leads) · Auth (login) · RLS   │
  └──────────────────────────────────────────────────┘
```

- **Supabase** = database + login + beveiliging (Row Level Security).
- **Calculator** = de HTML, leest prijzen live uit Supabase (met fallback naar
  hardcoded prijzen als de database even niet bereikbaar is).
- **Admin-dashboard** = kleine Next.js-webapp op Vercel, met Supabase-login.

## 4. Datamodel (Supabase-tabellen)

Genormaliseerd zodat de prijzen via formuliertjes te bewerken zijn:

| Tabel | Doel | Belangrijkste velden |
|---|---|---|
| `pricing_sizes` | de standaardmaten | `label`, `vol_label`, `m3`, `base_price`, `sort_order`, `active` |
| `pricing_package_items` | onderdelen standaardpakket | `name`, `prices` (5 bedragen), `sort_order` |
| `pricing_option_groups` | optiegroepen | `key`, `label`, `type` (single/multi), `hint`, `optional`, `exclusive_sets`, `sort_order` |
| `pricing_options` | losse opties | `group_id`, `name`, `description`, `prices` (5 bedragen), `sort_order` |
| `settings` | losse instellingen | `btw_tarief`, `cta_quote_url`, `cta_call_url`, … |
| `calculator_leads` | leads (bestaat al) | + `status`, `notes`, `assigned_to` toevoegen |
| `admins` | wie mag inloggen/beheren | gekoppeld aan Supabase Auth-gebruiker |

> Alternatief (sneller te bouwen, minder flexibel): hele prijsconfig als één
> JSON-record met versiegeschiedenis. Aanbeveling: genormaliseerd, want dat
> geeft een net bewerkbaar dashboard.

## 5. Beveiliging (Row Level Security)

- **Prijstabellen**: iedereen mag *lezen* (calculator), alleen ingelogde
  admins mogen *schrijven*.
- **`calculator_leads`**: bezoekers mogen alleen *insert* (lead insturen),
  alleen admins mogen lezen/wijzigen.
- **Admins**: beheerd via Supabase Auth; rol-check via `admins`-tabel.

## 6. Fasering

### Fase 1 — Prijzen naar de database (fundament)
- Tabellen aanmaken + huidige prijzen migreren.
- Calculator aanpassen: prijzen bij het laden uit Supabase halen (met fallback).
- RLS instellen (publiek lezen, admin schrijven).
- **Resultaat**: prijzen zijn centraal beheerd; calculator werkt ongewijzigd.

### Fase 2 — Admin-dashboard met login
- Next.js-project, Supabase-login (e-mail + wachtwoord of magic link).
- **Prijsbeheer**: maten, pakketonderdelen, opties, btw en CTA-links bewerken.
- **Leads-inbox**: lijst, detail, status (nieuw/benaderd/offerte/gewonnen/verloren),
  notities, zoeken/filteren.
- "Publiceren"-knop zodat prijswijzigingen bewust live gaan.
- **Resultaat**: SSPW beheert alles zelf via een afgeschermde omgeving.

### Fase 3 — Inbouwen op sspw.nl + afwerking
- Embed op sspw.nl (methode hangt af van het platform — zie §8).
- E-mailnotificatie bij nieuwe lead (Supabase Edge Function).
- Export van leads (CSV/Excel) + eenvoudige statistieken.
- **Resultaat**: volledig live op de website, met opvolging en inzicht.

## 7. Hosting & kosten (indicatie)

- **Supabase**: gratis tier volstaat om te starten; later ~$25/mnd (Pro).
- **Vercel**: let op — het gratis Hobby-plan is alleen voor niet-commercieel
  gebruik. Voor een zakelijk dashboard is **Vercel Pro (~$20/mnd)** nodig,
  óf we hosten het dashboard elders.
- **Domein**: dashboard op een subdomein, bijv. `dashboard.sspw.nl`.

## 8. Inbouwen op sspw.nl (afhankelijk van platform)

- **WordPress** → embed via iframe/HTML-blok of een kleine plugin.
- **Custom/Webflow/Shopify** → iframe of script-embed.
- De calculator blijft een zelfstandig stuk dat we via een embed inladen, zodat
  de website zelf nauwelijks aangeraakt hoeft te worden.

## 9. Wat ik van SSPW nodig heb om te starten

1. **Platform van sspw.nl** (zie hieronder hoe te bepalen).
2. **Supabase-projecttoegang** (welk project; ik kan de tabellen opzetten).
3. **Login-keuzes**: e-mail+wachtwoord of magic link; welke e-mailadressen
   admin worden.
4. **Branding** voor het dashboard (logo/kleuren — huisstijl is al bekend).

### Platform achterhalen (5 seconden)
- Open `https://sspw.nl`, rechtermuisknop → **Paginabron bekijken**, zoek
  (Ctrl/Cmd+F) op `wp-content`. Gevonden → **WordPress**.
- Of ga naar `https://sspw.nl/wp-admin` → verschijnt een inlogscherm →
  **WordPress**.

## 10. Aanbevolen eerste concrete stap

Fase 1: prijzen naar Supabase. Dit is het fundament en verandert niets aan de
werking voor de bezoeker, maar maakt al het andere mogelijk.

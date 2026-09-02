# Car Maintenance Data Schema (v0.1 — POC)

## 1. `vehicles` — one entry per make/model/variant/engine/fuel combination

```json
{
  "vehicleId": "maruti-swift-2022-k12n-petrol-mt",
  "make": "Maruti Suzuki",
  "model": "Swift",
  "generation": "2022 (3rd gen, Heartect)",
  "variant": "VXI / ZXI / ZXI+",
  "engineCode": "Z12E",
  "fuelType": "Petrol",
  "transmission": "MT | AMT",
  "marketsSoldIn": ["India"],
  "sourceNote": "Compiled from Maruti Suzuki India service literature; verify exact items against the service booklet supplied with the specific car, as figures can vary by exact trim/engine revision."
}
```

Why this granularity: a Swift petrol MT and a Swift CNG AMT genuinely have different intervals (e.g., CNG needs extra valve checks). Fuel type + transmission + engine code is the minimum needed to not mislead a user.

## 2. `serviceIntervals` — one entry per due point for a given `vehicleId`

```json
{
  "vehicleId": "maruti-swift-2022-k12n-petrol-mt",
  "intervalId": "swift-10k-12mo",
  "kmThreshold": 10000,
  "monthThreshold": 12,
  "wholeVisitLabel": "3rd service (first paid service)",
  "isComplimentary": false,
  "tasks": [
    {
      "taskId": "engine-oil-replace",
      "category": "Engine",
      "action": "Replace",
      "plainLanguage": "Engine oil and oil filter get replaced. This is routine and expected at this visit.",
      "payNote": "Labour is free under Maruti's schedule; you pay for the oil and filter themselves."
    },
    {
      "taskId": "air-filter-inspect",
      "category": "Engine",
      "action": "Inspect",
      "plainLanguage": "Air filter gets checked and cleaned if needed — it should NOT be replaced yet unless visibly damaged or very dirty.",
      "payNote": "Should be free if it's just a check/clean."
    }
  ]
}
```

Why `action` matters: this is the field that directly fights overcharging. If a task's `action` is `Inspect` but the service center bills you for `Replace`, that's a flag — the app can literally say "this should be a free check, not a replacement" in plain words.

## 3. `taskCatalog` — shared definitions so wording is consistent across all vehicles

```json
{
  "taskId": "engine-oil-replace",
  "displayName": "Engine oil & filter change",
  "whyItMatters": "Old oil loses its ability to lubricate and cool the engine, causing faster wear.",
  "typicalRedFlags": [
    "Being told the oil needs changing before the due km/month mark with no evidence (e.g. no oil-life warning)",
    "Being upsold a 'premium' oil grade not specified in the manual without being told it's optional"
  ]
}
```

## 4. App output logic (plain-language generation)

Given a `vehicleId`, `currentOdometerKm`, and `lastServiceDate`:
1. Find the next `serviceIntervals` entry(ies) whose `kmThreshold` or `monthThreshold` is closest to/already passed.
2. For each task in that interval, render `plainLanguage` + `payNote`.
3. Flag anything from the catalog's `typicalRedFlags` as a callout: "Ask why, if this is proposed."
4. Explicitly list what is **NOT** due yet, since dealers upselling early is the #1 complaint — e.g. "Your air filter isn't due for replacement until 20,000 km — if a replacement is pushed today, ask why."

---

**Data-sourcing caveat (important):** the interval data itself must come from actual owner's manuals/service booklets to be trustworthy. I compiled representative POC data below from public service-guide sources, not the primary manuals — good enough to prove the concept, not yet good enough to ship. Before launch, every vehicle's schedule needs to be checked against the real manual/booklet.

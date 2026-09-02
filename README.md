# Spotter Food Catalog Data Pipeline

## Overview

This folder contains the **food-catalog data pipeline** used to build the initial global `Food` catalog for Spotter.

The goal of this pipeline is not to dump a public dataset directly into PostgreSQL.

The goal is to take authoritative food-composition data, understand its structure, normalize it into **Spotter's own canonical data model**, validate it, remove or resolve records that would create a bad user experience, and finally produce **one deterministic CSV that is safe to import into the application database**.

The complete process is:

```text
USDA FoodData Central
        │
        ├── Foundation Foods
        ├── SR Legacy
        └── FNDDS
        │
        ▼
RAW USDA CSV FILES
        │
        ▼
Notebook 1
Normalization + structural validation
        │
        ├── staging data
        ├── rejected data
        ├── duplicate candidates
        └── schema-shaped candidate data
        │
        ▼
Notebook 2
Final product-level catalog cleanup
        │
        ├── invalid-row removal
        ├── mixed-dish removal
        ├── source-ID deduplication
        ├── exact-name deduplication
        ├── conflict auditing
        └── final invariant checks
        │
        ▼
foods_final.csv
        │
        ▼
Node / Prisma importer
        │
        ▼
PostgreSQL `foods` table
```

The important architectural principle is:

> **USDA does not define Spotter's database model. Spotter defines its own model, and USDA is transformed to fit it.**

This keeps the application independent from the structure of any external dataset.

---

# 1. Why this pipeline exists

A nutrition dataset is not automatically a good application catalog.

Public datasets frequently contain:

- very technical food names;
- overlapping records from different source systems;
- multiple preparation states;
- nutrients stored in long relational tables rather than one row per food;
- different category systems;
- missing nutrient values;
- old and new identifiers;
- foods that behave more like recipes than atomic foods;
- records intended for scientific analysis rather than consumer search.

If Spotter imported the raw USDA data directly, users could encounter things such as:

```text
Chicken, broilers or fryers, breast, meat only, cooked, roasted
Chicken, broilers or fryers, breast, meat only, cooked, grilled
Chicken, roasting, meat only, cooked, roasted
Chicken, broilers or fryers, breast, meat only, raw
...
```

That may be scientifically useful, but it is not automatically a good product experience.

The pipeline therefore separates two concerns:

```text
DATA SOURCE
"What USDA knows"

        ↓ normalization

SPOTTER CATALOG
"What Spotter chooses to expose"
```

---

# 2. Final Spotter `Food` model

The pipeline is designed around this application model:

```prisma
model Food {
  id String @id @default(uuid()) @db.Uuid

  nameEn   String  @map("name_en") @db.VarChar(200)
  nameAr   String? @map("name_ar") @db.VarChar(200)
  aliases  Json?   @db.JsonB
  category String? @db.VarChar(100)

  caloriesPer100g          Decimal @map("calories_per_100g") @db.Decimal(8, 2)
  proteinGramsPer100g      Decimal @map("protein_grams_per_100g") @db.Decimal(8, 2)
  carbohydrateGramsPer100g Decimal @map("carbohydrate_grams_per_100g") @db.Decimal(8, 2)
  fatGramsPer100g          Decimal @map("fat_grams_per_100g") @db.Decimal(8, 2)

  createdByUserId String? @map("created_by_user_id") @db.Uuid

  source           String? @db.VarChar(50)
  externalSourceId String? @map("external_source_id") @db.VarChar(100)
  sourceVersion    String? @map("source_version") @db.VarChar(50)

  isActive   Boolean   @default(true) @map("is_active")
  archivedAt DateTime? @map("archived_at") @db.Timestamptz(6)

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(6)
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz(6)

  createdByUser User? @relation(
    "UserCreatedFoods",
    fields: [createdByUserId],
    references: [id],
    onDelete: Cascade
  )

  recipeIngredients RecipeIngredient[]
  mealItems         MealItem[]

  @@unique([source, externalSourceId])
  @@index([nameEn])
  @@index([category])
  @@index([createdByUserId])
  @@index([isActive])

  @@map("foods")
}
```

The pipeline intentionally does **not** generate:

```text
id
created_at
updated_at
```

Those belong to PostgreSQL / Prisma.

Imported USDA foods are global Spotter catalog records, therefore:

```text
created_by_user_id = NULL
```

User-created foods are created later by the application itself and are **not part of this ingestion pipeline**.

---

# 3. Why nutrition is normalized per 100 grams

Spotter uses grams as the canonical internal quantity for normal foods.

For example:

```text
Chicken breast
165 kcal / 100g
31g protein / 100g
```

If the user logs:

```text
180g
```

Spotter calculates:

```text
ratio = 180 / 100 = 1.8

calories = 165 × 1.8
protein  = 31  × 1.8
```

This is much cleaner than storing arbitrary values such as:

```text
1 serving
1 slice
1 cup
1 scoop
1 egg
```

as the nutritional source of truth.

Those human-friendly portions may be added later as UX conversions, but nutrition remains normalized to grams.

Therefore every imported row must provide:

```text
calories_per_100g
protein_grams_per_100g
carbohydrate_grams_per_100g
fat_grams_per_100g
```

---

# 4. Data source decision

The food catalog uses **USDA FoodData Central**.

Three USDA data types are used:

```text
Foundation Foods
SR Legacy
FNDDS
```

The following are deliberately excluded from this first catalog:

```text
Branded Foods
Full Download of All Data Types
```

## 4.1 Foundation Foods

Foundation Foods are detailed analytical records for basic or minimally processed foods.

They are valuable because they provide high-quality nutritional data.

Think conceptually of foods such as:

```text
apple
egg
rice
chicken
lentils
milk
```

Foundation is treated as the **highest-priority source** when two USDA data types contain the same normalized food name.

---

## 4.2 SR Legacy

SR Legacy is USDA's historic Standard Reference database.

It provides broad food coverage and many preparation states.

Examples:

```text
Potato, raw
Potato, boiled
Potato, baked

Chicken breast, raw
Chicken breast, roasted
Chicken breast, fried
```

These are not automatically duplicates.

Preparation state can materially change nutrition and therefore represents a meaningful product distinction.

SR Legacy is useful for coverage, but it is the lowest source priority when the same normalized food name exists in a newer source.

---

## 4.3 FNDDS

FNDDS stands for:

> Food and Nutrient Database for Dietary Studies

FNDDS is particularly interesting for Spotter because it represents foods closer to what people report actually eating.

It also uses its own WWEIA category system.

Examples can include more consumption-oriented foods than Foundation alone.

FNDDS is given the second-highest source priority.

---

## 4.4 Why Branded Foods are excluded

Branded Foods contain specific commercial products.

Examples:

```text
specific yogurt brand
specific cereal
specific protein bar
specific frozen meal
specific soda
```

This is valuable later for features such as:

```text
barcode scanning
brand-specific search
packaged supermarket foods
```

But it creates major additional complexity:

- huge dataset size;
- duplicate product variants;
- changing labels;
- manufacturer-specific serving sizes;
- brand search;
- barcode identifiers;
- frequent updates.

Spotter's first catalog focuses on **generic foods**, so Branded Foods are intentionally excluded.

---

# 5. Project directory structure

Recommended structure:

```text
Spotter-EDA/
│
├── data/
│   │
│   ├── raw/
│   │   └── usda/
│   │       ├── foundation/
│   │       │   └── ... USDA CSV files ...
│   │       │
│   │       ├── sr_legacy/
│   │       │   └── ... USDA CSV files ...
│   │       │
│   │       └── fndds/
│   │           └── ... USDA CSV files ...
│   │
│   ├── processed/
│   │   ├── usda_foods_stage.csv
│   │   ├── foods_for_db.csv
│   │   ├── usda_foods_rejected.csv
│   │   └── usda_duplicate_name_candidates.csv
│   │
│   └── final/
│       ├── foods_final.csv
│       │
│       └── audit/
│           ├── dropped_invalid_rows.csv
│           ├── dropped_mixed_dishes.csv
│           ├── dropped_exact_name_duplicates.csv
│           └── duplicate_conflicts.csv
│
├── spotter_usda_food_normalization.ipynb
├── spotter_finalize_food_catalog.ipynb
└── README.md
```

The three data layers matter:

```text
raw
processed
final
```

They should not be treated as interchangeable.

---

# 6. Data-layer philosophy

## `raw/`

This contains the original downloaded USDA data.

Do not manually edit these files.

They are the immutable source material.

If normalization logic changes, the pipeline should always be able to start again from raw data.

---

## `processed/`

This contains normalized and diagnostic intermediate data.

These files are useful for:

```text
EDA
debugging
duplicate analysis
validation analysis
category analysis
pipeline development
```

They are not automatically the final database catalog.

---

## `final/`

This contains product-ready data.

Only:

```text
foods_final.csv
```

is intended to be imported into the `foods` table.

The `audit/` files explain what was removed or flagged during finalization.

---

# 7. Notebook 1 — normalization

Notebook:

```text
spotter_usda_food_normalization.ipynb
```

Purpose:

> Convert three different USDA datasets into one consistent Spotter-shaped staging dataset.

It performs **structural normalization**, not final catalog curation.

---

# 8. USDA data is relational

USDA does not provide one convenient CSV containing:

```text
food name
calories
protein
carbs
fat
category
```

Instead, information is distributed across multiple files.

The pipeline reconstructs the food records using joins.

Conceptually:

```text
food.csv
   │
   ├──────── food_nutrient.csv
   │             │
   │             ├── calories
   │             ├── protein
   │             ├── carbohydrate
   │             └── fat
   │
   ├──────── food_category.csv
   │
   ├──────── foundation_food.csv
   │
   ├──────── sr_legacy_food.csv
   │
   └──────── survey_fndds_food.csv
                 │
                 └── wweia_food_category.csv
```

Different source types require slightly different metadata joins.

---

# 9. Nutrient extraction

The four required Spotter nutrients are extracted from USDA's long-form nutrient table.

Important USDA nutrient IDs used by the notebook:

```text
1003 → Protein
1004 → Total lipid / fat
1005 → Carbohydrate
1008 → Legacy Energy
2047 → Energy using Atwater general factors
2048 → Energy using Atwater specific factors
```

A typical `food_nutrient.csv` structure looks conceptually like:

```text
fdc_id | nutrient_id | amount
--------------------------------
123    | 1003        | 31.0
123    | 1004        | 3.6
123    | 1005        | 0.0
123    | 1008        | 165
```

The notebook filters to only the nutrient IDs required by Spotter and then pivots them into one row per food.

---

# 10. Energy handling

Energy requires special handling.

Older USDA data commonly uses:

```text
1008
```

for energy.

Newer Foundation records may use:

```text
2047
2048
```

instead.

The notebook therefore chooses energy using this order:

```text
2048
 ↓ if unavailable
2047
 ↓ if unavailable
1008
```

This prevents valid Foundation foods from being rejected simply because they do not use the legacy energy field.

The staging file retains the energy nutrient ID used so this can be audited.

---

# 11. Source identifiers

Spotter stores:

```text
source
external_source_id
source_version
```

These fields are important for provenance and future updates.

Example:

```text
source = USDA_FDC_FOUNDATION
external_source_id = 12345
source_version = 2026-04-30
```

The purpose is not merely documentation.

The Prisma model contains:

```prisma
@@unique([source, externalSourceId])
```

This gives the future importer a stable conflict key.

That allows idempotent imports:

```text
first import
→ create food

same dataset imported again
→ update/upsert existing food

not
→ create duplicate food
```

---

# 12. Why `fdc_id` alone is not treated as the only identity

FoodData Central uses FDC IDs as record identifiers, but source-specific identifiers may also exist.

The first notebook tries to preserve more source-stable identifiers when available:

```text
Foundation / SR Legacy
→ NDB number when available

FNDDS
→ Food Code when available

fallback
→ FDC ID
```

The raw FDC ID is still retained in staging for traceability.

This gives the application better provenance while keeping USDA implementation details outside the production schema.

---

# 13. Category normalization

The three USDA sources do not necessarily share the same category system.

For example:

```text
Foundation / SR Legacy
→ Standard Reference categories

FNDDS
→ WWEIA categories
```

Spotter does not want its application logic permanently tied to USDA category labels.

The first notebook therefore maps raw source categories into a smaller Spotter-oriented taxonomy.

Current broad categories include:

```text
MIXED_DISHES
FATS_OILS
SEAFOOD
POULTRY
MEAT
EGGS
DAIRY
LEGUMES
NUTS_SEEDS
FRUIT
VEGETABLE
GRAINS
BEVERAGES
SWEETS_SNACKS
CONDIMENTS
OTHER
```

This classification is intentionally coarse.

The original USDA category is preserved in the staging data so the mapping can be changed later without redownloading the source dataset.

---

# 14. Why categories are coarse

The category is currently used mainly for:

```text
catalog browsing
filtering
EDA
product organization
```

It is not intended to encode every possible nutritional or culinary distinction.

A smaller taxonomy is easier to maintain and easier for frontend users to understand.

The application can expand it later when real product requirements justify more detail.

---

# 15. Name normalization

The pipeline distinguishes between:

```text
display name
comparison key
```

The USDA English description is cleaned only structurally:

```text
Unicode normalization
whitespace cleanup
trim
```

The pipeline intentionally does **not** aggressively rewrite names.

For duplicate detection, it creates a private normalized comparison key by:

```text
lowercasing
removing punctuation
collapsing whitespace
```

Example:

```text
"Chicken breast, grilled"
"CHICKEN BREAST - GRILLED"
" chicken breast grilled "
```

may produce the same comparison key.

But the displayed `name_en` remains preserved.

This avoids accidentally changing food semantics during ingestion.

---

# 16. Arabic names and aliases

The current pipeline intentionally sets:

```text
name_ar = NULL
aliases = NULL
```

The initial catalog is English-only.

This is deliberate.

Arabic localization and search aliases are separate product/data-quality tasks.

They should not be automatically generated during nutritional ingestion unless a trusted source and review process are defined.

---

# 17. Validation in Notebook 1

Your Prisma model requires all four major nutrition values.

Therefore the notebook does **not** silently transform missing nutrition into zero.

For example:

```text
Food X

calories = 120
protein  = 4
carbs    = missing
fat      = 3
```

The notebook does not do:

```text
carbs = 0
```

because missing and zero have different meanings.

Instead, the row is rejected from the schema-ready candidate file and written to:

```text
usda_foods_rejected.csv
```

with a reason.

Possible reasons include:

```text
missing_name
name_over_200_chars

missing_calories_per_100g
missing_protein_grams_per_100g
missing_carbohydrate_grams_per_100g
missing_fat_grams_per_100g

negative_calories_per_100g
negative_protein_grams_per_100g
...

missing_external_source_id
```

---

# 18. Notebook 1 outputs

## `usda_foods_stage.csv`

This is the broad normalized staging file.

It contains:

```text
Spotter-oriented fields
+
diagnostic USDA fields
```

Use it for:

```text
EDA
debugging
source comparison
category analysis
nutrient inspection
```

Do not import it directly into PostgreSQL.

---

## `usda_foods_rejected.csv`

Contains rows that failed structural validation.

Do not import them.

Use this file to understand whether:

```text
the source data is incomplete
the pipeline needs improvement
or the row truly cannot satisfy the Spotter schema
```

---

## `usda_duplicate_name_candidates.csv`

Contains rows whose normalized English names collide.

Important:

> This does not mean the data is automatically wrong.

It means:

```text
multiple USDA records
appear to represent the same user-facing name
```

This file exists because duplicate removal requires product reasoning.

---

## `foods_for_db.csv`

This is a schema-shaped candidate file.

It contains the correct columns for the Prisma `Food` model.

However, after Notebook 1 it is still considered **candidate data**, because duplicate resolution and product-level catalog decisions have not yet been applied.

Notebook 2 consumes this file.

---

# 19. Understanding duplicates

The word "duplicate" can be misleading.

There are several distinct cases.

## Case A — true repeated user-facing food

Example:

```text
Foundation
Egg, whole, raw

SR Legacy
Egg, whole, raw
```

A user should probably not see:

```text
Egg, whole, raw
Egg, whole, raw
```

in search results.

One source should win.

---

## Case B — meaningful preparation differences

Example:

```text
Potato, raw
Potato, boiled
Potato, fried
```

These are **not duplicates**.

Preparation changes composition and energy density.

All may remain in the catalog.

---

## Case C — same normalized name, different nutrition

Example:

```text
Foundation
Chicken breast, roasted
165 kcal

SR Legacy
Chicken breast, roasted
171 kcal
```

This is still probably the same user-facing food, but the source measurements differ.

The finalization notebook chooses a deterministic winner and records the disagreement in an audit file.

---

# 20. Why simple `drop_duplicates("name")` is dangerous

A careless implementation such as:

```python
df.drop_duplicates("name")
```

would make decisions without understanding:

```text
source quality
source recency
nutrition disagreement
normalization differences
product behavior
```

Instead, Spotter uses an explicit source priority and preserves conflict information for later inspection.

---

# 21. Notebook 2 — final catalog construction

Notebook:

```text
spotter_finalize_food_catalog.ipynb
```

Purpose:

> Convert normalized candidate data into the actual global Spotter food catalog.

This is the notebook that produces:

```text
foods_final.csv
```

---

# 22. Product-level rules in Notebook 2

Notebook 2 applies the following final policy:

```text
1. Verify required columns
2. Standardize types
3. Reject impossible/missing records
4. Remove mixed dishes
5. Enforce source-ID uniqueness
6. Detect exact normalized-name duplicates
7. Audit nutritional conflicts
8. Choose one source winner
9. Assert final database invariants
10. Export the final CSV
```

---

# 23. Final validation rules

A final database row must have:

```text
name_en
calories_per_100g
protein_grams_per_100g
carbohydrate_grams_per_100g
fat_grams_per_100g
source
external_source_id
```

The finalizer also rejects clearly impossible values such as:

```text
negative nutrition
calories > 1000 per 100g
protein > 100g per 100g
carbohydrate > 100g per 100g
fat > 100g per 100g
```

These are final catalog safeguards, not claims that every nutrition record outside these boundaries is scientifically impossible under every imaginable representation.

The purpose is to protect the application from obvious ingestion errors.

---

# 24. Why `MIXED_DISHES` are removed

Spotter's architecture separates:

```text
Food
Recipe
RecipeIngredient
```

A `Food` is the reusable base nutrition catalog.

Examples:

```text
chicken breast
rice
olive oil
tomato
egg
banana
```

A `Recipe` is a constructed/prepared dish.

Examples:

```text
Koshary
Chicken rice
Lasagna
Homemade protein oats
```

Therefore FNDDS rows categorized as:

```text
MIXED_DISHES
```

are excluded by default from the final `Food` catalog.

Otherwise Spotter could end up mixing two concepts:

```text
Food: Chicken sandwich
Recipe: Chicken sandwich
```

The removal is configurable and fully audited.

If the product later intentionally wants prepared generic dishes inside `Food`, this policy can be changed.

---

# 25. Source priority

When two or more USDA records share the same normalized food name, the current winner order is:

```text
1. USDA_FDC_FOUNDATION
2. USDA_FDC_FNDDS
3. USDA_FDC_SR_LEGACY
```

Conceptually:

```text
Foundation
    ↓
FNDDS
    ↓
SR Legacy
```

This policy gives preference to current analytical data while preserving FNDDS as the preferred consumption-oriented fallback before SR Legacy.

---

# 26. Duplicate conflict detection

Not every duplicate group agrees nutritionally.

The notebook considers a same-name duplicate group noteworthy when differences exceed configured tolerances.

Current audit thresholds:

```text
calories:      > 10 kcal / 100g
protein:       > 2 g / 100g
carbohydrate:  > 2 g / 100g
fat:           > 2 g / 100g
```

These thresholds do **not** determine whether the final export succeeds.

They determine whether the duplicate group is written to:

```text
duplicate_conflicts.csv
```

for inspection.

The selected source still follows the explicit priority rule.

This gives us both:

```text
deterministic output
+
human auditability
```

---

# 27. Final deduplication behavior

For each normalized food-name group:

```text
sort by source priority
        ↓
select preferred source
        ↓
keep its nutrition
        ↓
drop remaining same-name rows
```

Nutrition is never combined or averaged across sources.

For example, Spotter does **not** do this:

```text
Foundation calories = 165
SR calories         = 171

average = 168
```

That would create a synthetic value that no source actually reported.

Instead:

```text
Foundation wins
→ 165 is preserved
```

---

# 28. Category inheritance during duplicate resolution

There is one limited case where duplicate metadata may help the selected winner.

If the preferred winner has:

```text
category = NULL
```

or:

```text
category = OTHER
```

and another same-name source contains a more useful normalized category, the finalizer may reuse that category.

It does **not** mix nutrition values.

Only category metadata may be improved this way.

---

# 29. Final invariants

Before writing `foods_final.csv`, the notebook asserts that:

```text
(source, external_source_id) is unique

normalized food name is unique

required nutrition is not missing

nutrition is non-negative

name_en length <= 200

created_by_user_id is NULL
```

If these assertions fail, the notebook stops.

This is intentional.

A final export should fail loudly rather than silently create a corrupted production seed.

---

# 30. Final output

The file intended for the database is:

```text
data/final/foods_final.csv
```

Only this file should be passed to the future Prisma import script.

Its columns are:

```text
name_en
name_ar
aliases
category

calories_per_100g
protein_grams_per_100g
carbohydrate_grams_per_100g
fat_grams_per_100g

created_by_user_id

source
external_source_id
source_version

is_active
archived_at
```

Database-generated fields are intentionally absent:

```text
id
created_at
updated_at
```

---

# 31. Audit outputs

Notebook 2 creates:

```text
data/final/audit/
```

These files should **not** be imported.

They exist to make the pipeline explainable.

---

## `dropped_invalid_rows.csv`

Contains rows removed because they did not satisfy the final Food contract.

---

## `dropped_mixed_dishes.csv`

Contains records removed because they are classified as mixed dishes and therefore belong more naturally in the future Recipe pipeline.

---

## `dropped_exact_name_duplicates.csv`

Contains same-name rows that lost during source-priority deduplication.

The audit also records which row was kept.

---

## `duplicate_conflicts.csv`

Contains duplicate-name groups whose nutritional values differ enough to deserve review.

This is especially useful for future catalog-quality work.

---

# 32. The final CSV is not the same thing as the raw USDA dataset

After this pipeline, Spotter owns a curated canonical representation.

Conceptually:

```text
USDA
"What exists scientifically"

        ↓

Spotter normalization

        ↓

Spotter Food catalog
"What our product exposes"
```

This separation is important because the application can later add additional food sources without changing its API contract.

For example:

```text
USDA
Canada Nutrient File
regional food database
manual Spotter curation
```

could all eventually normalize to the same `Food` model.

---

# 33. Future database importer

The notebooks stop at CSV export.

They do **not** directly connect to the production database.

That is intentional.

The next layer should be a Node.js / Prisma importer.

Recommended flow:

```text
foods_final.csv
       ↓
Node import script
       ↓
parse + validate
       ↓
Prisma upsert
       ↓
foods table
```

Use the existing unique key:

```prisma
@@unique([source, externalSourceId])
```

so imports are idempotent.

Conceptually:

```text
same source + same external ID exists
→ update / preserve record

does not exist
→ create record
```

This is better than:

```text
delete entire foods table
reinsert everything
```

because existing recipe and meal relationships may eventually depend on stable food IDs.

---

# 34. Why foods should be archived instead of deleted

The Prisma model includes:

```text
isActive
archivedAt
```

This is intentional.

Once foods are referenced by:

```text
RecipeIngredient
MealItem
```

destructive deletion can damage historical and relational integrity.

The safer lifecycle is:

```text
active food
   ↓
source becomes outdated / food should disappear from search
   ↓
isActive = false
archivedAt = timestamp
```

Historical references remain valid.

---

# 35. Relationship to meal logging

The global catalog feeds meal logging.

Conceptually:

```text
Food
  │
  │ user selects 180g
  ▼
MealItem
```

The service calculates:

```text
food nutrition / 100g
× consumed grams
```

and stores the result in `MealItem`.

The `MealItem` stores a nutrition snapshot so historical meals do not change when the Food catalog changes later.

---

# 36. Relationship to recipes

The same Food catalog is also the base for constructed recipes.

Conceptually:

```text
Food
  ▲
  │
RecipeIngredient
  ▲
  │
Recipe
```

Example:

```text
My Chicken Rice

RecipeIngredient
├── Chicken breast → 300g
├── Rice           → 200g
└── Olive oil      → 15g
```

The backend reads Food nutrition and calculates recipe totals.

Therefore catalog-quality problems propagate into recipe calculations, which is why this normalization pipeline is treated seriously.

---

# 37. User-created foods are separate from imported foods

Imported foods have:

```text
created_by_user_id = NULL
```

A user-created food will have:

```text
created_by_user_id = <user UUID>
```

The same table supports both concepts.

Example:

```text
USDA global food
Chicken breast
createdByUserId = null
```

versus:

```text
User food
Nour's homemade protein shake
createdByUserId = user UUID
```

The data pipeline is responsible only for global imported foods.

---

# 38. Search is intentionally outside the data pipeline

The notebooks do not implement application search.

After import, the backend will own:

```text
search
pagination
category filtering
active/inactive filtering
ownership filtering
```

A typical future query will conceptually search:

```text
global foods
+
current user's custom foods
```

while excluding:

```text
other users' custom foods
archived foods
```

Keeping search in the application layer prevents the frontend from depending on USDA's data structure or API.

---

# 39. Why no external nutrition API is called at runtime

The selected architecture is:

```text
external dataset
      ↓
offline ingestion
      ↓
Spotter PostgreSQL
      ↓
Spotter API
      ↓
frontend
```

not:

```text
frontend
   ↓
Spotter
   ↓
external nutrition API
   ↓
Spotter
   ↓
frontend
```

This gives Spotter control over:

```text
latency
availability
search
pagination
ranking
categories
naming
data normalization
cost
source updates
product behavior
```

It also avoids making core meal logging permanently dependent on an external provider's runtime uptime and API contract.

---

# 40. How to run the pipeline

## Step 1 — download USDA data

Download:

```text
Foundation Foods
SR Legacy
FNDDS
```

Do not use:

```text
Branded
Full Download
```

for the current MVP catalog.

---

## Step 2 — extract files

Place them under:

```text
data/raw/usda/foundation/
data/raw/usda/sr_legacy/
data/raw/usda/fndds/
```

Folder names must match the notebook configuration.

For example:

```text
sr_legacy
```

is not the same path as:

```text
sr-legacy
```

unless the notebook configuration is changed accordingly.

---

## Step 3 — run Notebook 1

Open:

```text
spotter_usda_food_normalization.ipynb
```

Run the cells from top to bottom.

Expected processed outputs:

```text
data/processed/usda_foods_stage.csv
data/processed/foods_for_db.csv
data/processed/usda_foods_rejected.csv
data/processed/usda_duplicate_name_candidates.csv
```

---

## Step 4 — inspect Notebook 1 results

At minimum, inspect:

```text
row counts
category counts
rejected count
duplicate-name count
sample food names
nutrition outliers
```

This is the EDA checkpoint.

The goal is to understand the normalized data before finalization.

---

## Step 5 — run Notebook 2

Open:

```text
spotter_finalize_food_catalog.ipynb
```

Run all cells.

The notebook will:

```text
remove invalid rows
remove mixed dishes
deduplicate source IDs
resolve exact-name duplicates
create conflict audits
assert invariants
```

---

## Step 6 — verify the final report

Notebook 2 displays:

```text
input processed rows
invalid rows removed
mixed dishes removed
exact-name duplicates removed
duplicate conflict groups
FINAL DATABASE ROWS
```

Also inspect:

```text
rows by source
rows by category
random final sample
```

---

## Step 7 — final file

If every assertion passes:

```text
data/final/foods_final.csv
```

is the only CSV intended for database import.

---

# 41. Troubleshooting

## Error: `food.csv` cannot be found

Check the configured directory name.

Example problem:

```text
notebook expects:
data/raw/usda/sr_legacy/

actual:
data/raw/usda/sr-legacy/
```

Either rename the folder or update the notebook path.

---

## Error: FNDDS WWEIA category column cannot be found

Different USDA releases can use slightly different field names.

For example, one release may contain:

```text
wweia_food_category
wweia_food_category_description
```

instead of:

```text
wweia_food_category_code
```

The notebook's defensive `pick_column(...)` logic should include the actual field name from the downloaded release.

Always inspect:

```python
pd.read_csv(path, nrows=0).columns.tolist()
```

before guessing.

---

## Many rejected rows

Do not immediately replace missing nutrients with zero.

First inspect:

```text
usda_foods_rejected.csv
```

and identify the dominant rejection reasons.

The correct response may be:

```text
fix nutrient extraction
support another USDA energy field
exclude unsupported records
```

rather than inventing data.

---

## Many duplicate-name conflicts

This is not necessarily a pipeline failure.

It means multiple USDA data types report the same user-facing name with meaningfully different nutrition.

Inspect:

```text
data/final/audit/duplicate_conflicts.csv
```

The current pipeline still chooses a deterministic source winner.

---

# 42. Current intentional limitations

The first version of the catalog does not yet solve everything.

Current limitations include:

```text
English names only
no Arabic translations
no curated search aliases
coarse categories
no branded products
no barcode support
no restaurant foods
no image data
no user-friendly portion table
no country-specific regional food datasets
no automatic semantic shortening of technical USDA names
```

These are deliberate scope decisions.

The goal is to first establish:

```text
a trustworthy
repeatable
traceable
database-ready
generic food catalog
```

before expanding complexity.

---

# 43. Future improvements

Potential future phases:

## Better names

Build a curated display-name layer:

```text
USDA technical description
→ Spotter user-friendly English name
```

while preserving original source metadata.

---

## Search aliases

Populate aliases such as:

```json
[
  "chicken",
  "grilled chicken",
  "chicken breast"
]
```

to improve search without changing the canonical name.

---

## Arabic localization

Add reviewed:

```text
name_ar
```

and potentially Arabic aliases.

Do not use automatic translation as database truth without review.

---

## Portions

Add common UX conversions such as:

```text
1 large egg → 50g
1 slice bread → 30g
1 cup rice → 158g
```

while keeping nutrition canonical per 100g.

---

## Additional regional sources

Potentially normalize trusted regional food-composition databases into the same Food model.

The existing architecture already supports multiple sources through:

```text
source
external_source_id
source_version
```

---

## Branded foods

Add a separate ingestion strategy later for:

```text
brand
barcode
product name
manufacturer
serving size
```

Do not mix that complexity into the current generic food pipeline prematurely.

---

# 44. Key engineering decisions summarized

## Decision 1

**Use official USDA bulk datasets rather than relying on a runtime nutrition API.**

Reason:

```text
control
performance
search ownership
data consistency
reduced runtime dependency
```

---

## Decision 2

**Normalize external data to Spotter's schema.**

Reason:

The external source must not define application architecture.

---

## Decision 3

**Store nutrition per 100g.**

Reason:

One deterministic calculation model for gram-based food logging.

---

## Decision 4

**Do not silently turn missing nutrients into zero.**

Reason:

Missing data and zero nutrition are not equivalent.

---

## Decision 5

**Preserve source metadata.**

Reason:

Traceability, updates, debugging, and idempotent imports.

---

## Decision 6

**Keep raw, processed, and final data separate.**

Reason:

Reproducibility and safe iteration.

---

## Decision 7

**Do not blindly deduplicate by food name.**

Reason:

Some preparation states are meaningfully different.

---

## Decision 8

**When exact normalized names collide, use explicit source priority.**

Current priority:

```text
Foundation > FNDDS > SR Legacy
```

---

## Decision 9

**Audit nutritional disagreements instead of hiding them.**

Reason:

Deterministic production data and transparent data-quality review can coexist.

---

## Decision 10

**Keep mixed dishes out of `Food` by default.**

Reason:

Spotter has a separate `Recipe` model.

---

## Decision 11

**Archive catalog foods rather than destructively deleting them.**

Reason:

Recipes and historical meal records may reference them.

---

## Decision 12

**Keep the database importer separate from EDA notebooks.**

Reason:

Notebooks handle data analysis and transformation; application code handles database writes and deployment behavior.

---

# 45. Final mental model

The entire system can be understood as:

```text
                   EXTERNAL WORLD
                         │
                         ▼
                 USDA FoodData Central
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Foundation      SR Legacy         FNDDS
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                       RAW
                         │
                         ▼
                 NORMALIZATION
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
       accepted       rejected        duplicate
       candidates                       audit
          │
          ▼
                 FINALIZATION
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
      invalid drop    mixed drop     duplicate audit
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                  foods_final.csv
                         │
                         ▼
                  Prisma importer
                         │
                         ▼
                  PostgreSQL foods
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
       Meal logging               Recipes
```

---

# 46. Which file goes into the database?

The answer is intentionally simple:

```text
data/final/foods_final.csv
```

Not:

```text
usda_foods_stage.csv
```

Not:

```text
foods_for_db.csv
```

Not:

```text
usda_foods_rejected.csv
```

Not:

```text
usda_duplicate_name_candidates.csv
```

Not any file inside:

```text
data/final/audit/
```

Only:

```text
foods_final.csv
```

is the final catalog input for the Prisma import step.

---

# 47. Next engineering step

The data pipeline is complete when:

```text
foods_final.csv
```

has been generated and reviewed.

The next task belongs to the backend project:

> Build an idempotent Node.js + Prisma importer that reads `foods_final.csv` and upserts global Food rows using `(source, externalSourceId)`.

That importer should:

```text
parse CSV
validate fields
convert decimal strings safely
normalize NULL values
upsert records
report created / updated / skipped counts
fail loudly on malformed data
```

After that, Spotter can build:

```text
GET /api/foods
search
pagination
category filtering
custom user foods
meal logging
recipe construction
```

on top of a catalog it owns and understands.

---

## Final principle

The most important lesson from this pipeline is not the particular USDA files or pandas code.

It is this:

> **A production application should not treat an external dataset as its database schema.**

The external dataset is a source.

Spotter owns:

```text
the canonical model
the validation rules
the product taxonomy
the deduplication policy
the lifecycle
the API
the user experience
```

That is what turns downloaded data into a real product asset.

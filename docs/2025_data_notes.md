# 2025 Spotify Data – Excel Model & Business Rules

This document describes how the local workbook `spotify_sandbox_2025_Oct_15_v2.xlsx` is structured and how to interpret any 2025 metrics that come from it. This file is a **prototype / play space**, not the final 2025 report. A clean rebuild is planned once full-year Extended data is available in early 2026.

---

## Privacy & Repo Contents

- The actual Spotify Extended Streaming History export and full Excel model are **not** included in this repository for privacy reasons.
- This repo only contains:
  - De-identified or paste-value sample tables,
  - The schema and transformation logic,
  - Documentation of the business rules (e.g., album_family, Taylor’s Versions handling).
- The descriptions below refer to the **local Excel model** I work with on my machine.

---

## 1. Data Sources

**Primary source**

- **Spotify Extended Streaming History** export
- Coverage: **through 2025-10-15** (Spotify’s Extended export lags a bit behind real time)
- Loaded into Excel via Power Query

**Scope used in this model**

- All queries that feed 2025 reporting are filtered to **Year = 2025** based on the `ts` column.
- November–December 2025 listening is **not included** in this prototype.

There is also a manual lookup table for album families:

- Sheet: `album_family_map`
- Fields: `album_artist_name`, `album_album_name`, `album_family`, `album_key`

---

## 2. Query Architecture (ETL)

The workbook follows a simple fact + dimension pattern:

### 2.1 `fact_plays_2025`  (fact table)

- Source: Extended Streaming History (Excel table from the raw file)
- Key steps:
  - Parse `ts` as Date/Time
  - Add `Year` column from `endTime`
  - Filter `Year = 2025`
  - Add `album_key`:
    - `album_key = album_artist_name & " || " & album_album_name`
- Output: one row per play with fields including:
  - `endTime`
  - `track_name`
  - `album_artist_name`
  - `album_album_name`
  - `spotify_track_uri`
  - `msPlayed`
  - `album_key`
  - other helper fields from earlier modelling (e.g., `canonical_track`)

This is the **canonical 2025 fact table**.

### 2.2 `album_family_map`  (dimension / lookup)

- Source: manually maintained Excel table on the `album_family_map` sheet
- Purpose: define **album-era families** so that multiple editions of the same record can be collapsed.
- Example mappings:
  - Ozzy Osbourne:
    - `Diary Of A Madman`  
      `Diary of a Madman (40th Anniversary Expanded Edition)` → **Diary Of A Madman**
    - `No More Tears (30th Anniversary Expanded Edition)`  
      `No More Tears (Expanded Edition)` → **No More Tears**
    - `Bark At The Moon (Expanded Edition)` → **Bark At The Moon**
  - Taylor Swift:
    - `Red`, `Red (Taylor’s Version)` → **Red**
    - `Fearless`, `Fearless (Platinum Edition)`, `Fearless (Taylor’s Version)` → **Fearless**
    - Same logic for other Taylor’s Version albums as needed.
- `album_key` is computed the same way as in the fact table:
  - `album_key = album_artist_name & " || " & album_album_name`

This table is edited by hand as new messy albums are discovered.

### 2.3 `fact_plays_2025_with_album_family`  (enriched fact table)

- Built via a **Left Outer join** from `fact_plays_2025` to `album_family_map` on `album_key`.
- Steps:
  1. Merge `fact_plays_2025` (left) with `album_family_map` (right) on `album_key`.
  2. Expand only the `album_family` column from the map.
  3. Add `album_family_final`:

     ```m
     album_family_final =
       if [album_family] = null
       then [album_album_name]
       else [album_family]
     ```

  4. Drop the temporary `album_family` column and rename `album_family_final` → `album_family`.

- Result: every play row has:
  - `album_family`: clean album-era name
    - Ozzy expanded editions are collapsed
    - Taylor’s Versions are collapsed into their original album eras
    - Albums not yet in the map default to their raw `album_album_name`

This is the table used as the data source for 2025 pivots (Top Albums, etc.).

---

## 3. Measures & Pivot Logic

### 3.1 Base measure

- **`msPlayed`** is the core measure from Spotify (milliseconds streamed).

### 3.2 Derived measure in pivots

Inside PivotTables, a calculated field is defined:

- **Name:** `Minutes`
- **Definition:** `= msPlayed / 60000`
- **Purpose:** human-readable listening time per album/artist.

This field **only lives in the PivotTable layer** on purpose; it is *not* stored in the fact table.

---

## 4. Business Rules & Modeling Decisions

These rules explain how to interpret the numbers:

### 4.1 Album families

- The unit for “Top Albums 2025” is the **album family**, not the exact edition.
- Editions that are collapsed:
  - Anniversary, expanded, legacy, and special editions of the same core album.
  - Taylor’s Version albums are **explicitly collapsed** with their original versions:
    - e.g., all versions of *Red* (including *Red (Taylor’s Version)*) are reported as a single album family **Red**.
- Albums not present in `album_family_map` are reported under their original `album_album_name`.

### 4.2 Artist interpretation

- Some albums may appear with variant album-artist metadata (e.g., `Blizzard Of Ozz` under **Randy Rhoads** in some editions).
- For reporting and narrative, these are treated as belonging to the **primary canonical artist** (e.g., Ozzy Osbourne).
- A more formal `artist_family` dimension may be added later; for now this is handled interpretively when describing results.

### 4.3 Inclusions / exclusions

- All 2025 plays in Extended history up to 2025-10-15 are included.
- No exclusions for skips, short plays, or partial tracks beyond whatever Spotify already enforces for `msPlayed` in the export.
- Rows with missing album information (`album_album_name` blank) may appear as `(blank)` in pivots and are treated as “unknown / uncategorized content.”

---

## 5. Current Limitations (Prototype Status)

- Extended Streaming History stops at **2025-10-15**:
  - November and December listening are not yet included.
- Album families are **partial**:
  - Ozzy Osbourne and Taylor Swift have explicit mappings.
  - Other artists are only mapped if needed for top-album clarity.
- No Spotify API enrichment yet:
  - No album IDs, artist IDs, or market-specific metadata.
  - All modeling uses the text fields from the export.
- No song-family layer yet:
  - Plays are still aggregated by `canonical_track` or `spotify_track_uri`.
  - Future work will add `song_family` (e.g., grouping studio + live versions of “Morning Elvis”).

---

## 6. Planned January 2026 Rebuild

Once a new Extended Streaming History export is available in early 2026:

1. Download full-year 2025 Extended data.
2. Rebuild `fact_plays_2025` from scratch using the full file.
3. Reapply and expand:
   - `album_family_map`
   - (optional) `artist_family_map`
   - new `song_family_map`
4. Rebuild `fact_plays_2025_with_album_family` and all 2025 summary pivots.
5. Optionally port this logic into a Python + pandas pipeline and/or a lightweight web or dashboard layer.

The **private Excel model (not stored in this repo)** is the authoritative prototype that defines the business rules and schema for the future 2025 rebuild.
This document captures the structure and logic of that model without exposing personal streaming data.

---
name: weather-compare
description: Compare the weather forecast for two UK places by name. Use this skill whenever the user wants to compare weather between two UK locations, asks which place has better weather, wants to know where would be nicer to visit, or mentions comparing forecasts for UK towns or villages. Trigger even if the user doesn't say "weather-compare" explicitly — any request like "which has better weather, X or Y?" or "is it sunnier in X than Y?" should use this skill.
---

# Weather Compare

Compare the weather forecast for two UK places and recommend which has better conditions.

## Steps

### 1. Get the two place names

If the user hasn't provided two place names, ask for them now. You need exactly two UK place names (e.g. "Polkerris" and "Polzeath").

### 2. Check API key

Verify `$METEOBLUE_API_KEY` is set:

```bash
echo ${METEOBLUE_API_KEY:?}
```

If it's empty or unset, stop and tell the user: "Please set the `METEOBLUE_API_KEY` environment variable. You can get a key at https://www.meteoblue.com/en/weather-api."

### 3. Ensure GBPN.csv is available

The place name database is cached as `GBPN.csv` in the same directory as this SKILL.md file. You know the path to this SKILL.md — use its directory as `SKILL_DIR`.

If `$SKILL_DIR/GBPN.csv` doesn't exist, download and unzip it:

```bash
curl -L https://gazetteer.org.uk/GBPN.zip -o $SKILL_DIR/GBPN.zip
unzip -o $SKILL_DIR/GBPN.zip -d $SKILL_DIR/
rm $SKILL_DIR/GBPN.zip
```

Only download if the file is missing — no need to re-fetch on every use.

### 4. Look up coordinates

For each place, run:

```bash
duckdb -c "SELECT Latitude, Longitude FROM read_csv('$SKILL_DIR/GBPN.csv') WHERE \"Place Name\" = 'PLACE_NAME';"
```

If a place is not found (empty result), tell the user and ask them to check the spelling or try a nearby town. Do not proceed until both places are found.

### 5. Fetch weather forecasts

For each place, fetch the Meteoblue basic-day forecast using `Europe/London` timezone so dates align with the user's local day:

```bash
curl -s "https://my.meteoblue.com/packages/basic-day?name=PLACE_NAME&lat=LAT&lon=LON&tz=Europe%2FLondon&apikey=$METEOBLUE_API_KEY"
```

### 6. Surf meteogram (coastal places only)

For each place, decide whether it is a coastal location — use your general knowledge (e.g. Polzeath is a surf beach, Polkerris is a harbour village). If uncertain, lean toward including it.

For any coastal place, download the surf meteogram and display it inline using the Read tool:

```bash
curl -s "https://my.meteoblue.com/images/meteogram_surf?lat=LAT&lon=LON&location_name=PLACE_NAME&apikey=$METEOBLUE_API_KEY" -o /tmp/PLACENAME_surf.png
```

Then use the Read tool on `/tmp/PLACENAME_surf.png` to display it inline.

### 7. Compare and recommend

Detect the user's time intent (e.g. "today", "this weekend", "next week"). If unspecified, compare across the next 3–7 days. Otherwise focus on the relevant day(s) only.

Compare across these dimensions:

- **Rain**: `precipitation_probability` (%) and `precipitation` (mm) and `precipitation_hours`
- **Felt temperature**: use `felttemperature_max`, `felttemperature_min`, and `felttemperature_mean` (accounts for wind chill — more useful than raw temperature, especially at the coast)
- **Wind**: `windspeed_mean` and `windspeed_max` (m/s)
- **UV index**: `uvindex` (higher = more sunshine)
- **Weather symbol**: `pictocode` (1=sunny, higher = worse conditions)

Give an honest verdict:
- Name the winner clearly
- Explain the key differences (e.g. "Polzeath has half the rainfall and higher UV")
- If it's close or depends on what you're doing (hiking vs. beach), say so
- Mention the specific date(s) or forecast window you're comparing

Keep the response concise and practical — the user is trying to decide where to go.

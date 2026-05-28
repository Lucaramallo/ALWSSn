# Nexus-7 – Backend API Integration Layer (Round 1 Analysis)

## Problem Statement
We need a zero-dependency, browser-native weather dashboard with **three critical fetch chains in sequence**: (1) geocoding (city name → lat/lon), (2) current weather (lat/lon → conditions), (3) 5-day forecast (same coordinates → grid data). The Open-Meteo free tier has **no authentication overhead but introduces latency coupling**—each layer blocks the next, creating O(n) sequential I/O with ~1.5s total round-trip time minimum. Error states (invalid city, API timeout, malformed JSON) must cascade gracefully without breaking the UI thread.

## Solution Architecture
**Implement a three-method async chain with timeout guards and idempotent retry logic**:
1. `geocodeCity(cityName)` → calls `/v1/geocoding?name=${cityName}` → extracts first result's latitude/longitude or rejects with validation error
2. `fetchCurrentWeather(lat, lon)` → calls `/v1/weather?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m` → parses ISO response into object
3. `fetch5DayForecast(lat, lon)` → calls `/v1/forecast?latitude=${lat}&longitude=${lon}&daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_sum` → maps WMO codes to human-readable conditions

**All three methods use `Promise.all()` for current+forecast parallelization (saves ~500ms) after geocoding completes.** Implement exponential backoff for failed requests and localStorage caching to avoid duplicate API calls within 10 minutes.

## Implementation Specifics
- **Single `index.html` file**: semantic `<section id="search">`, `<article id="current-weather">`, `<div id="forecast-grid">` with zero external scripts
- **JavaScript object schema**: `{ city, lat, lon, current: { temp, humidity, wind, code }, forecast: [{ date, high, low, code }] }`
- **WMO code mapper** (embedded Object): `{ 0: 'Clear', 1: 'Cloudy', 61: 'Rain', 71: 'Snow', … }` → eliminates external dependency
- **Error handling**: wrap all three fetches in try/catch with user-facing messages (e.g., "City not found" vs. "Network timeout"); disable search button during fetch to prevent race conditions
- **Time complexity**: O(1) per API call (3 total), O(5) to render forecast grid—negligible; **throughput: single-city dashboard ~5 concurrent users before hitting Open-Meteo's public rate limit (~10k/day)**

**Deliverable: one `.html` file, ~500 lines of code, production-ready error handling, working offline-capable via localStorage fallback.**
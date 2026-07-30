# Veather Development Notes

**Developer**: Primary user is red-green colorblind  
**Development Date**: 2026-07-30  
**Status**: v1.1 Production (post-deployment refinements)

---

## Key Design Decisions

### Color Palette (Critical)
- **User is red-green colorblind** - this drove all color choices
- Avoided green entirely - used **cyan/teal** (#44dddd) for 60-69°F range instead
- Tested 3 palettes in color-palette.html, chose "Option 2: More Contrast"
- 10 distinct temperature ranges with high contrast between adjacent colors
- Amber/orange spectrum for hot (not pure red)
- Blue gradient for cold with strong saturation differences
- White for extreme cold (<20°F) - high visibility on dark background

### Radar Integration
- **Chose RainViewer** over NWS static images or iframe embeds
- Reasoning: 
  - Free, no API key
  - Works seamlessly with location search
  - Animated radar (better UX than static)
  - Easy to recenter map when location changes
- Limitations: Radar tiles only exist to zoom level 6 (scaled beyond that)
- Configuration: maxNativeZoom: 6, maxZoom: 10

### Feels-Like Temperature
- **Only shows when ±3°F different** from actual temp
- Inline format "85°/92°" saves horizontal space in hourly grid
- Uses NWS heatIndex or windChill (only provided when relevant)
- Decision: Avoid clutter for negligible differences

### Hourly Column Toggles
- **AMT and UV hidden by default** to reduce screen clutter
- User can toggle any column except Temp (always visible)
- Preferences saved to localStorage
- Mobile consideration: fewer columns = better readability

### Saved Locations
- **Max 5 locations** to avoid UI clutter
- Moved to dedicated "Locations" tab (not dropdown) per user request
- Prevents scrolling past location list to see weather
- Custom labels for user personalization ("Home", "Work", etc.)

### Mobile Optimization
- **Primary use case is mobile** - drove all sizing decisions
- 44x44px minimum touch targets (Apple/Android guideline)
- Larger fonts across the board (1.8em title, 3.5em temp)
- Increased padding and gaps (25px cards, 20px spacing)
- Responsive breakpoints at 450px & 500px
- Tabs emoji-only on all screens:
  - Dynamic weather icon for Forecast tab (updates with conditions)
  - 🕐 Hourly, 🗺️ Radar, ⭐ Locations
  - 1.5em font size, always visible (no text labels)
  - Eliminated overflow issues completely
- Wind display compact: "15⬇️" (no space between)
- Time column 70px wide (fits "12:00 PM" without wrapping)
- Column toggles scroll horizontally on mobile
- Radar map reduced to 400px height (less scrolling)
- Current conditions stats enlarged: 2em values, 1.2em labels, centered layout
- Current stats use vertical layout: emoji (2.5em) → label → value

### Weather Idioms (Header)
- **Rotating contextual phrases** displayed in header
- Based on current conditions (not random)
- Examples: "Walkin on sunshine", "Storms a-brewin", "Hot, humid, might rain"
- Priority order: severe → precip → temp → humidity → wind → clouds → clear
- Multiple applicable idioms rotate every 4 seconds
- Special combo: temp≥90°F + precip≥20% + humidity≥70% = "Hot, humid, might rain"
- Day/night aware for clear/sunny conditions and defaults
- Timer cleared on location change to prevent memory leaks

### Current Conditions Data Source
- **Uses first hourly forecast point** (most accurate "right now" data)
- Applies to: temperature, precipitation, wind, humidity
- Shows actual current conditions separate from 12-hour period forecast
- Current 12-hour period box follows current conditions (e.g., "This Afternoon", "Tonight")
- Reasoning: Hourly data more accurate for current moment than 12-hour average
- Precipitation display simplified: single value (not avg [min-max] format)
- Forecast periods: NWS value if available, else max from hourly data

### Dynamic Weather Emojis
- **Precipitation emoji changes based on probability and type**:
  - 🌂 (closed umbrella) when <10% chance
  - ⛈️ thunderstorms (not just ⚡)
  - ❄️ snow, 🧊 ice, ☔ rain, etc.
- **Wind emoji adapts to speed**:
  - 💨 normal (<15 mph)
  - 🌬️ windy face (≥15 mph)
  - User testing showed 15 mph feels "windy" in Hyattsville area
- **Humidity emoji reflects conditions**:
  - 🌵 dry (<40%) - rare in humid MD climate
  - 💧 normal (40-70%)
  - 💦 humid (>70%)
  - Thresholds chosen for local climate patterns

### Location Search
- **Nominatim (OpenStreetMap)** for geocoding
- Free, no API key, good US coverage
- Accepts: ZIP, "City, ST", or city name
- Default location: Hyattsville, MD (38.946974, -76.932044)
- Auto-saved as first saved location

---

## Technical Decisions

### Architecture
- **Single HTML file** - no build process, easy deployment
- All-in-one approach (HTML/CSS/JS) for simplicity
- CDN dependencies (Leaflet) - no local files needed
- localStorage for persistence (locations, column prefs)

### NWS API Integration
- **No caching** of weather data (always fresh on refresh)
- 30-minute auto-refresh interval
- Parallel requests: forecast + hourly + alerts
- Hourly data used to calculate avg/min/max for forecast periods

### State Management
- Global variables (not React/Vue needed for this scale)
- No complex state library needed
- localStorage for persistence only
- Radar state separate (map, frames, animation)

---

## Known Issues & Gotchas

### Radar
- Tiles only exist to zoom 6 (RainViewer limitation)
- Beyond zoom 6, tiles are scaled (pixelated but functional)
- Animation uses setInterval (500ms), not requestAnimationFrame
- Radar data not auto-refreshed (only on tab activation)

### Weather Idioms
- Default fallback must check isDaytime (fixed: was always "Perfect for stargazing")
- Clear/sunny check must exclude "partly" and "mostly" to avoid overlaps
- Cloudy check must exclude "partly" to avoid overlap with "partly cloudy" idiom (fixed)
- Timer must be cleared on location change to prevent memory leaks
- Multiple idioms rotate every 4 seconds (not immediately obvious from code)

### Dynamic Emojis
- Precipitation emoji needs probability value passed to function (not just forecast object)
- Wind emoji calculated from numeric speed (parseInt extracts mph value)
- Humidity emoji needs numeric value, not display string (extract before formatting)
- All three emoji functions must be called for current conditions, current period, and all forecast periods

### Geolocation
- Requires HTTPS in production browsers
- iOS Safari requires user gesture (can't auto-request)
- Some browsers block on file:// protocol

### NWS API
- US/territories only (no international support)
- Occasionally returns 503 (service unavailable)
- No official rate limit, but be reasonable
- Grid system can be confusing (uses office codes like "LWX")

### Nominatim
- Rate limit ~1 request/second
- "Portland" returns Oregon (ambiguous without state)
- International results possible (app filters to US only)

### Browser Compatibility
- Requires ES6+ (async/await)
- localStorage required for saved locations
- Geolocation optional (falls back to search)
- Tested: Chrome, Firefox, Safari, Edge (desktop + mobile)

---

## User Feedback Integration

### Changes Made During Development

1. **Feels-like column removed from hourly tab**
   - User: "they're all the same"
   - Solution: Conditional inline display only when different by 3°F+

2. **Saved locations moved to tab**
   - User: "don't want to scroll past it"
   - Solution: Dedicated "Locations" tab instead of dropdown

3. **Font sizes increased**
   - Implicit need for mobile optimization
   - Solution: Comprehensive size increase (temps, buttons, text)

4. **Column toggles added**
   - Solution to horizontal space constraints
   - AMT & UV hidden by default per user preference

5. **Color palette iteration**
   - Tested 3 options in color-palette.html
   - User chose "Option 2: More Contrast"
   - Critical for colorblind accessibility

6. **Emoji consistency**
   - Added emojis to hourly headers to match forecast tab
   - Labels in forecast current conditions: "☔ PRECIP", "💨 WIND", "💦 HUMID"

7. **Precipitation display simplified**
   - Original: "NWS% (avg% [min-max])" was too complex
   - Solution: Show single value (NWS or max from hourly)
   - Current conditions: Use first hourly data point

8. **Weather idioms added**
   - Fill blank space in header with contextual phrases
   - User wanted "decorative" content, not duplicate weather data
   - Chose weather-matched idioms over static emoji row
   - Rotates every 4 seconds when multiple apply

9. **Mobile tabs overflow**
   - Initial fix at 400px breakpoint wasn't enough
   - Solution: 500px breakpoint, smaller padding/font, reduced gap
   - Allows horizontal scroll if still too wide

10. **Wind display wrapping**
    - Two-digit wind speeds forced emoji to second line
    - Solution: Remove space between number and emoji ("15⬇️")

11. **Time column overflow risk**
    - "12:00 PM" potentially wider than 60px column
    - Solution: Increased to 70px, added white-space: nowrap

12. **Tab buttons still overflowing on mobile**
    - 500px breakpoint with reduced padding/font still not enough
    - Solution: Switch to emoji-only tabs on all screen sizes
    - No responsive CSS complexity, just always show icons
    - Dynamic forecast icon updates with current weather

13. **Current weather stats too small**
    - User requested larger font sizes
    - Solution: Increased values to 2em, labels to 1.2em, lightened label color to #999
    - Centered layout with emoji (2.5em) on top, label, then value

14. **Current temp showing wrong value**
    - Displayed 69°F when actual temp was 80s
    - Bug: Used 12-hour period low instead of actual current temp
    - Solution: Pull current temp from first hourly data point like other stats
    - Added separate box for current 12-hour period forecast

15. **Weather idioms overlapping**
    - "Partly cloudy" also triggered "Silver linings abound" (cloudy idiom)
    - Solution: Exclude "partly" from cloudy/overcast check

16. **Emoji improvements requested**
    - Thunder: Changed ⚡ to ⛈️ (full thunderstorm emoji)
    - Wind: Added 🌬️ windy face for 15+ mph (originally 25, adjusted based on user feel)
    - Humidity: Dynamic 🌵/💧/💦 based on <40%/40-70%/>70%
    - Precip: 🌂 closed umbrella when <10% chance
    - Applied to current conditions, current period, and all forecast periods

17. **Current weather stats label position**
    - User wanted labels before emojis for better readability
    - Solution: Changed from "[emoji] LABEL" to "LABEL [emoji]"
    - Maintains vertical layout (label → emoji → value)

18. **No-data emoji handling**
    - Thursday forecast showed cactus (🌵) with "N/A" humidity
    - Bug: Days beyond hourly data returned 0, triggered dry emoji
    - Solution: Added 🤷 shrug emoji for all no-data cases
    - Applied to precipitation, wind, and humidity emojis

---

## File Structure Rationale

```
veather/
├── veather.html          # Main app (production)
├── manifest.json         # PWA manifest
├── README.md             # User documentation
├── SPECS.md              # Technical deep-dive
├── DEVELOPMENT.md        # This file (context & decisions)
└── color-palette.html    # Dev tool (not for production)
```

**Why separate SPECS.md from README.md?**
- README: User-focused (how to use, deploy, troubleshoot)
- SPECS: Developer-focused (APIs, algorithms, data models)
- Keeps each document focused and scannable

---

## Future Development Notes

### Before Adding Features
1. Test on mobile first (primary use case)
2. Consider colorblind accessibility
3. Keep it simple (resist feature creep)
4. localStorage has size limits (~5-10MB)
5. NWS API is US-only (international = different API)

### If Refactoring
- Consider service worker for offline support
- IndexedDB if localStorage too small
- Web Workers for heavy processing
- Virtual scrolling if hourly list grows

### If Adding Location Features
- Max 5 locations is intentional (UI constraint)
- Don't auto-save every search (clutters list)
- Default location (Hyattsville) should always exist

### If Changing Colors
- Test with colorblind simulator
- Avoid red/green pairs
- Maintain high contrast between adjacent ranges
- Document why in this file

---

## Testing Checklist for Future Changes

**Must test:**
- [ ] Mobile Safari (iOS)
- [ ] Chrome Android
- [ ] Desktop Chrome/Firefox
- [ ] Location search (ZIP, city, geolocation)
- [ ] Saved locations (save, edit, delete, load)
- [ ] All 4 tabs render correctly (emoji-only on all screens)
- [ ] Forecast tab icon updates with current weather
- [ ] Column toggles work and scroll on mobile
- [ ] Radar animation plays/pauses
- [ ] Temperature colors span full range
- [ ] Feels-like conditional display works
- [ ] Weather idioms rotate (multiple conditions)
- [ ] Weather idioms day/night aware (clear day vs night)
- [ ] Weather idioms don't overlap (partly cloudy ≠ cloudy)
- [ ] Tabs always show emojis (no overflow on any screen size)
- [ ] Wind display compact (no line wrapping)
- [ ] Precipitation shows simplified format
- [ ] Current temp uses hourly data (not 12-hour period low)
- [ ] Current 12-hour period box shows after current conditions
- [ ] Dynamic emojis: 🌂 <10% precip, 🌬️ 15+ mph wind, 🌵/💧/💦 humidity
- [ ] Current weather stats show label before emoji (PRECIP [emoji])
- [ ] No-data emoji (🤷) shows for missing precip/wind/humidity data

**Known working locations for testing:**
- 20782 (ZIP - Hyattsville, MD)
- Seattle, WA (City/State)
- Portland (ambiguous - returns Portland, OR)
- Miami, FL (tropical - tests heat index)
- Anchorage, AK (cold - tests wind chill)

---

## Dependencies & Versions

**CDN Resources:**
- Leaflet v1.9.4 (map library)
- OpenStreetMap tiles (base map)

**APIs:**
- NWS Weather API (no versioning, public)
- RainViewer API (no versioning, free)
- Nominatim API (no versioning, OSM)

**No package.json** - all dependencies via CDN  
**No build process** - vanilla HTML/CSS/JS

---

## Deployment Notes

### Where It Works
- GitHub Pages ✅
- Netlify ✅
- Vercel ✅
- Any static file server ✅

### Requirements
- HTTPS (for geolocation)
- No backend needed
- No environment variables
- No build step

### What to Deploy
- veather.html (required)
- manifest.json (required for PWA)
- README.md (optional, for repo)
- SPECS.md (optional, for developers)
- DEVELOPMENT.md (optional, for context)

**Do NOT deploy:**
- color-palette.html (dev tool only)

---

## Auto-Compact Recovery Info

**If conversation context is lost, start here:**

1. **User Profile**: Primary user is red-green colorblind - drives all color decisions
2. **Architecture**: Single HTML file, no frameworks, mobile-first
3. **Key Features**: Location search, saved locations (max 5), radar (RainViewer), colorblind palette, weather idioms, dynamic emojis
4. **Critical Files**: veather.html (main), SPECS.md (technical), this file (context)
5. **Mobile Focus**: Touch targets 44px+, large fonts, emoji-only tabs (always visible, no text)
6. **Tabs**: Dynamic forecast icon (updates with weather), 🕐 Hourly, 🗺️ Radar, ⭐ Locations - 1.5em, no responsive hiding
7. **Radar Config**: maxNativeZoom: 6, maxZoom: 10 (RainViewer limitation)
8. **Color Palette**: Option 2 from color-palette.html (cyan not green, high contrast)
9. **Feels-Like**: Only shown when ±3°F different (inline format to save space)
10. **Weather Idioms**: Rotate every 4s in header, contextual to conditions, day/night aware, no overlap (partly ≠ cloudy)
11. **Current Conditions**: Use first hourly data point (temp, precip, wind, humidity) for accuracy
12. **Current Period Box**: Shows current 12-hour forecast period after current conditions
13. **Precipitation**: Simplified display - single value, not range format
14. **Dynamic Emojis**: 🌂 <10% precip, ⛈️ thunder, 🌬️ 15+ mph wind, 🌵/💧/💦 humidity (<40%/40-70%/>70%), 🤷 no data
15. **Current Stats**: Vertical layout with label (1.2em, #999) → emoji (2.5em) → value (2em), centered

**Quick Reference Commands:**
```bash
# View app
open veather/veather.html

# Test colors
open veather/color-palette.html

# Read specs
cat veather/SPECS.md

# Read this file
cat veather/DEVELOPMENT.md
```

---

**End of Development Notes**

# Veather Technical Specifications

**Version**: 1.1  
**Last Updated**: 2026-07-30  
**Status**: Production

---

## Architecture Overview

### Single-Page Application (SPA)
- Monolithic HTML file with embedded CSS and JavaScript
- No build process or bundling required
- All dependencies loaded from CDN
- Client-side only (no backend)

### Core Technologies
- **HTML5** - Semantic markup, meta tags for PWA
- **CSS3** - Grid/Flexbox layouts, custom properties, media queries
- **Vanilla JavaScript (ES6+)** - Async/await, fetch, localStorage
- **Leaflet.js** - Map rendering and interaction

### Application Flow
```
Page Load
  ↓
Initialize Preferences (localStorage)
  ↓
Load Default Location (Hyattsville, MD)
  ↓
Fetch Weather Data (NWS API)
  ↓
Render UI (Forecast/Hourly/Radar/Locations)
  ↓
User Interactions → Update Location → Refresh Data
```

---

## Data Models

### Location Object
```javascript
{
  lat: Number,          // Latitude (-90 to 90)
  lon: Number,          // Longitude (-180 to 180)
  name: String,         // Display name from geocoding (e.g., "Seattle, WA")
  label: String | null  // Optional custom label (e.g., "Home", "Work")
}
```

**Storage**: `localStorage` key `veather-locations` as JSON array  
**Max Size**: 5 locations  
**Uniqueness**: Checked by lat/lon within 0.01 degree tolerance

### Weather Data Structure
```javascript
currentData = {
  location: {
    city: String,
    state: String
  },
  forecast: [
    {
      number: Number,
      name: String,              // "Tonight", "Thursday", etc.
      startTime: ISO8601,
      endTime: ISO8601,
      isDaytime: Boolean,
      temperature: Number,       // Integer
      temperatureUnit: "F",
      temperatureTrend: String | null,
      windSpeed: String,         // "5 to 10 mph"
      windDirection: String,     // "N", "NE", "E", etc.
      icon: String,              // URL (not used)
      shortForecast: String,
      detailedForecast: String,
      probabilityOfPrecipitation: {
        unitCode: String,
        value: Number | null     // 0-100
      },
      relativeHumidity: {
        unitCode: String,
        value: Number | null     // 0-100
      }
    }
    // ... 13 more periods (14 total: 7 days × day/night)
  ],
  hourly: [
    {
      startTime: ISO8601,
      endTime: ISO8601,
      isDaytime: Boolean,
      temperature: Number,
      temperatureUnit: "F",
      windSpeed: String,
      windDirection: String,
      icon: String,
      shortForecast: String,
      probabilityOfPrecipitation: { value: Number },
      relativeHumidity: { value: Number },
      heatIndex: {               // Only present when applicable
        unitCode: String,
        value: Number
      },
      windChill: {               // Only present when applicable
        unitCode: String,
        value: Number
      },
      quantitativePrecipitation: { value: Number },  // inches
      snowfallAmount: { value: Number },             // inches
      iceAccumulation: { value: Number },            // inches
      uvIndex: Number                                // 0-11+
    }
    // ... 155 more hours (156 total: ~6.5 days)
  ],
  alerts: [
    {
      properties: {
        event: String,           // "Severe Thunderstorm Warning"
        headline: String,
        description: String,
        severity: String,        // "Severe", "Moderate", "Minor"
        urgency: String,
        onset: ISO8601,
        expires: ISO8601
      }
    }
  ]
}
```

### Column Preferences
```javascript
COLUMNS = {
  temp: { label: 'Temp', visible: true, alwaysOn: true },
  precip: { label: 'Precip', visible: true },
  amt: { label: 'Amt', visible: false },
  wind: { label: 'Wind', visible: true },
  humid: { label: 'Humid', visible: true },
  uv: { label: 'UV', visible: false }
}
```

**Storage**: `localStorage` key `veather-columns` as JSON object  
**Updates**: On checkbox toggle, saved immediately

### Radar Frame Data
```javascript
radarFrames = [
  {
    path: String,    // "/v2/radar/1234567890/256"
    time: Number     // Unix timestamp (seconds)
  }
  // ... typically 10-15 frames
]
```

**Source**: RainViewer API `weather-maps.json`  
**Refresh**: On radar tab activation (not cached)

---

## API Integration

### NWS Weather API

**Base URL**: `https://api.weather.gov`

#### 1. Points Endpoint
```
GET /points/{latitude},{longitude}
```

**Response** (relevant fields):
```json
{
  "properties": {
    "forecast": "https://api.weather.gov/gridpoints/LWX/97,71/forecast",
    "forecastHourly": "https://api.weather.gov/gridpoints/LWX/97,71/forecast/hourly",
    "forecastZone": "https://api.weather.gov/zones/forecast/MDZ013",
    "relativeLocation": {
      "properties": {
        "city": "Hyattsville",
        "state": "MD"
      }
    }
  }
}
```

**Usage**: Initial call to get forecast URLs  
**Rate Limit**: No documented limit  
**Error Handling**: 404 = coordinates outside US/territories

#### 2. Forecast Endpoint
```
GET /gridpoints/{office}/{gridX},{gridY}/forecast
```

**Response**: See "Weather Data Structure" above  
**Usage**: 7-day day/night forecast  
**Caching**: Client caches until manual refresh

#### 3. Hourly Forecast Endpoint
```
GET /gridpoints/{office}/{gridX},{gridY}/forecast/hourly
```

**Response**: Similar to forecast but 156 hourly periods  
**Usage**:
- **Current conditions**: First hourly data point (most accurate "right now")
- **Forecast periods**: Aggregated stats (avg, min, max) for 12-hour windows
- **Hourly tab**: First 24 hours displayed with charts

**Data Points**: temp, wind, precip, humidity, heat index, wind chill, UV

#### 4. Alerts Endpoint
```
GET /alerts/active/zone/{zoneId}
```

**Response**: Array of active alerts  
**Usage**: Display severe weather warnings  
**Polling**: Only on manual refresh (not auto-polled)

**Headers Sent**:
```
User-Agent: Veather/1.0 (weather web app)
```

### RainViewer API

**Base URL**: `https://api.rainviewer.com/public`

#### Weather Maps Metadata
```
GET /weather-maps.json
```

**Response**:
```json
{
  "radar": {
    "past": [
      { "path": "/v2/radar/1234567890/256", "time": 1234567890 }
    ],
    "nowcast": [
      { "path": "/v2/radar/1234567891/256", "time": 1234567891 }
    ]
  }
}
```

**Usage**: Get available radar frame timestamps  
**Refresh**: Each time radar tab is opened

#### Radar Tiles
```
GET https://tilecache.rainviewer.com{path}/256/{z}/{x}/{y}/2/1_1.png
```

**Parameters**:
- `{path}`: From weather-maps.json (e.g., `/v2/radar/1234567890/256`)
- `{z}`: Zoom level (4-10, native only to 6)
- `{x}/{y}`: Tile coordinates
- `2`: Color scheme (2 = original)
- `1_1`: Smooth/snow settings

**Format**: PNG with transparency  
**Caching**: Handled by Leaflet tile layer

### Nominatim Geocoding API

**Base URL**: `https://nominatim.openstreetmap.org`

#### Forward Geocoding (Search)
```
GET /search?q={query}&format=json&limit=1&countrycodes=us
```

**ZIP Code Query**:
```
GET /search?postalcode={zip}&country=US&format=json&limit=1
```

**Response**:
```json
[
  {
    "lat": "38.9469740",
    "lon": "-76.9320440",
    "display_name": "Hyattsville, Prince George's County, Maryland, ...",
    "type": "city",
    "importance": 0.5
  }
]
```

**Rate Limit**: ~1 request/second (enforced by server)  
**Error Handling**: Empty array = no results

#### Reverse Geocoding
```
GET /reverse?lat={lat}&lon={lon}&format=json
```

**Response**: Similar to forward geocoding  
**Usage**: Get location name from coordinates (for "Use My Location")

---

## Algorithms & Logic

### Temperature Color Coding

**Function**: `getTempColor(tempF)`

**Palette** (Colorblind-Friendly):
```javascript
if (tempF >= 95) return '#aa3300';  // Dark burnt orange
if (tempF >= 90) return '#ff5500';  // Red-orange
if (tempF >= 80) return '#ffaa00';  // Amber
if (tempF >= 70) return '#ffdd77';  // Peach/tan
if (tempF >= 60) return '#44dddd';  // Bright cyan
if (tempF >= 50) return '#5599ff';  // Sky blue
if (tempF >= 40) return '#3366cc';  // Royal blue
if (tempF >= 30) return '#aaccff';  // Pale blue
if (tempF >= 20) return '#ddeeff';  // Ice blue
return '#ffffff';                   // White
```

**Design Notes**:
- Avoids red/green confusion (uses cyan instead of green)
- High contrast between adjacent ranges
- Warmer colors (amber/orange) for hot temps
- Cooler colors (blue spectrum) for cold temps
- White for extreme cold (high visibility on dark background)

### Feels-Like Temperature

**Logic**:
```javascript
// Check heatIndex first (hot conditions)
if (hour.heatIndex?.value != null) {
  feelsLike = hour.heatIndex.value;
  // Convert if needed (Celsius → Fahrenheit)
}
// Check windChill (cold conditions)
else if (hour.windChill?.value != null) {
  feelsLike = hour.windChill.value;
}
// Otherwise, no feels-like available
else {
  feelsLike = null;
}

// Display only if significantly different (3°F+)
if (feelsLike != null && Math.abs(feelsLikeF - actualTempF) >= 3) {
  display = `${actualTemp}°/${feelsLike}°`;
} else {
  display = `${actualTemp}°`;
}
```

**Reasoning**:
- NWS only provides heat index/wind chill when relevant
- 3°F threshold avoids clutter for negligible differences
- Inline format saves horizontal space in hourly grid

### Humidity & Precipitation Aggregation

**Function**: `getHumidityStats(startTime, endTime)`

**Algorithm**:
```javascript
1. Filter hourly data to hours within [startTime, endTime)
2. Extract all non-null humidity values
3. Calculate:
   - avg = Math.round(sum / count)
   - min = Math.min(...values)
   - max = Math.max(...values)
4. Return { avg, min, max } or null if no data
```

**Display Format**:
- **Forecast periods**: `avg% [min-max]` (e.g., `65% [58-72]`)
- **Current conditions**: Single value from first hourly data point (e.g., `65%`)

**Function**: `getPrecipitationStats(startTime, endTime)`

**Algorithm**: Same as humidity stats (avg, min, max from hourly data)

**Display Logic**:
```javascript
// Current conditions: use first hour
if (currentData.hourly[0].probabilityOfPrecipitation?.value != null) {
  precipDisplay = `${currentData.hourly[0].probabilityOfPrecipitation.value}%`;
}

// Forecast periods: NWS value or max from hourly
if (period.probabilityOfPrecipitation?.value != null) {
  precipDisplay = `${period.probabilityOfPrecipitation.value}%`;
} else {
  const stats = getPrecipitationStats(period.startTime, period.endTime);
  precipDisplay = stats ? `${stats.max}%` : 'N/A';
}
```

**Reasoning**:
- Current conditions show "right now" data (first hourly point)
- Forecast periods show NWS aggregate or max hourly chance
- Simplified from previous format that showed full range

### Wind Direction Conversion

**Function**: `getWindDirectionEmoji(direction)`

**Mapping**:
```javascript
// NWS provides where wind is FROM
// Arrow shows where wind is blowing TO
const dirMap = {
  'N': '⬇️',   // Wind from North → blowing South
  'NE': '↙️',  // Wind from NE → blowing SW
  'E': '⬅️',   // Wind from East → blowing West
  'SE': '↖️',
  'S': '⬆️',
  'SSW': '↗️',
  'SW': '↗️',
  'W': '➡️',
  'NW': '↘️'
  // ... etc
};
```

**Meteorological Convention**: Wind direction is where it's **coming from**  
**User-Friendly Display**: Arrow shows where it's **going to**

### Weather Icon Selection

**Function**: `getConditionIcon(forecast)`

**Logic** (priority order):
```javascript
// 1. Check for severe weather (always shown)
if (text.includes('tornado')) return '🌪️';
if (text.includes('hurricane')) return '🌀';
if (text.includes('thunder')) return '⛈️';

// 2. Check for precipitation type
if (text.includes('snow')) return '❄️';
if (text.includes('sleet') || text.includes('ice')) return '🧊';
if (text.includes('rain')) return '🌧️';
if (text.includes('drizzle')) return '🌦️';

// 3. Check for obscuration
if (text.includes('fog')) return '🌫️';

// 4. Check cloudiness (time-dependent)
if (text.includes('partly cloudy')) return '⛅';
if (text.includes('cloudy')) return '☁️';

// 5. Default clear (time-dependent)
if (isNight) return '🌙';
return '☀️';
```

**Day/Night Detection**:
1. Use `isDaytime` property if present
2. Fall back to checking `name` for "night"
3. Affects clear/cloudy icons only (not precipitation)

### Severe Weather Detection

**Function**: `getSevereWeather(forecast)`

**Keywords** (case-insensitive):
```javascript
'tornado'           → '🌪️ Tornado'
'severe thunder'    → '⛈️ Severe Storm'
'hurricane'         → '🌀 Hurricane/Tropical'
'high wind'         → '🌬️ High Winds'
'flood'             → '🌊 Flood'
'dense fog'         → '🌫️ Dense Fog'
'smoke'             → '♨️ Smoke'
'air quality'       → '😷 Air Quality'
'extreme heat'      → '🥵 Extreme Heat'
'extreme cold'      → '🥶 Extreme Cold'
```

**Display**: Joined with ` • ` separator  
**Example**: `🌪️ Tornado • 🌊 Flood`

### Dynamic Weather Emojis

**Purpose**: Contextual emojis that adapt to current conditions

**Function**: `getPrecipEmoji(forecast, precipProb)`

**Logic**:
```javascript
// No data available
if (precipProb == null) return '🤷';

// Very low precipitation chance
if (precipProb < 10) return '🌂'; // Closed umbrella

// Type-based precipitation (checks forecast text)
if (text.includes('thunder')) return '⛈️';
if (text.includes('freezing rain') || text.includes('ice')) return '🧊';
if (text.includes('snow')) return '❄️';
if (text.includes('wintry mix')) return '🌨️';
// Default: ☔
```

**Function**: `getWindEmoji(windSpeedStr)`

**Logic**:
```javascript
if (!windSpeedStr || isNaN(windSpeed)) return '🤷';
if (windSpeed >= 15) return '🌬️'; // Windy face
return '💨'; // Light breeze
```

**Threshold Note**: 15 mph chosen based on Hyattsville, MD local climate feel

**Function**: `getHumidityEmoji(humidityValue)`

**Logic**:
```javascript
if (!humidityValue || humidityValue === 0) return '🤷';
if (humidityValue < 40) return '🌵'; // Dry/desert
if (humidityValue > 70) return '💦'; // Humid/muggy
return '💧'; // Normal
```

**Threshold Note**: 40% lower bound chosen for humid subtropical climate (rarely <30% in Hyattsville, MD)

### Weather Idioms

**Function**: `getWeatherIdioms(forecast)`

**Purpose**: Returns array of contextual weather phrases that rotate in header

**Algorithm** (priority order):
```javascript
1. Check severe weather:
   - tornado → "Watch for flying houses"
   - flood → "Everything's coming up Milhouse"

2. Check special combinations:
   - (temp/feels ≥90°F) AND (precip ≥20%) AND (humidity ≥70%)
     → "Hot, humid, might rain"

3. Check precipitation:
   - thunder/storm → "Storms a-brewin"
   - snow/blizzard → "Let is snow"
   - ice/freezing → "Got skates?"
   - rain/shower/drizzle → "Tut tut, it looks like rain"
   - fog → "Lost in the fog"

4. Check temperature extremes:
   - temp ≥90°F → "ITS GON BE HOT"
   - temp ≤32°F → "ITS GON BE COLD"

5. Check humidity:
   - humidity ≥80% → "It's not the heat, it's the humidity"

6. Check wind:
   - windSpeed ≥20mph → "It's not <i>that</i> the wind is blowing..."

7. Check cloud cover:
   - partly cloudy/mostly sunny → "Half cloudy, or half sunny?"
   - cloudy/overcast → "Silver linings abound"

8. Check clear/sunny (exclude partly/mostly):
   - Night → "Perfect for stargazing"
   - Day → "Walkin on sunshine"

9. Default if none match:
   - "Perfect for stargazing"
```

**Rotation Logic**:
```javascript
// Display first idiom immediately
idiomIndex = 0;
updateIdiomDisplay();

// Rotate every 4 seconds if multiple idioms
if (idiomsList.length > 1) {
  setInterval(() => {
    idiomIndex = (idiomIndex + 1) % idiomsList.length;
    updateIdiomDisplay();
  }, 4000);
}
```

**Design Notes**:
- Multiple applicable idioms all displayed in rotation
- Example: Hot, sunny, humid day rotates through:
  1. "ITS GON BE HOT"
  2. "Walkin on sunshine"
  3. "It's not the heat, it's the humidity"
- Special combo prevents redundant hot + sunny when rain likely
- Night/day detection uses `isDaytime` property

### Radar Animation

**State Management**:
```javascript
radarCurrentFrame = 0;           // Index of current frame
radarIsPlaying = false;          // Animation state
radarAnimationTimer = null;      // setInterval ID
```

**Play Logic**:
```javascript
function radarPlayPause() {
  if (radarIsPlaying) {
    clearInterval(radarAnimationTimer);
    radarIsPlaying = false;
  } else {
    radarAnimationTimer = setInterval(() => {
      radarNext();  // Advance frame
    }, 500);        // 500ms per frame (2 fps)
    radarIsPlaying = true;
  }
}

function radarNext() {
  if (radarCurrentFrame < radarFrames.length - 1) {
    showRadarFrame(radarCurrentFrame + 1);
  } else {
    showRadarFrame(0);  // Loop back to start
  }
}
```

**Frame Display**:
```javascript
function showRadarFrame(frameIndex) {
  // Remove old layer
  if (radarLayer) radarMap.removeLayer(radarLayer);
  
  // Add new layer with frame's path
  radarLayer = L.tileLayer(radarUrl, {
    opacity: 0.6,
    maxNativeZoom: 6,  // RainViewer limit
    maxZoom: 10        // Allow zooming but scale tiles
  }).addTo(radarMap);
  
  // Update timestamp display
  updateTimestamp(frame.time);
}
```

---

## State Management

### Global State Variables

```javascript
// Location State
let currentLat = DEFAULT_LAT;
let currentLon = DEFAULT_LON;
let currentLocationName = DEFAULT_NAME;

// Weather Data
let currentData = null;  // See "Weather Data Structure"

// Radar State
let radarMap = null;
let radarLayer = null;
let radarFrames = [];
let radarCurrentFrame = 0;
let radarAnimationTimer = null;
let radarIsPlaying = false;

// Idiom State
let idiomsList = [];              // Array of applicable idioms
let idiomIndex = 0;               // Current idiom being displayed
let idiomRotationTimer = null;    // setInterval ID for rotation
```

### localStorage Keys

| Key | Type | Description |
|-----|------|-------------|
| `veather-locations` | Array\<Location\> | Saved locations (max 5) |
| `veather-columns` | Object | Hourly column visibility preferences |

**Persistence**:
- Saved on every change (immediate write)
- No expiration/TTL
- User clears manually via browser settings

### State Transitions

**Location Change**:
```
User Action (search/click/geolocation)
  ↓
Update currentLat/currentLon/currentLocationName
  ↓
loadWeather() → Fetch NWS data
  ↓
renderWeather() / renderHourly()
  ↓
updateLocationDisplay() → Update header
  ↓
renderSavedLocations() → Highlight active location
  ↓
If radar tab active: radarMap.setView([newLat, newLon])
```

**Tab Switch**:
```
User clicks tab
  ↓
switchTab(tabName)
  ↓
Update .active class on tabs
  ↓
Show/hide tab-content divs
  ↓
If radar tab: initRadar() (lazy initialization)
```

**Weather Refresh**:
```
Manual refresh or 30min timer
  ↓
loadWeather()
  ↓
Fetch NWS points → forecast → hourly → alerts (parallel)
  ↓
Update currentData
  ↓
renderWeather() / renderHourly()
  ↓
Update "Last updated" timestamp
```

---

## Event Handling

### User Interactions

**Search Input**:
```javascript
document.getElementById('search-input').addEventListener('keypress', (e) => {
  if (e.key === 'Enter') {
    searchLocation();
  }
});
```

**Tab Switching**:
```javascript
<button class="tab" onclick="switchTab('forecast')">
```
- Updates active tab styling
- Shows/hides content divs
- Lazy-initializes radar map if needed

**Location Actions**:
```javascript
// Save current location
toggleSaveLocation()
  → Check duplicates
  → Check max limit (5)
  → Prompt for label
  → Save to localStorage
  → Update UI

// Load saved location
loadLocation(index)
  → Update current lat/lon/name
  → Refresh weather data
  → Update UI

// Edit location label
editLocation(index)
  → Prompt for new label
  → Update localStorage
  → Re-render

// Delete location
deleteLocation(index)
  → Confirm dialog
  → Remove from array
  → Save to localStorage
  → Re-render
```

**Column Toggles**:
```javascript
toggleColumn(columnId)
  → Update COLUMNS[id].visible
  → Save to localStorage
  → renderHourly() (full re-render)
```

**Radar Controls**:
```javascript
radarPlayPause() → Start/stop animation
radarPrev()      → Show previous frame
radarNext()      → Show next frame (loop at end)
```

### Auto-Refresh

```javascript
// Initial load
loadWeather();

// Refresh every 30 minutes
setInterval(loadWeather, 30 * 60 * 1000);
```

**Note**: Radar frames are NOT auto-refreshed (only on tab activation)

---

## Performance Considerations

### Network Requests

**Initial Page Load**:
1. Leaflet.js + CSS (CDN, cached)
2. NWS points API (~1KB JSON)
3. NWS forecast API (~50KB JSON)
4. NWS hourly API (~300KB JSON)
5. NWS alerts API (~5KB JSON, often empty)

**Total**: ~356KB + CDN resources

**On Location Change**: Same as above (not cached)

**Radar Tab Activation**:
1. RainViewer weather-maps.json (~2KB)
2. OpenStreetMap base tiles (~50-100KB, cached)
3. RainViewer radar tiles (~20-50KB, depends on zoom)

### Rendering Optimizations

**Hourly Tab**:
- Renders only 24 hours (not full 156)
- Uses CSS Grid (GPU-accelerated)
- Column toggles re-render entire list (no differential updates)

**Forecast Tab**:
- Renders 14 periods (fixed)
- Each period is independent (no shared state)

**Radar Tab**:
- Lazy initialization (only on first visit)
- Leaflet handles tile caching
- Animation uses setInterval (not requestAnimationFrame)

**Header Idioms**:
- Rotates via `setInterval` (4000ms) when multiple idioms applicable
- Uses `innerHTML` (not DOM manipulation) for simplicity
- Timer cleared and reset on location change to prevent memory leaks

### Memory Usage

**Estimated**:
- Weather data: ~350KB in memory
- Radar frames: ~2KB metadata, tiles cached by browser
- localStorage: <5KB (locations + preferences)
- DOM nodes: ~500 elements (hourly tab)

**Total**: <1MB heap + browser tile cache

---

## Browser Compatibility

### Required APIs

| API | Minimum Version |
|-----|-----------------|
| Fetch | Chrome 42+, Firefox 39+, Safari 10.1+ |
| ES6 (async/await) | Chrome 55+, Firefox 52+, Safari 10.1+ |
| CSS Grid | Chrome 57+, Firefox 52+, Safari 10.1+ |
| localStorage | All modern browsers |
| Geolocation | All modern browsers (HTTPS only) |

### Progressive Enhancement

**Core Features** (work everywhere):
- Weather display
- Manual location search
- Saved locations
- Tab navigation

**Enhanced Features** (graceful degradation):
- Geolocation → Falls back to search
- Radar → Requires Canvas support
- Column toggles → Requires localStorage

### Known Limitations

- **iOS Safari**: Geolocation requires user gesture (can't auto-request on load)
- **Firefox**: Leaflet rendering slower than Chrome (tile loading)
- **Edge Legacy**: CSS Grid bugs (not supported anymore)
- **file:// protocol**: Geolocation blocked, CORS issues with some APIs

---

## Security Considerations

### API Keys

**None required** - All APIs are public and free:
- NWS: No authentication
- RainViewer: No authentication
- Nominatim: No authentication (rate-limited)

### CORS

**All APIs support CORS**:
- NWS: `Access-Control-Allow-Origin: *`
- RainViewer: `Access-Control-Allow-Origin: *`
- Nominatim: `Access-Control-Allow-Origin: *`

### Data Validation

**User Input**:
- Search query: Passed to Nominatim (no SQL injection risk)
- Custom labels: Stored in localStorage (XSS risk minimal, not rendered as HTML)

**API Responses**:
- NWS: Trusted source, no validation needed
- Nominatim: Display names rendered as text (not HTML)
- RainViewer: URLs validated by Leaflet

### localStorage Security

**Stored Data**:
- Locations: Non-sensitive
- Preferences: Non-sensitive

**No PII stored** (except user-chosen location labels)

### HTTPS

**Required for**:
- Geolocation API
- PWA installation
- Service worker (future)

**Current**: Works on HTTP for testing, but geolocation disabled

---

## Testing Strategy

### Manual Testing Checklist

**Location Search**:
- [ ] ZIP code search (20782)
- [ ] City, State search (Seattle, WA)
- [ ] City name only (Portland - ambiguous)
- [ ] Invalid search (returns no results)
- [ ] International location (should fail, US only)

**Geolocation**:
- [ ] Browser permission granted
- [ ] Browser permission denied
- [ ] HTTPS vs HTTP behavior

**Saved Locations**:
- [ ] Save location (max 5)
- [ ] Edit label
- [ ] Delete location
- [ ] Load saved location
- [ ] Duplicate detection
- [ ] Active location highlight

**Weather Display**:
- [ ] Forecast tab renders
- [ ] Hourly tab renders
- [ ] Alerts display (when active)
- [ ] Temperature colors accurate
- [ ] Wind direction arrows correct
- [ ] Feels-like conditional display

**Radar**:
- [ ] Map loads and centers
- [ ] Play/pause animation
- [ ] Previous/next frame
- [ ] Timestamp updates
- [ ] Zoom in/out (4-10)

**Mobile**:
- [ ] Touch targets (44x44px minimum)
- [ ] Font sizes readable (temps 3em+, tabs 1em on mobile)
- [ ] Tabs resize and scroll on small screens (<400px)
- [ ] Column toggles scroll horizontally without wrapping
- [ ] Search bar and buttons stay on one line
- [ ] Timestamps use white-space: nowrap
- [ ] Wind display compact (no space between speed/emoji)
- [ ] Time column wide enough (70px) for "12:00 PM"

### Browser Testing

**Desktop**:
- Chrome (Windows/Mac)
- Firefox (Windows/Mac)
- Safari (Mac)
- Edge (Windows)

**Mobile**:
- iOS Safari
- Chrome Android
- Samsung Internet

---

## Future Technical Considerations

### Service Worker Implementation

**Caching Strategy**:
```javascript
// Cache static assets
- veather.html
- manifest.json
- Leaflet CDN resources

// Cache-first for tiles
- OpenStreetMap tiles (7 day TTL)
- RainViewer tiles (1 hour TTL)

// Network-first for data
- NWS API responses
- Nominatim responses
```

### Performance Improvements

**Potential Optimizations**:
1. Differential rendering for hourly tab (only update changed rows)
2. Virtual scrolling for hourly tab (render visible rows only)
3. Web Workers for data processing (aggregation, color calculations)
4. IndexedDB for weather data caching (larger than localStorage)
5. requestAnimationFrame for radar animation (smoother)

### Accessibility Enhancements

**ARIA Labels**:
```html
<button aria-label="Search for location">🔍</button>
<div role="tablist">
  <button role="tab" aria-selected="true">Forecast</button>
</div>
```

**Keyboard Navigation**:
- Tab order logical
- Enter key for search
- Arrow keys for radar frame navigation
- Escape to close modals (if added)

### Internationalization

**Considerations**:
- Metric/Imperial unit toggle
- Temperature °C/°F conversion
- Wind speed km/h vs mph
- Precipitation mm vs inches
- Date/time formatting (locale-aware)

**Limitation**: NWS API is US-only (would need different API for international)

---

## Appendix

### Color Palette Reference

**Temperature Colors** (hex values):
```
#aa3300 - 95°F+   - Dark burnt orange
#ff5500 - 90-94°F - Red-orange
#ffaa00 - 80-89°F - Amber
#ffdd77 - 70-79°F - Peach/tan
#44dddd - 60-69°F - Bright cyan
#5599ff - 50-59°F - Sky blue
#3366cc - 40-49°F - Royal blue
#aaccff - 30-39°F - Pale blue
#ddeeff - 20-29°F - Ice blue
#ffffff - <20°F   - White
```

**UI Colors**:
```
#1a1a1a - Background (dark gray)
#2a2a2a - Card background
#444    - Border color
#4da6ff - Accent blue (tabs, links)
#ddd    - Primary text
#999    - Secondary text
#666    - Tertiary text
#cc6600 - Warning/alert (orange)
```

### NWS Grid System

**How it works**:
1. Lat/lon → `/points` endpoint
2. Returns grid office code (e.g., "LWX") and X/Y coordinates
3. Grid coordinates used for forecast endpoints
4. Each office covers specific region
5. Grid resolution: 2.5km × 2.5km

**Example**:
- Hyattsville, MD: 38.9469, -76.9320
- Office: LWX (Sterling, VA)
- Grid: 97, 71
- Forecast URL: `/gridpoints/LWX/97,71/forecast`

### Leaflet Map Configuration

```javascript
L.map('radar-map', {
  maxZoom: 10,     // User can zoom to level 10
  minZoom: 4       // User can zoom out to level 4
})

L.tileLayer(url, {
  maxNativeZoom: 6,  // Tiles only exist up to zoom 6
  maxZoom: 10,       // But allow zooming to 10 (scale tiles)
  opacity: 0.6       // Semi-transparent radar overlay
})
```

**Zoom Levels**:
- 4: Regional (entire Northeast visible)
- 6: Local (city + suburbs)
- 8: Neighborhood
- 10: Street-level (tiles are scaled/pixelated)

---

**End of Technical Specifications**

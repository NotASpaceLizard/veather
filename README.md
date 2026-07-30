# Veather - Weather Web App

A clean, dark-themed, mobile-optimized weather application using the National Weather Service (NWS) API and RainViewer radar.

## Overview

Single-page progressive web app that displays current conditions, multi-day forecast, detailed hourly weather data, and animated radar for any US location. Features location search, saved locations, weather idioms, and colorblind-friendly design. Built with vanilla HTML/CSS/JavaScript - no frameworks required.

## Features

### Current Implementation

#### Location Search & Management
- **Search bar** - Accepts multiple input formats:
  - ZIP code (e.g., "20782")
  - City, State (e.g., "Seattle, WA")
  - City name (e.g., "Portland")
- **"Use My Location" button** - Browser geolocation integration
- **Saved locations** - Save up to 5 locations with custom labels
  - Quick access from dedicated Locations tab
  - Edit labels (e.g., "Home", "Work", "Mom's house")
  - Delete unwanted locations
  - Active location highlighted
- **Geocoding** - Powered by Nominatim (OpenStreetMap)
- **Persistent storage** - Locations saved to localStorage

#### Header
- **Weather idioms** - Rotating contextual phrases based on current conditions:
  - Match weather conditions (sunny, rainy, stormy, snowy, etc.)
  - React to temperature extremes (hot, cold)
  - Respond to humidity and wind
  - Multiple idioms rotate every 4 seconds when applicable
  - Examples: "Walkin on sunshine", "Storms a-brewin", "Hot, humid, might rain"

#### Forecast Tab
- **Current conditions card** - Large display with:
  - Temperature with condition icon
  - Short forecast description
  - Precipitation probability (current hour data)
  - Wind speed & direction (current hour data, arrow emoji shows where wind is blowing TO)
  - Humidity (current hour data)
  - Severe weather warnings when applicable

- **Multi-day forecast periods** - Each showing:
  - Period name (e.g., "Tonight", "Thursday", "Thursday Night")
  - High/Low temperature with icon
  - Detailed forecast text
  - Precipitation (NWS value or max from hourly data)
  - Wind speed & direction
  - Humidity (average with range from hourly data)

#### Hourly Tab
- **24-hour detailed view** with visual bar charts for:
  - Temperature with conditional feels-like (shows "85°/92°" when different by 3°F+)
  - Precipitation probability
  - Precipitation amount (inches)
  - Wind speed & direction (no space between, e.g., "15⬇️", color-coded by intensity)
  - Humidity percentage
  - UV index (color-coded by danger level)
- **Customizable columns** - Toggle which data columns to display
  - AMT and UV hidden by default to reduce clutter
  - Preferences saved to localStorage
  - Grid adjusts dynamically
  - Toggles scroll horizontally on mobile without wrapping

#### Radar Tab
- **Interactive map** - Powered by Leaflet + OpenStreetMap
- **Animated radar** - RainViewer radar overlay
  - Past frames + nowcast data
  - Play/pause animation controls (▶️/⏸️ emoji buttons)
  - Previous/next frame navigation (⏮️/⏭️ buttons)
  - Timestamp display
  - Auto-loop playback
- **Dynamic centering** - Automatically centers on current location
- **Zoom support** - Zoom levels 4-10 (regional to local view)

#### Weather Alerts
- Active NWS alerts displayed prominently at top of Forecast tab
- Orange background for high visibility
- Shows alert type and headline

#### Smart Features
- **Weather icons** - Emoji-based, context-aware (day/night detection)
- **Dynamic emojis** - Adapt to conditions:
  - Precipitation: 🌂 <10% chance, ⛈️ thunder, ❄️ snow, 🧊 ice, ☔ rain, 🤷 no data
  - Wind: 🌬️ ≥15 mph, 💨 light breeze, 🤷 no data
  - Humidity: 🌵 <40% (dry), 💧 40-70% (normal), 💦 >70% (humid), 🤷 no data
- **Wind direction arrows** - Shows where wind is blowing (not where it's from)
- **Severe weather detection** - Flags 🌪️tornadoes, ⛈️storms, 🌊floods, 🥵heat, 🥶cold, etc.
- **Auto-refresh** - Weather data updates every 30 minutes
- **Manual refresh** - Button at bottom with last updated timestamp
- **Temperature color coding** - Colorblind-friendly palette:
  - Hot (95°F+): Dark burnt orange → red-orange → amber
  - Mild (60-79°F): Peach/tan → bright cyan
  - Cold (20-59°F): Sky blue → royal blue → pale blue → ice blue → white

#### Design
- **Dark theme** - #1a1a1a background with high contrast
- **Mobile-optimized** - Touch-friendly buttons (44x44px), larger fonts, optimized spacing
  - Tabs resize on small screens to fit better
  - Column toggles scroll horizontally without wrapping
  - Search bar stays on one line with location buttons
  - Timestamps and wind values prevent line wrapping
- **Responsive layout** - Adapts to screen size with breakpoints at 400px & 500px
- **Accessibility** - Colorblind-friendly color palette, clear labels, emoji indicators
- **PWA-ready** - Installable with manifest.json

## Technical Details

### Location Management
- **Default location**: `38.946974, -76.932044` (Hyattsville, MD)
- **Current location**: Stored in variables `currentLat`, `currentLon`, `currentLocationName`
- **Saved locations**: Stored in localStorage as array of `{lat, lon, name, label?}` objects
- **Geocoding**: Nominatim API (free, no API key required)
  - Forward geocoding: address → coordinates
  - Reverse geocoding: coordinates → address

### API Integration
**NWS Weather API** (3 endpoints):
1. `/points/{lat},{lon}` - Gets forecast URLs for location
2. `/forecast` - Gets 7-day forecast periods (14 periods: day/night)
3. `/forecastHourly` - Gets 156-hour detailed hourly forecast
4. `/alerts/active/zone/{zone}` - Gets active weather alerts

**RainViewer Radar API**:
- `weather-maps.json` - Gets available radar frame timestamps
- Tile endpoint - Serves radar imagery as map tiles

**Nominatim Geocoding API**:
- `/search` - Forward geocoding (address → coordinates)
- `/reverse` - Reverse geocoding (coordinates → address)

### Data Processing
- **Hourly data aggregation** - Calculates average, min, max for humidity & precipitation per forecast period
- **Feels-like temperature** - Uses NWS `heatIndex` (hot) or `windChill` (cold) when available
- **Temperature color coding** - 10 color ranges optimized for colorblindness (amber/orange/cyan/blue spectrum)
- **Wind speed color coding** - Calm (blue) → moderate (yellow) → high (orange)
- **UV index color coding** - Low (blue) → moderate (blue) → high (yellow) → extreme (orange)

### Dependencies
- **Leaflet** (v1.9.4) - Interactive map library (loaded from CDN)
- **OpenStreetMap** - Base map tiles
- **RainViewer** - Radar overlay tiles

### File Structure
```
veather/
├── veather.html       # Main app (all-in-one: HTML/CSS/JS)
├── manifest.json      # PWA manifest for installation
├── color-palette.html # Development tool for color palette testing
└── README.md          # This file
```

## Usage

### Local Development
1. Open `veather.html` in any modern web browser
2. App loads with default location (Hyattsville, MD)
3. Search for locations using:
   - ZIP code: Type "20782" and press Enter
   - City, State: Type "Seattle, WA" and press Enter
   - City name: Type "Portland" and press Enter
4. Use 📍 button to get your current location via browser
5. Save locations with ⭐ button (up to 5 saved)
6. Switch between tabs:
   - **Forecast** - Current conditions + 7-day outlook
   - **Hourly** - 24-hour detailed data with charts
   - **Radar** - Animated precipitation radar
   - **Locations** - Manage saved locations
7. Click 🔄 button at bottom to manually refresh weather data

### Deployment
- Can be hosted on any static file server (GitHub Pages, Netlify, Vercel, etc.)
- No backend required
- No build process needed
- Works completely client-side

### Installation as PWA
- On mobile browsers, use "Add to Home Screen" option
- App will open in standalone mode (fullscreen, no browser UI)

## Browser Compatibility
- Requires ES6+ JavaScript support
- Fetch API for HTTP requests
- Geolocation API (optional, for "Use My Location" feature)
- localStorage for saved locations and preferences
- Works on all modern browsers (Chrome, Firefox, Safari, Edge)
- Tested on mobile (iOS Safari, Chrome Android)

## Future Enhancement Ideas

### Potential Features to Add
- [ ] Notification support for severe weather alerts
- [ ] Extended forecast (beyond 7 days)
- [ ] Historical weather comparison
- [ ] Weather trend charts/graphs
- [ ] Sunrise/sunset times with twilight info
- [ ] Moon phase display
- [ ] Share weather function (share location or screenshot)
- [ ] Units toggle (Imperial/Metric)
- [ ] Multiple weather alert sources
- [ ] Air quality index integration
- [ ] Pollen count data
- [ ] Astronomy data (meteor showers, ISS passes)

### Technical Improvements
- [ ] Service worker for offline support
- [ ] Loading states/skeleton screens
- [ ] Better error handling and retry logic
- [ ] Performance optimizations (debouncing, lazy loading)
- [ ] Enhanced accessibility (ARIA labels, keyboard navigation)
- [ ] Export/import saved locations
- [ ] Dark/light mode toggle
- [ ] Customizable themes

## Development Notes

### Data Sources
- **Current conditions** - Uses first hourly forecast data point for most accurate "right now" data:
  - Temperature and feels-like from current hour
  - Precipitation probability from current hour
  - Wind speed and direction from current hour
  - Humidity from current hour
- **Forecast periods** - Uses NWS 12-hour periods:
  - Precipitation: Shows NWS value if available, otherwise max from hourly data
  - Humidity: Calculates avg [min-max] range from hourly data within period
  - Wind: Uses NWS period value

### Icon Logic
- Day/night detection uses `isDaytime` property from API
- Falls back to checking period name for "night"
- Weather conditions override time-based icons (rain always shows rain, etc.)

### Weather Idioms
- Rotates through all applicable idioms every 4 seconds (if multiple apply)
- Priority order: severe weather → precipitation → temperature → humidity → wind → clouds → clear
- Special combo: "Hot, humid, might rain" (temp/feels ≥90°F, precip ≥20%, humidity ≥70%)
- Examples by condition:
  - Clear/sunny: "Walkin on sunshine" / "Perfect for stargazing" (night)
  - Partly cloudy: "Half cloudy, or half sunny?"
  - Rain: "Tut tut, it looks like rain"
  - Snow: "Let is snow"
  - Storms: "Storms a-brewin"
  - Hot (≥90°F): "ITS GON BE HOT"
  - Cold (≤32°F): "ITS GON BE COLD"
  - Humid (≥80%): "It's not the heat, it's the humidity"
  - Windy (≥20mph): "It's not <i>that</i> the wind is blowing..."

### Wind Direction
- API provides where wind is **coming FROM** (meteorological convention)
- App converts to arrow showing where wind is **blowing TO** (user-friendly)
- Example: "N" wind (from north) shows ⬇️ (blowing south)

### Temperature Colors (Colorblind-Friendly)
Color palette designed for red-green colorblindness:
- Uses **cyan/teal** instead of green for mid-range temps
- **Amber/orange** spectrum for hot (avoids pure red)
- **Blue** gradient for cold with high contrast between shades
- Tested with Option 2 "More Contrast" palette from color-palette.html

### Feels-Like Temperature
- Only displayed when significantly different (±3°F) from actual temp
- Uses NWS `heatIndex` property when available (hot, humid conditions)
- Uses NWS `windChill` property when available (cold, windy conditions)
- Format: "85°/92°" (actual/feels-like) inline to save space

### Radar Data
- RainViewer provides past radar frames + short-term nowcast
- Radar tiles only available up to zoom level 6
- Map allows zoom 4-10 (scales tiles beyond native resolution)
- Animation plays at 500ms per frame

## Troubleshooting

### No data loading
- Check browser console for errors
- Verify internet connection
- NWS API may be temporarily unavailable

### Location search not working
- Check browser console for errors
- Nominatim API may be rate-limited (wait a moment and retry)
- Ensure spelling is correct for city names
- Use "City, ST" format for better results (e.g., "Portland, OR" not just "Portland")

### Geolocation not working
- Check browser permissions for location access
- HTTPS required for geolocation on most browsers
- Some browsers block geolocation on file:// protocol (use local server)

### Wrong location results
- Try more specific search (add state abbreviation)
- Ensure coordinates are within US/territories (NWS coverage area)
- Use ZIP code for most accurate results

### Alerts not showing
- Alerts only show when active for the location's forecast zone
- NWS determines zone automatically from coordinates
- Check NWS website to verify alerts exist for your area

### Radar not loading
- Check browser console for errors
- RainViewer API may be temporarily unavailable
- Clear browser cache and reload
- Ensure internet connection is stable

### Saved locations disappeared
- Check browser localStorage hasn't been cleared
- Some browsers clear localStorage in private/incognito mode
- Export/backup important locations manually (copy from Locations tab)

## Attribution

- **Weather data**: National Weather Service (NWS) API - free, public API maintained by NOAA
- **Radar imagery**: RainViewer API - free radar data aggregation service
- **Geocoding**: Nominatim (OpenStreetMap) - free geocoding service
- **Base maps**: OpenStreetMap contributors
- **Mapping library**: Leaflet - open-source JavaScript library

---

**Last Updated**: 2026-07-30
**Version**: 1.1
**Status**: Production ready

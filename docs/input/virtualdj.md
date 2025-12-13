# Virtual DJ

> NOTE: This source does not support Oldest mix mode.

**What's Now Playing** integrates with Virtual DJ through multiple data extraction methods to provide
comprehensive track information and playlist access. This guide explains how the integration works and how to
configure it.

## How It Works

Virtual DJ support combines three complementary data sources:

### 1. History M3U Files (Real-time Track Detection)

Virtual DJ writes history files in M3U format to track recently played songs. These files use Virtual DJ's
proprietary `#EXTVDJ` format extension to embed rich metadata directly in the history file.

**What's Now Playing** monitors the History directory for changes and immediately extracts:

* Artist and title from `#EXTVDJ` XML tags
* Optional remix information (can be toggled)
* File paths (if available)

**Example EXTVDJ Format:**

```m3u
#EXTM3U
#EXTVDJ:<time>21:39</time><lastplaytime>1674884385</lastplaytime><artist>Nick Cave & The Bad Seeds</artist><title>Hollywood</title>
netsearch://dz1873796677
#EXTVDJ:<time>21:41</time><lastplaytime>1674884510</lastplaytime><artist>Kid 'N Play</artist><title>Can You Dig That</title><remix>Extended Mix</remix>
netsearch://dz85144450
```

> Note: Each `#EXTVDJ` tag is a single line in the actual file. The last entry represents the currently
> playing track.

### 2. Database.xml (Enhanced Metadata)

Virtual DJ maintains a master XML database (`database.xml`) with complete metadata for all tracks in your
collection. **What's Now Playing** periodically reads and indexes this database to provide enhanced metadata
including:

* Album name
* Genre
* Year
* BPM
* Musical key
* Record label
* Track number
* File paths

This database is parsed using SAX streaming to handle large collections (10,000+ tracks) efficiently.

### 3. Playlist Files (Request System Support)

Virtual DJ stores playlists in two formats depending on version:

**VirtualDJ 2024+ (Unified Format):**

* Playlists stored as `.vdjfolder` XML files in the `MyLists` directory
* Contains embedded metadata (artist, title, album, etc.) directly in the playlist file

**Example .vdjfolder Format:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<playlist>
  <song path="/path/to/track.mp3" artist="Test Artist" title="Test Song" album="Test Album"
  genre="Electronic" bpm="128" key="Am"/>
</playlist>
```

**VirtualDJ 2023 and Earlier (Legacy Format):**

* `.m3u` files in `Playlists` directory (simple file path lists)
* `.vdjfolder` files scattered throughout the VDJ directory structure
* Requires cross-referencing with `database.xml` for metadata

## Instructions

[![Virtual DJ Source Selection](images/virtualdj-source-selection.png)](images/virtualdj-source-selection.png)

1. Open Settings from the **What's Now Playing** icon
2. Select Core Settings->Source from the left-hand column
3. Select Virtual DJ from the list of available input sources
4. Select Input Sources->Virtual DJ from the left-hand column

## Setup

[![Virtual DJ Settings](images/virtualdj-settings.png)](images/virtualdj-settings.png)

### Configure Directories

**What's Now Playing** typically auto-detects Virtual DJ directories in these locations:

* `Documents/VirtualDJ/` (traditional location)
* `AppData/Local/VirtualDJ/` (Windows 11+ default)

If auto-detection fails or you use a custom location:

1. **History Directory** - Directory where VirtualDJ writes `.m3u` history files
   * Default: `Documents/VirtualDJ/History`
   * Use **Select Dir** button to choose custom location
2. **Playlists Directory** - Directory for playlist files
   * Default: `Documents/VirtualDJ/Playlists`
   * Use **Select Dir** button to choose custom location

### Database Settings

1. Click **Re-read** to build the metadata database
   * Parses `database.xml` to extract full metadata for all tracks
   * May take several minutes for large collections (10,000+ tracks)
   * Background process - application remains responsive
2. Set **Max age** for automatic database refreshes (default: 7 days)
   * Database automatically rebuilds when older than this threshold
   * Ensures metadata stays current when you add/edit tracks
3. **Use Remix Field** - Controls handling of remix information
   * Enabled: Combines remix field with title (e.g., "Song Title (Extended Mix)")
   * Disabled: Shows only base title without remix information

## Data Flow

Here's how **What's Now Playing** processes track information:

1. **Track Change Detection:** File watcher detects changes to History `.m3u` files
2. **Basic Metadata Extraction:** Parses `#EXTVDJ` tags for artist/title (immediate)
3. **Enhanced Metadata Lookup:** Queries local database for additional metadata (BPM, genre, album, etc.)
4. **Output:** Combined metadata sent to all configured outputs (OBS, Twitch, Discord, etc.)

### Performance Characteristics

* **Track detection latency:** <1 second (monitored via file system events)
* **Database refresh:** Background process, non-blocking
* **Memory usage:** Efficient SAX parsing - handles 50,000+ track libraries
* **Disk usage:** Two SQLite databases cached locally (songs and playlists)

## Version Compatibility

**Supported VirtualDJ Versions:**

* VirtualDJ 2024 and later (full support including unified MyLists playlists)
* VirtualDJ 2023 and earlier (full support with legacy playlist format)
* VirtualDJ 8.x (basic support - M3U history parsing)

**History File Format:**

* Extended M3U with `#EXTVDJ` proprietary tags (all versions)
* UTF-8, ASCII, CP1252, and UTF-16 encoding support

## Troubleshooting

### No Track Updates

* Check that History directory path is correct
* Ensure VirtualDJ is writing history files (play tracks long enough to trigger writes)
* Verify file system permissions allow reading the History directory
* Try enabling "Use Polling Observer" in Settings → Quirks for network drives

### No Enhanced Metadata

* Click **Re-read** to build database from VirtualDJ's collection
* Verify `database.xml` exists in the parent directory of Playlists
  (typically `Documents/VirtualDJ/database.xml`)
* Check that database age hasn't exceeded max age threshold
* Large collections (10,000+ tracks) may take several minutes to parse

### Playlist Requests Not Working

* Ensure Playlists directory is correct
* Click **Re-read** after making playlist changes in VirtualDJ
* Playlist files only update when VirtualDJ is closed or manually saved
* For VirtualDJ 2024+, verify `MyLists` directory exists and contains `.vdjfolder` files
* Check that artist query scope is set correctly (entire library vs selected playlists)

### Changing Metadata

If What's Now Playing acts erratic when changing metadata in VirtualDJ:

1. Pause What's Now Playing from the menu
2. Make your metadata changes in VirtualDJ
3. Save changes and play the next track
4. Click **Re-read** to rebuild the database
5. Unpause What's Now Playing

### File Path Substitution

For networked setups or different drive mappings:

* Configure path substitution in Settings → Quirks
* Use "File Substitution" to map Virtual DJ's paths to local paths
* Example: Map `Z:\Music` (network) to `C:\LocalMusic` (local)

## Advanced Configuration

### Artist Query Scope

Controls which tracks are considered when checking if an artist is in your library:

* **Entire Library** - Searches all tracks in `database.xml`
* **Selected Playlists** - Searches only specified playlists (comma-separated names)

### Database Caching

Two SQLite databases are maintained in your system's cache directory:

* `virtualdj-songs.db` - Indexed metadata from `database.xml`
* `virtualdj-playlists.db` - Indexed playlist contents

These databases are automatically rebuilt when:

* Clicking the **Re-read** button
* Database age exceeds **Max age** threshold
* Database files are missing or corrupted

## Playback Position Tracking

**Current Status:** Virtual DJ integration does **NOT** track playback position or elapsed time.

### What's Available in EXTVDJ Format

Virtual DJ's history files include timing fields that could theoretically support position tracking:

* `<time>` - Wall clock time when the track started (24-hour format)
* `<lastplaytime>` - Unix epoch timestamp when the track was last played

### Why Position Tracking Is Not Implemented

**What's Now Playing** currently **does not parse or use** these timing fields because:

1. **Track Change Detection Focus** - The primary goal is detecting *what* is playing, not *when* or *how long*
2. **File-Based Polling Limitations** - History files are updated intermittently by Virtual DJ, not continuously
3. **Accuracy Concerns** - File modification timestamps and write delays make real-time position tracking unreliable
4. **No Duration in History** - EXTVDJ format doesn't include track duration, requiring database lookups
5. **No Update Frequency Control** - Virtual DJ controls when history files are written, not What's Now Playing

### Potential Accuracy If Implemented

If position tracking were added using the `lastplaytime` timestamp:

**Best Case Scenario:**

* **±1-3 seconds accuracy** when history files are written immediately
* Calculated position = (current time - lastplaytime)
* Works for tracks longer than ~30 seconds

**Real-World Limitations:**

* **Variable write delays** - VDJ may buffer writes for several seconds
* **No update during playback** - Position only updates when file is rewritten
* **Crossfader/deck switching** - Rapid changes may have stale timestamps
* **File system caching** - OS-level buffering can delay file watcher notifications
* **Network drives** - SMB/NFS latency compounds accuracy issues

### Alternative Approaches for Better Accuracy

For applications requiring precise playback position:

1. **Database Duration Field** - Virtual DJ's `database.xml` includes track duration
   * Calculate remaining time: `duration - (current_time - lastplaytime)`
   * Still limited by history file update frequency
2. **MPRIS2 Support** - Virtual DJ doesn't currently expose MPRIS2 interface
   * Would provide native position updates if implemented
3. **API Integration** - Virtual DJ lacks public real-time API
   * No official method for millisecond-accurate position tracking
4. **Polling Observer Tuning** - Enable "Use Polling Observer" in Settings → Quirks
   * Reduces file watcher latency on some systems
   * Still limited by VDJ's write frequency

### Using MPRIS2 for Better Accuracy

**MPRIS2** (Media Player Remote Interfacing Specification) is a Linux DBus protocol that provides real-time
playback information. **What's Now Playing** supports MPRIS2 natively for track metadata.

**Supported MPRIS2 Players:**

* **Mixxx** - Free open-source DJ software with full MPRIS2 support (Linux only)
* **VLC** - Media player with MPRIS2 support
* **Rhythmbox, Amarok, Clementine** - Music players with MPRIS2
* **Virtual DJ does NOT support MPRIS2** (no plans announced)

**How to Use MPRIS2:**

1. **Switch to MPRIS2-compatible software** (e.g., Mixxx for DJing)
2. **In What's Now Playing Settings:**
   * Core Settings → Source → Select "MPRIS2"
   * Input Sources → MPRIS2 → Select your player from detected sources
3. **Benefits:**
   * Track duration available immediately (`mpris:length`)
   * Real-time metadata updates via DBus signals
   * No file polling or write delays
   * Works only on Linux (DBus requirement)

**Position Tracking:** MPRIS2 sources provide position data via `org.mpris.MediaPlayer2.Player.Position`
property, but **What's Now Playing does not currently extract or use this data**. The implementation only
reads track metadata (artist, title, album, duration) from `Metadata` property. Adding position tracking
would require polling the `Position` property periodically.

**See:** [MPRIS2 Documentation](mpris2.md) for full setup instructions.

### Using JRiver for Better Accuracy

**JRiver Media Center** is a commercial media player (Windows/Mac/Linux) with a web service API (MCWS) that
provides comprehensive playback information.

**How to Use JRiver:**

1. **Enable Media Network in JRiver:**
   * Tools → Options → Media Network
   * Check "Use Media Network to share this library"
   * Note the port number (default: 52199)
2. **In What's Now Playing Settings:**
   * Core Settings → Source → Select "JRiver"
   * Input Sources → JRiver → Configure host/port
   * Use `localhost` or `127.0.0.1` for local setup
3. **Benefits:**
   * Track duration via `DurationMS` field
   * HTTP-based API (works across network)
   * Supports Windows, Mac, and Linux
   * Commercial software ($60-80)

**Position Tracking:** The current What's Now Playing implementation uses JRiver's `/Playback/Info` endpoint
which provides track metadata and duration but does not include playback position data. JRiver's MCWS API may
have other endpoints that provide position information - consult JRiver's API documentation for complete
endpoint details.

**See:** [JRiver Documentation](jriver.md) for full setup instructions.

### Maximum Accuracy: What You Can Do

To achieve the best possible accuracy with **What's Now Playing**, regardless of your music source:

**For Virtual DJ Users:**

1. **Enable Polling Observer** (Settings → Quirks)
   * Check "Use Polling Observer"
   * Set polling interval to 0.5-1.0 seconds
   * Reduces file watcher latency on some systems
2. **Use Local Drives** - Avoid network shares
   * Network latency adds 100-500ms delays
   * File system caching is more reliable on local drives
3. **Rebuild Database Regularly** (Input Sources → Virtual DJ)
   * Click "Re-read" after adding/editing tracks
   * Ensures duration metadata is current
4. **Accept Limitations** - VDJ file-based approach limits accuracy to ±3-5 seconds

**For Maximum Accuracy (Any Platform):**

1. **Switch to MPRIS2-compatible software** (Linux only)
   * **Mixxx** - Free DJ software with full MPRIS2 support
   * Best option for accurate track change detection
   * Position tracking would require code enhancement
2. **Use JRiver Media Center** (Windows/Mac/Linux)
   * Commercial option with API support
   * Provides duration but not real-time position
3. **Implement Position Tracking** (Advanced)
   * Fork What's Now Playing and add:
     * MPRIS2 `Position` property polling (data available from DBus but not extracted)
     * JRiver position tracking (investigate MCWS API for position endpoints)
     * Virtual DJ `lastplaytime` parsing with duration calculation
   * Contributions welcome to the project!

**Hardware Optimization:**

* **SSD over HDD** - Reduces file I/O latency
* **Wired network** - Avoid WiFi for network drives
* **Dedicated machine** - Reduce CPU contention for file watchers

### Recommendation

For streaming/DJ applications, **track change detection** (current implementation) is usually sufficient since
viewers primarily care about *what song is playing* rather than *exact position within the song*. If precise
position tracking is critical, consider:

* **Best option:** Switch to Mixxx (Linux) with MPRIS2 for accurate track detection
* **Commercial option:** Use JRiver Media Center for cross-platform API access
* **Accept limitations:** Virtual DJ with file polling provides ±3-5 second accuracy
* **Contribute code:** Implement position tracking and submit a pull request!

## Technical Details

### File Format Specifications

**EXTVDJ Tag Structure:**

```text
#EXTVDJ:<time>HH:MM</time><lastplaytime>UNIX_TIMESTAMP</lastplaytime><artist>ARTIST_NAME</artist><title>TITLE</title><remix>REMIX_INFO</remix>
```

* `time` - 24-hour format timestamp (currently not parsed by What's Now Playing)
* `lastplaytime` - Unix epoch timestamp (currently not parsed by What's Now Playing)
* `artist` - Track artist (supports special characters and ampersands)
* `title` - Track title
* `remix` - Optional remix/version information

**Database.xml SAX Parsing:**

* Streams XML to handle large files without loading entire file into memory
* Extracts `<Song>` elements with `FilePath` attribute
* Parses `<Tags>` child elements for metadata fields
* Inserts into SQLite with indexed artist/title columns for fast lookups

**VDJFolder Format:**

* XML format with `<song>` elements
* Attributes include: `path`, `artist`, `title`, `album`, `genre`, `year`, `bpm`, `key`, `label`,
`tracknumber`
* Newer format provides richer metadata than legacy M3U playlists

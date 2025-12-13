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
#EXTVDJ:<time>21:39</time><lastplaytime>1674884385</lastplaytime><artist>Nick Cave & The Bad
Seeds</artist><title>Hollywood</title>
netsearch://dz1873796677
#EXTVDJ:<time>21:41</time><lastplaytime>1674884510</lastplaytime><artist>Kid 'N
Play</artist><title>Can You Dig That</title><remix>Extended Mix</remix>
netsearch://dz85144450
```

The last entry in the file represents the currently playing track.

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

## Technical Details

### File Format Specifications

**EXTVDJ Tag Structure:**

```xml
#EXTVDJ:<time>HH:MM</time><lastplaytime>UNIX_TIMESTAMP</lastplaytime><artist>ARTIST_NAME</artist>
<title>TITLE</title><remix>REMIX_INFO</remix>
```

* `time` - 24-hour format timestamp
* `lastplaytime` - Unix epoch timestamp
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

# MPV Remote App - Modifications

This document describes the modifications made to the MPV Remote app.

## Features Added

### November 8, 2025 - Major UI/UX Improvements & Playlist Management

#### 1. Playback Control Button Sizing
- **Increased button sizes** for better mobile usability
- Changed font-size from 10px to 20px
- Added explicit width/height of 70px with max-width of 150px
- Added 5px padding to container to prevent buttons from going off-screen
- Improved flex layout for better responsiveness

**File: `src/components/playerController.vue`**
```css
.videoControls {
  padding: 0 5px;
}
.videoControls ion-button {
  font-size: 20px;
  min-width: 70px;
  height: 70px;
  flex: 1;
  max-width: 150px;
  margin: 0;
}
```

#### 2. Favorites Feature
- **Added ability to favorite/star videos and songs** for easy filtering and access
- Star icon displayed next to favorited items in file browser
- Filter button in file browser to show only favorites (including subdirectories)
- Smart label: "Favorite Video?" or "Favorite Song?" based on file type detection
- Larger 24px star icons with -2px positioning adjustment for better visibility

**Backend Changes (`mpv-remote-node`):**

**File: `crud.js`**
- Added `favorited` column to `mediastatus` table (INTEGER DEFAULT 0)
- Migration code to add column to existing databases
- `toggleFavorite(filepath)` - Toggle favorite status for a file
- `getFavorites(directory)` - Get all favorites, optionally filtered by directory (with recursive subdirectory support using SQL LIKE)

**File: `remoteServer.js`**
- POST `/api/v1/favorites/toggle` - Toggle favorite status
- GET `/api/v1/favorites?directory=<path>` - Get favorites for directory and subdirectories
- GET `/api/v1/favorites/status/:filepath` - Get favorite status for specific file
- Added `path` property to MPV status data (full file path, not just filename)

**Frontend Changes (`mpv-remote-app`):**

**File: `src/components/playerController.vue`**
- Added "Favorite Video?"/"Favorite Song?" button with star icon
- Media type detection based on track-list or file extension
- Changed to use `playerData.value.path` instead of `filename` for proper directory association
- Star fills in when favorited

**File: `src/components/fileBrowserModal.vue`**
- Added star filter button in header (yellow color when active)
- Star icons displayed next to favorited files (24px, gold color)
- Filter fetches from backend to show favorites in current directory and all subdirectories
- Creates synthetic file entries from favorite data when filtering

#### 3. Saved Playlists Feature
- **Complete playlist management system** - save, load, edit, and manage multiple playlists
- Two-tab interface: "Current Queue" (active MPV playlist) and "Saved" (saved playlists)
- Save current queue as a named playlist
- Load saved playlists into MPV
- View and edit saved playlist contents
- Reorder entries within saved playlists (drag and drop)
- Delete individual entries from saved playlists
- Delete entire saved playlists

**Backend Changes (`mpv-remote-node`):**

**File: `crud.js`**
- Created `saved_playlist` table (id, name, created_date)
- Created `saved_playlist_entry` table (id, playlist_id, file_path, position)
- `createSavedPlaylist(name, entries)` - Create new saved playlist
- `getSavedPlaylists()` - Get all saved playlists
- `getSavedPlaylist(id)` - Get specific playlist with entries
- `deleteSavedPlaylist(id)` - Delete saved playlist
- `addEntryToSavedPlaylist(playlistId, filePath)` - Add entry to playlist
- `removeEntryFromSavedPlaylist(entryId)` - Remove entry from playlist
- `updatePlaylistEntryPositions(playlistId, entries)` - Reorder playlist entries

**File: `remoteServer.js`**
- GET `/api/v1/saved-playlists` - List all saved playlists
- POST `/api/v1/saved-playlists` - Create new playlist with entries
- GET `/api/v1/saved-playlists/:id` - Get playlist details with entries
- DELETE `/api/v1/saved-playlists/:id` - Delete playlist
- POST `/api/v1/saved-playlists/:id/entries` - Add entry to playlist
- DELETE `/api/v1/saved-playlists/:playlistId/entries/:entryId` - Remove entry
- POST `/api/v1/saved-playlists/:id/load` - Load playlist into MPV (clears current queue, adds all entries)
- PUT `/api/v1/saved-playlists/:id/reorder` - Reorder playlist entries

**Frontend Changes (`mpv-remote-app`):**

**File: `src/views/Playlist.vue`**
- Complete rewrite with tabbed interface
- **Current Queue tab:** Shows MPV's active playlist with "Save Queue" button, reordering, and item removal
- **Saved tab:** Lists all saved playlists with creation dates
- Click playlist to view/edit contents
- Play button to load playlist into MPV
- Delete button for playlists
- Reorderable entries within playlist view (ion-reorder-group)
- Remove entries from saved playlists

**File: `src/components/fileBrowserModal.vue`**
- Multi-select mode with long-press to activate
- "Add to Queue" button - adds selected files to MPV's current queue
- "Add to Playlist" button with action sheet:
  - "Add to Existing Playlist" - shows list of saved playlists to choose from
  - "Create New Playlist" - prompts for name and creates new playlist with selected files
- Alert dialogs for playlist creation with name input

#### 4. Drawer Menu Updates & Standalone File Browser
- **Renamed "Playlist" to "Playlists"** in drawer menu
- **Added "File Browser" menu item** with folder icon for direct access

**File: `src/Container.vue`**
- Updated `appPages` array to change "Playlist" to "Playlists"
- Added new File Browser menu item with folderOutline icon

**File: `src/router/index.js`**
- Added route for `/folder/filebrowser`

**File: `src/views/FileBrowser.vue`** (NEW)
- Complete standalone File Browser page (not a modal)
- Menu button in header instead of back button
- All features from modal version: favorites filter, sorting, search, collections, multi-select
- Add to queue and add to playlists functionality
- Direct file playback integration
- Proper page navigation (no modal dismiss issues)

## Features Added

### 1. Display Control Buttons
Added two new buttons to the power menu in the app:
- **Turn Off Display** - Remotely turns off the monitor/display
- **Turn On Display** - Remotely wakes up the display

These buttons appear in the power action sheet alongside Shutdown, Reboot, and Quit options.

## Implementation Details

### Frontend Changes

**File: `src/views/Player.vue`**

Modified the `onShutdownClicked` function to add two new action sheet buttons:

```javascript
{
  role: "disable-display",
  text: "Turn Off Display",
},
{
  role: "enable-display",
  text: "Turn On Display",
}
```

These buttons make API calls to `/api/v1/computer/disable-display` and `/api/v1/computer/enable-display` respectively.

### Backend Changes

**File: `remoteServer.js` (in mpv-remote-node)**

#### 1. Display Off Command Function
Added `getDisplayOffCommand()` function that detects the desktop environment and returns the appropriate command:
- **KDE Plasma (Wayland/X11)**: Uses KDE's global shortcut via dbus
- **GNOME**: Uses GNOME ScreenSaver dbus interface
- **X11 fallback**: Uses `xset dpms force off`

#### 2. Display On Command Function
Added `getDisplayOnCommand()` function that wakes the display:
- **Wayland**: Uses ydotool wrapper script to simulate key press
- **X11**: Uses `xset dpms force on`

#### 3. Shutdown Action Updates
Added two new cases to the `shutdownAction()` function:
- `disable-display`: Executes display off command
- `enable-display`: Executes display on command

#### 4. API Endpoint Updates
Updated `/api/v1/computer/:action` endpoint to accept the new actions:
- `disable-display`
- `enable-display`

#### 5. Auto-start ydotoold (Wayland)
Added code in `main()` to detect Wayland and automatically start `ydotoold` daemon if not running.

## Platform Support

### Turn Off Display
- ✅ Linux X11 - Uses `xset dpms force off`
- ✅ Linux Wayland (KDE) - Uses KDE global shortcut via dbus
- ✅ Linux Wayland (GNOME) - Uses GNOME ScreenSaver dbus
- ❌ Windows - Not implemented
- ❌ macOS - Not implemented

### Turn On Display
- ✅ Linux X11 - Uses `xset dpms force on`
- ✅ Linux Wayland - Uses ydotool to simulate key press
- ❌ Windows - Not implemented
- ❌ macOS - Not implemented

## Wayland Setup Requirements

For display wake functionality on Wayland, the following setup is required:

### 1. Install ydotool
```bash
sudo pacman -S ydotool  # Arch/Manjaro
# or
sudo apt install ydotool  # Debian/Ubuntu
```

### 2. Create ydotool wrapper script
```bash
sudo tee /usr/local/bin/ydotool-wake << 'EOF'
#!/bin/bash
YDOTOOL_SOCKET=/tmp/.ydotool_socket /usr/bin/ydotool key 42:1 42:0
EOF

sudo chmod +x /usr/local/bin/ydotool-wake
```

### 3. Configure passwordless sudo
```bash
# For the wrapper script
echo "$USER ALL=(ALL) NOPASSWD: /usr/local/bin/ydotool-wake" | sudo tee /etc/sudoers.d/ydotool-wake
sudo chmod 440 /etc/sudoers.d/ydotool-wake

# For the daemon
echo "$USER ALL=(ALL) NOPASSWD: /usr/bin/ydotoold" | sudo tee /etc/sudoers.d/ydotoold
sudo chmod 440 /etc/sudoers.d/ydotoold
```

### 4. Start ydotoold daemon
```bash
sudo ydotoold
```

Or create a systemd service for auto-start (see mpv-remote-node/DISPLAY-CONTROL.md for details).

## Building the Android APK

### Prerequisites

**On ARM64 systems (like Rock 5B):**

1. **Install Java 17**
```bash
sudo pacman -S jre17-openjdk jdk17-openjdk
sudo archlinux-java set java-17-openjdk
```

2. **Install Android SDK Command-line Tools**
```bash
cd ~/
wget https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip
unzip commandlinetools-linux-13114758_latest.zip
mkdir -p ~/android-sdk/cmdline-tools
mv cmdline-tools ~/android-sdk/cmdline-tools/latest
```

3. **Set up environment variables** (add to ~/.bashrc):
```bash
export ANDROID_HOME=$HOME/android-sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

4. **Install SDK packages**
```bash
source ~/.bashrc
yes | sdkmanager --licenses
sdkmanager "platform-tools" "platforms;android-35" "build-tools;33.0.0"
```

5. **Download ARM64 Android build tools**
```bash
cd ~/
wget https://github.com/lzhiyong/android-sdk-tools/releases/download/35.0.2/android-sdk-tools-static-aarch64.zip
unzip android-sdk-tools-static-aarch64.zip
```

6. **Replace x86 tools with ARM64 versions**
```bash
cp -r ~/build-tools/* ~/android-sdk/build-tools/33.0.0/
cp -r ~/platform-tools/* ~/android-sdk/platform-tools/
chmod +x ~/android-sdk/build-tools/33.0.0/* ~/android-sdk/platform-tools/*

# Replace aapt2 in Gradle cache
find ~/.gradle/caches -name "aapt2" -type f -exec cp ~/build-tools/aapt2 {} \;
```

7. **Create local.properties**
```bash
echo "sdk.dir=$HOME/android-sdk" > /home/alarm/projects/mpv-remote-app/android/local.properties
```

### Build Steps

1. **Build the Vue app**
```bash
cd /home/alarm/projects/mpv-remote-app
npm run build
```

2. **Sync with Capacitor**
```bash
npx cap sync android
```

3. **Build the APK**
```bash
cd android
./gradlew clean assembleDebug
```

4. **Copy APK to home directory**
```bash
cp app/build/outputs/apk/debug/app-debug.apk ~/mpv-remote-display-control.apk
```

### Transfer APK to Another Device

From another device on the same network:
```bash
scp alarm@<IP-ADDRESS>:~/mpv-remote-display-control.apk .
```

## Known Issues & Temporary Fixes

### Issue 1: Capacitor 3 with Android API 35

**Problem:** Capacitor 3.4.3 is incompatible with Android API 35 due to deprecated `setAppCacheEnabled()` method.

**Temporary Fix:**
```bash
sed -i '421s/settings.setAppCacheEnabled(true);/\/\/ settings.setAppCacheEnabled(true); \/\/ Removed - deprecated in API 33+/' node_modules/@capacitor/android/capacitor/src/main/java/com/getcapacitor/Bridge.java
```

**Permanent Fix (Recommended):**
Upgrade to Capacitor 5 or higher:
```bash
npm install @capacitor/core@latest @capacitor/android@latest @capacitor/cli@latest --save
npm install @capacitor/app@latest @capacitor/haptics@latest @capacitor/keyboard@latest @capacitor/status-bar@latest @capacitor/storage@latest --save
npx cap sync android
```

Note: Upgrading Capacitor may require additional migration steps. See https://capacitorjs.com/docs/updating

### Issue 2: Cordova Framework 7.0.0 Not Found

**Problem:** jcenter repository was shut down and Cordova 7.0.0 is not available in Maven Central.

**Solution Applied:**
Updated `android/variables.gradle`:
```gradle
cordovaAndroidVersion = '10.1.1'
```

Changed all `jcenter()` references to `mavenCentral()` in:
- `android/build.gradle`
- `android/capacitor-cordova-android-plugins/build.gradle`
- `node_modules/@capacitor/android/capacitor/build.gradle`

### Issue 3: Android 12+ Exported Requirement

**Problem:** Apps targeting Android 12+ must explicitly specify `android:exported` for activities with intent filters.

**Solution Applied:**
Added `android:exported="true"` to MainActivity in `android/app/src/main/AndroidManifest.xml`:
```xml
<activity android:exported="true" android:configChanges="..." android:name="com.husudosu.mpvremote.MainActivity" ...>
```

## Current Build Configuration

- **minSdkVersion:** 24 (Android 7.0)
- **targetSdkVersion:** 35 (Android 15)
- **compileSdkVersion:** 35 (Android 15)
- **Capacitor:** 3.4.3 (with temporary workaround)
- **Cordova Android:** 10.1.1
- **Android Gradle Plugin:** 7.2.1

## Testing

The app has been tested on:
- ✅ KDE Plasma Wayland on ARM64 (Rock 5B)
- Display on/off functionality works correctly with ydotool setup

## Future Improvements

1. **Upgrade to Capacitor 5+** - Remove temporary workaround and get proper API 35 support
2. **Add Windows/macOS support** - Implement display control for other platforms
3. **Improve error handling** - Show user-friendly errors if ydotoold is not running
4. **Add display status detection** - Show current display state in UI
5. **Configuration options** - Allow users to customize display control commands

## Related Documentation

- Backend setup: `../mpv-remote-node/DISPLAY-CONTROL.md`
- Backend installation script: `../mpv-remote-node/install-display-control.sh`
- Backend README: `../mpv-remote-node/README.md`

## Credits

Display control feature implemented November 2025.

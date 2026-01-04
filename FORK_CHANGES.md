# Fork Changes

## v1.3.1 - Black Screen Fix (January 2026)

### Issue
On LibreELEC, pressing Back during playback resulted in:
- Black screen with no UI
- Debug logs showing `------ Window Init () ------` (empty window ID)
- Continuous `OutputPicture - timeout waiting for buffer` errors
- Orphaned player instance causing audio/video stream stalls

### Root Cause
The addon created an empty `backWindow` (xbmcgui.Window()) and showed it before starting playback. This window became the "previous window" in Kodi's navigation stack. When pressing Back from fullscreen video, Kodi would deinitialize VideoFullScreen.xml and attempt to activate this empty window, resulting in an invalid window state.

The empty window was originally added in v1.1.2 to "prevent Kodi from being displayed between Playlist Shuffles", but it caused unintended side effects with window management.

### Fix
1. **Removed the `backWindow` creation entirely** - No longer create or show an empty window
2. **Navigate to Home screen before starting playback** - This sets Home as the proper "previous window" in Kodi's navigation stack
3. **Explicitly stop player before script exit** - Prevents orphaned player instances that cause buffer timeouts
4. **Navigate back to Home screen during cleanup** - Ensures proper window state on exit

### Code Changes
- **Line 129-130**: Added `ActivateWindow(home)` before starting playback instead of creating backWindow
- **Line 58-60**: Removed callback-based navigation (not needed with new approach)
- **Line 210**: Removed backWindow.close() from no-episodes exit path
- **Line 344-353**: Added explicit player.stop() and navigation to Home in cleanup section

### Testing
Tested on LibreELEC with Kodi Matrix. Pressing Back now properly returns to Home screen without black screens or buffer errors.

### Technical Details
The issue occurred because:
1. `backWindow = xbmcgui.Window()` creates a window with no XML definition
2. `backWindow.show()` makes it the active window and adds it to navigation stack
3. Starting video playback makes VideoFullScreen.xml active, with backWindow as "previous"
4. Pressing Back triggers `CGUIWindowManager::PreviousWindow`
5. Kodi deinitializes VideoFullScreen and tries to activate backWindow
6. Since backWindow has no window ID or XML, it shows as `Window Init ()`
7. The player instance continues running but with no valid output window, causing buffer timeouts

By navigating to Home before playback, the navigation stack becomes:
1. Home → VideoFullScreen (instead of backWindow → VideoFullScreen)
2. Pressing Back properly returns to Home
3. No orphaned player, no buffer timeouts


# Limbo Tracker

Limbo is a time tracking extension for the Gnome Desktop Environment. It was designed for Developers working ad multiple orojects at once to help automatically keept track of work time. The extension allows you to create different Projects with a Regex to match against windows. While the currently active window matches the Regex a session runs. Optionally you can allow an extra timeout while another Window (like your browser is being used) and track all window name change events (typically containing the viewed files). All locally and fully transparent.

## Database dependencies

The extension uses GOM 1.0, libgda 5.0, and its SQLite provider through GObject Introspection. These are system dependencies, not npm packages. On Ubuntu or Debian, install them with:

```sh
sudo apt install gir1.2-gom-1.0 gir1.2-gda-5.0 libgda-5.0-4t64
```

On releases where the library package is named `libgda-5.0-4`, install that package instead. Other distributions need the equivalent GOM and libgda 5.0 introspection and SQLite provider packages.

The extension creates `$XDG_DATA_HOME/limbo-tracker/limbo-tracker.db` (defaulting to `~/.local/share/limbo-tracker/limbo-tracker.db`) on enable. Uninstalling or disabling the extension does not remove this data.

## Persistence

`Project`, `Session`, and `WindowEvent` are GOM resources. Construct them with `{ repository: database.repository }`, set their properties, and call `save_sync()` to create or update them. `database.getProjects()` loads projects with their sessions and window events. Related resources are saved separately.

Call `database.delete(resource)` to delete a resource; deleting a project also deletes its sessions and events, and deleting a session deletes its events. Do not use GOM's inherited `delete_sync()` or `delete_async()` for these GJS-defined resources: GOM accesses their properties from a worker thread, which GJS cannot safely execute. The generic delete method uses Gda to avoid that limitation. Existing databases are migrated on first open.

The extension opens the database and creates `LimboTracking` when enabled, then destroys tracking and closes the database when disabled. `LimboTracking.projects` contains the loaded projects, and `LimboTracking.runningSessions` contains their open sessions. Matching active window titles start sessions or refresh their last activity time. Active titles are saved as window events for open sessions. Each session and event is persisted as it changes.

When a project has no currently open matching window, its session ends immediately. Otherwise, an inactive session ends when its last activity is older than `Project.pauseAfter`; this value is interpreted in milliseconds, and an unset value disables pause expiry. Inactivity is checked once per minute.

## Window tracking

`WindowTracker` in `components/window-tracker.ts` exposes the read-only GObject properties `active-window-title` and `window-titles`. In JavaScript, read them as `active_window_title` and `window_titles`, or subscribe to `notify::active-window-title` and `notify::window-titles`. The active title is `null` when no tracked window is focused; the title list contains one entry per living window, including duplicate or empty titles. The list returned by the getter is a snapshot.

Create a tracker with `new WindowTracker()` inside GNOME Shell, and call `destroy()` when it is no longer needed to disconnect its signals. `LimboTracking` owns the tracker lifecycle and connects its active-title and window-list notifications to session tracking and persistence.

Run the tracker and session lifecycle tests with `npm run test:window-tracker` and `npm run test:limbo-tracking`.

Run the integration test after installing the system dependencies:

```sh
npm install
npm run test:database
```# limbo-tracker

# Plan

## Goal

A free, open-source iPhone and iPad app to view and control a Mac that has
Screen Sharing turned on, over the local network. Reaching the Mac from
elsewhere is left to a VPN (Tailscale, WireGuard, OpenVPN): the app connects to
an IP address or hostname and doesn't care how the packets get there.

The feature reference is Screens 5 by Edovia: connection list, Bonjour
discovery, trackpad and touch input modes, a keyboard bar. We match the
workflow, not its name, artwork or code.

## What the Mac speaks (verified)

Probed a Mac running macOS 27 with Screen Sharing on:

- RFB version `003.889`, Apple's variant of RFB 3.8, on TCP port 5900.
- Security types offered: `30, 33, 35, 36`.
  - **30, Apple Remote Desktop auth.** Diffie-Hellman key exchange, then the
    macOS username and password are sent AES-encrypted. Documented in the
    community RFB spec and implemented by libvnc and RoyalVNCKit. This is what
    we use.
  - 33, 35, 36 are Apple-proprietary types used by Apple's own Screen Sharing
    app. Undocumented; out of scope.
  - 2 (VNC password) is only offered when "VNC viewers may control screen with
    password" is turned on in Sharing settings. Supported as a fallback.
- The Mac advertises itself over Bonjour as `_rfb._tcp`, so the app can list
  Macs on the network without typing an IP.

**Security note.** Type 30 protects the login only. After authentication,
screen pixels, keystrokes and clipboard travel as plain RFB. That is acceptable
on a home LAN; on any other network, connect through Tailscale or WireGuard,
which encrypt the tunnel.

## Approach

Build our own app on top of RoyalVNCKit.

| Option | Verdict |
| --- | --- |
| [RoyalVNCKit](https://github.com/royalapplications/royalvnc) (Swift, MIT, maintained by Royal Apps) | **Chosen.** Already handles ARD auth, VNC password, Tight/ZRLE/Zlib/Hextile/CopyRect, cursor and desktop-size pseudo-encodings, and clipboard text. Its iOS view is unfinished, but we write our own view layer anyway. |
| LibVNCClient (C, GPL-2.0) | Proven, but needs cross-compiling plus OpenSSL or libgcrypt for iOS, and forces the GPL. |
| Write RFB from scratch in Swift | Full control, but weeks re-implementing what RoyalVNCKit has. Fallback if RoyalVNCKit blocks us. |

Stack:

- Swift 6. SwiftUI for app chrome; UIKit for the remote view (gestures,
  hardware keyboard, pointer lock).
- SwiftData for saved connections, Keychain for passwords.
- Network.framework `NWBrowser` for Bonjour discovery.
- XcodeGen generates the Xcode project from `project.yml`, so the repo holds no
  `.xcodeproj`.
- Minimum iOS/iPadOS 18.
- MIT license, compatible with RoyalVNCKit and its dependencies.

## Layout

```
project.yml                  XcodeGen spec (app + tests)
Config/Local.xcconfig        your DEVELOPMENT_TEAM (git-ignored)
Config/Local.xcconfig.example
App/                         SwiftUI app: connection list, editor, settings
Packages/ScreensKit/         Swift package; builds and tests on macOS
  Sources/ScreensKit/
    Connections/             model, SwiftData store, Keychain
    Discovery/               Bonjour browser for _rfb._tcp
    Session/                 wraps RoyalVNCKit: connect, auth, reconnect, frames
    Input/                   gestures to pointer events, keys to X11 keysyms
  Sources/screens-probe/     CLI: connect, authenticate, save a screenshot
  Tests/
```

## Milestones

### M0: Protocol spike (needs only the Command Line Tools)

- A `screens-probe` CLI connects to the Mac with ARD auth via RoyalVNCKit, logs
  the encodings the server picks, saves one frame as PNG, and moves the cursor.
- Credentials come from the macOS Keychain, never from command-line arguments.
- Done when a PNG of the Mac's screen exists and the cursor moved.

### M1: Running on iPhone and iPad

- XcodeGen project, placeholder icon, signing via `Config/Local.xcconfig`.
- Add a connection by host, port, username and password; connect; render the
  screen; tap to click; pinch to zoom and pan; type with the on-screen
  keyboard.
- Build and install from the command line with `xcodebuild` and
  `xcrun devicectl`.
- Done when the Mac can be controlled from both devices over Wi-Fi.

### M2: Everyday use

- Sidebar like the Mac app: All Connections and Network (Bonjour). Grid/list
  toggle, search, last-connected time, a thumbnail of the last frame.
- Touch input in two modes: trackpad (relative cursor) and touch (tap where
  you want to click). Two-finger tap for right click, two-finger drag to
  scroll, tap-and-hold to drag.
- Keyboard bar: sticky ⌘ ⌥ ⌃ ⇧, Esc, Tab, arrows, F-keys, common shortcuts.
- Hardware keyboards on both devices; on iPad, trackpad or mouse with pointer
  lock in full screen.
- Clipboard text in both directions.
- Reconnect when the app returns from the background, since iOS drops sockets
  while the app is suspended.

### M3: Polish

- Display picker for Macs with several monitors (crop the combined
  framebuffer).
- Quality presets (encoding, colour depth, JPEG level) for Wi-Fi versus VPN.
- Face ID to unlock saved passwords.
- Wake-on-LAN.
- `vnc://` URL handling; multiple windows and Stage Manager on iPad.

### M4: Open-source hygiene

- GitHub Actions: build the app and run ScreensKit tests on a macOS runner.
- README: build and sideload guide, Mac setup.
- Optional: TestFlight or an AltStore source if a paid developer account is
  used.

## Out of scope

- Features that need Apple's private protocol: encrypted sessions (types
  33/35/36), High Performance mode, curtain mode, file transfer.
- A relay or cloud service: use a VPN.
- Windows and Linux hosts should work through any VNC server, but only macOS
  is tested.

## Installing on devices

- Needs full Xcode, not just the Command Line Tools, and an Apple ID added
  under Xcode → Settings → Accounts.
- With a free Apple ID (Personal Team), the app expires after 7 days and must
  be reinstalled; at most 3 sideloaded apps per device.
- With the Apple Developer Program ($99 a year), profiles last a year and
  TestFlight is available.
- First install: connect over USB, trust the computer, and turn on Developer
  Mode (Settings → Privacy & Security). After that, installs can go over Wi-Fi.
- iOS asks for Local Network permission on first launch; it's needed for
  Bonjour and LAN connections.

## Risks

| Risk | Mitigation |
| --- | --- |
| A future macOS drops security type 30 | Fall back to VNC password (type 2). |
| RoyalVNCKit rough edges on iOS, or slow rendering of Retina-size framebuffers | We own the view layer; move to a Metal renderer if CALayer is too slow; fix upstream or fork. |
| Unencrypted session on the LAN | Documented; use a VPN away from home. |
| iOS can't capture some system shortcuts (⌘Tab, ⌘Space, Globe) | Offer them on the keyboard bar. |
| "Screens" is Edovia's product name | Fine as a working name for a personal repo; pick another display name before any App Store or TestFlight release. |

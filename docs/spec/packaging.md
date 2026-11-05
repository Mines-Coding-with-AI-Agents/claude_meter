# Packaging

Status: Draft.
Related decisions: D-007.

## Purpose

Build, sign, and launch the app on the owner's Mac using only the Command Line Tools, without full Xcode.

## Requirements

- PKG-1: The project is a Swift package with a `Core` library target, an app executable target, and a test target.
- PKG-2: The build must work with `swift build -c release` and the tests with `swift test`, with no Xcode project.
- PKG-3: A script must assemble the `.app` bundle from the build output, with a checked-in `Info.plist` that sets `LSUIElement` to true, a bundle identifier, and the minimum macOS version.
- PKG-4: The script must sign the bundle.
  It must use a stable signing identity when one is available and fall back to ad-hoc signing otherwise, and it must print which one it used.
- PKG-5: Launch at login must use `SMAppService.mainApp` and must be controlled from settings (UI-13).
  If registration fails, for example outside a real `.app`, the app must show the error and not crash.
- PKG-6: The build script must be idempotent and must fail loudly, with a non-zero exit code and a clear message.
- PKG-7: The app must run only one instance.
  A second launch must activate the first.
- PKG-8: The repository must contain no secrets and no signing identities.
- PKG-9: Notarization and distribution are out of scope (N6).

## Acceptance criteria

- Given a clean checkout with only the Command Line Tools, when the build script runs, then a runnable `.app` is produced.
- Given the `.app` is launched, when no other instance exists, then a menu bar item appears and no Dock icon appears.
- Given ad-hoc signing, when the app is rebuilt, then the build output says ad-hoc signing was used.
- Given launch at login is enabled in settings, when the Mac restarts, then the app starts.

## Open

- O-1: Minimum macOS version.
  The owner's Mac runs a recent release.
  Decide the lowest version worth supporting.
- O-2: Does ad-hoc signing keep notification permission and launch at login across rebuilds?
  If not, a development signing identity is needed.
- O-3: How the app finds `claude` when launched at login, before the owner's shell environment exists (PRB-4).

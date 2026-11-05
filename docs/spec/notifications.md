# Notifications

Status: Draft.
Related decisions: D-007, D-008, D-010.

## Purpose

Deliver alerts to the owner reliably and only once.

## Requirements

- NTF-1: Notifications must be delivered through the macOS notification center (`UNUserNotificationCenter`).
- NTF-2: The app must check the current notification settings before requesting permission, and request permission only if it has not been decided.
- NTF-3: If permission is denied, the app must still show alerts as a banner inside the popover and must tell the owner how to enable notifications.
- NTF-4: Each alert must be delivered at most once per alert key.
  The alert key and the delivery time must be saved to `alert-state.json` before or when the notification is sent, so a restart does not repeat it (SC3).
- NTF-5: Alert state for a window must be cleared when that window ends, except for cooldown timestamps that are still in the future.
- NTF-6: A notification has a title (the app name), a body (the rendered rule message), and the rule `id` in its user info.
- NTF-7: Clicking a notification must open the popover.
- NTF-8: The app must not deliver a notification while the most recent reading is stale, except for `stale-data` and `reset` alerts (RUL-2, RUL-7).
- NTF-9: Notification delivery must go through an injected notifier interface, so tests never post real notifications.
- NTF-10: The popover must keep a list of the last 20 alerts with time and message, whether or not the notification was delivered.

## Acceptance criteria

- Given an alert fires and the app restarts, when the same condition still holds, then no second notification is sent.
- Given permission was denied, when an alert fires, then the popover shows it and no system notification is attempted.
- Given a notification is clicked, when the app is in the background, then the popover opens.
- Given the permission state is undecided, when the first alert fires, then permission is requested once.

## Open

- O-1: Should high-severity alerts use a time-sensitive delivery level that can break through Focus modes?
- O-2: Does notification delivery work with ad-hoc signing (`packaging.md` PKG-4)?
  Test early.
- O-3: Should there be a global mute or snooze, for example "snooze alerts for 4 hours"?

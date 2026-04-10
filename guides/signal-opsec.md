# Signal OpSec — What They Don't Tell You

> The FBI recovered deleted Signal messages from a seized iPhone in April 2026.  
> Signal was not broken. The OS was the problem.  
> Here's what you need to know.

---

## What happened

During the Prairieland ICE Detention Facility trial, FBI Special Agent Clark Wiethorn testified that investigators extracted Signal message content from a defendant's iPhone — **even after the app had been completely deleted**.

The method: iOS caches push notification previews in an internal database. Those previews persisted after Signal was removed. The FBI used forensic extraction software on the physical device to recover them.

Signal's encryption was never broken. The vulnerability was in how iOS handles notifications.

**Source:** [404 Media, April 9 2026](https://www.404media.co/fbi-extracts-suspects-deleted-signal-messages-saved-in-iphone-notification-database-2/)

---

## The fix (do this now)

Open Signal → **Settings → Notifications → Show** → select **"No Name or Content"**

This prevents iOS from storing message previews in its notification database. No preview, nothing to extract.

Do the same for every messaging app you care about.

---

## What was actually recovered

Only **incoming messages** were recoverable through this method. Outgoing messages were not found via the notification cache.

This matters for your threat model — but don't rely on it. The notification database is one vector. There are others.

---

## The real lesson: app-level encryption is not enough

Signal encrypts messages in transit. That's solid. But once a message lands on your device, it enters the OS ecosystem — and the OS has its own rules.

On a standard iPhone:
- iOS controls the notification system
- iOS controls the filesystem
- Apple controls iOS
- Apple complies with lawful requests

This is not a Signal problem. It's a platform problem.

---

## iOS vs GrapheneOS

| Feature | iOS | GrapheneOS |
|---------|-----|------------|
| Notification caching | System-controlled | User-controlled |
| App sandboxing | Strong but closed | Strong and auditable |
| Compliance with legal demands | Yes | Minimal surface |
| Push notification dependency | Apple APNs | Optional / self-hosted |
| Open source | No | Yes |

GrapheneOS on a Pixel device removes the Apple layer entirely. Signal on GrapheneOS with notifications disabled via the OS gives you significantly better isolation.

This is not for everyone. But if your threat model includes physical device seizure, it's worth considering.

---

## Metadata leaks Signal doesn't fix

Even with perfect notification settings, Signal leaks metadata at the network level:

- **Who you talk to** — Signal knows your phone number and your contacts' phone numbers
- **When you communicate** — connection timestamps are logged by your ISP
- **That you use Signal** — visible to your network provider

Mitigations:
- Use Signal over Tor (via Orbot on Android)
- Use a number not tied to your identity (eSIM purchased with cash, or a VoIP number)
- On GrapheneOS, disable Google Play Services and use Signal's built-in UnifiedPush support

---

## Disappearing messages — use them

Signal's disappearing messages feature deletes message content from both devices after a set timer. This reduces the window of exposure.

Recommended settings:
- Default timer: **1 week** for most conversations
- Sensitive conversations: **1 day or less**
- Enable **"Set as default"** so new conversations inherit the timer

This does not help if your device is seized before the timer expires. It helps against longer-term exposure.

---

## Full checklist

- [ ] Signal → Settings → Notifications → **No Name or Content**
- [ ] Enable disappearing messages by default
- [ ] Use Signal over Tor if your threat model requires it
- [ ] Consider GrapheneOS if physical device seizure is a realistic threat
- [ ] Use a phone number not tied to your real identity for sensitive use
- [ ] Lock screen notifications off for all sensitive apps at the OS level

---

## What Signal does right

This guide is not an argument against Signal. Signal remains one of the best options for encrypted messaging:

- Open source, audited cryptography
- Minimal metadata retention on their servers
- Sealed sender hides who is messaging whom
- Forward secrecy — past messages stay encrypted even if your key is compromised

The point is that Signal is a tool, not a guarantee. Your threat model determines whether the tool is sufficient.

---

## Bottom line

The FBI did not break Signal. They extracted notification previews from iOS.

Fix your notification settings. Understand your platform. Know your threat model.

*App-level encryption means nothing if the OS is the adversary.*

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*

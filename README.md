# DAB

**[Download the latest APK →](https://github.com/Mild-Solvent/DAB/releases/latest/download/DAB.apk)** ·
**[Website →](https://mild-solvent.github.io/DAB/)**

A friends-only map for Android. The world starts covered in fog and uncovers as you
walk it; your friends show up on it, and you can message and call them. There is no
DAB server, no account and no analytics — phones talk to each other directly, and
fall back to free public relays that only ever see ciphertext.

- **Fog of war** — explored 180 m cells are stored on the phone, nowhere else.
- **Friends, live or last-seen** — live positions stream device-to-device over WebRTC;
  when a friend is offline you still see their last known position from an encrypted
  replaceable event.
- **Encrypted chat, photos and files** — messages live on the two phones only.
- **Audio and video calls** — DTLS-SRTP, STUN only. If a direct path cannot be made,
  the app says so rather than relaying you through an unapproved server.
- **Offline maps** — download the visible area and the map keeps working with no
  connectivity. Tiles from [OpenFreeMap](https://openfreemap.org), no API key.
- **Text-and-calls mode** — a hard switch: the map never mounts and GPS is never requested.

## Install

1. Open [the latest release](https://github.com/Mild-Solvent/DAB/releases/latest) on
   the phone and download `DAB.apk`.
2. Allow your browser to install unknown apps (Settings → Apps → your browser →
   *Install unknown apps*).
3. Open the downloaded file and install.

**Updating:** download the newer APK and install it over the old one. Every release is
signed with the same key, so friends, chats and the explored map survive the update —
no uninstall needed.

Requires Android 8+. iOS is supported by the source but is not distributed here.

## Honest limits

DAB is alpha software written for a handful of friends. It has not had a security
review. The APK is signed with a development key, so Android will flag it as coming
from an unknown developer. Relays and call peers can observe your IP address and the
timing of your traffic — peer-to-peer cannot hide that.

[`PRIVACY.md`](PRIVACY.md) is a plain-language account of what the app keeps private
and what it exposes, written from reading the code rather than from the design intent.

## This repository

This repo is the public front door: the website (GitHub Pages, served from
[`index.html`](index.html)) and the APK releases. The application source lives
separately.

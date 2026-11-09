# UTF-8 clipboard image overlay

This overlay runs with the official Rakazo app and supervisor. Its build starts
from the official Bookworm Computer image at `46e95fa` and recompiles the installed
Debian x11vnc source package. The final stage contains the rebuilt x11vnc binary
and the patched noVNC module.

## Build

From the repository root, run the following command.

```sh
docker build -t rakazo-computer:unicode infra/sandboxes/computer/utf8-clipboard
```

The final stage is named `computer-clipboard`. A downstream Dockerfile can append
`FROM computer-clipboard` to add its document and media tools. The three build
files are `Dockerfile`, `x11vnc-utf8.patch` and `novnc-provide.patch`.

Use the resulting image through Rakazo's existing `RAKAZO_COMPUTER_IMAGE` setting.

## Implementation

- x11vnc connects LibVNCServer's UTF-8 clipboard callback to the image's existing
  `xclip` utility. xclip owns PRIMARY and CLIPBOARD according to x11vnc's selection
  settings. Each selection has one current xclip process, which exits when its
  ownership is replaced. Existing view-only and allowed-input checks run first.
  The patch initializes the Unix socket accept length, which otherwise caused an
  intermittent connection failure in the rebuilt server.
- Desktop selections are requested as UTF8_STRING, with XA_STRING conversion for
  legacy owners. Outgoing Extended Clipboard data uses CRLF wire newlines and a
  Latin-1 fallback for classic RFB clients. Ownership notifications invalidate the
  outgoing cache so copying the same remote text again still updates the viewer.
- noVNC sends an unsolicited UTF-8 Provide when the server advertises that action
  and a sufficient byte limit. The stock bridge then sends Ctrl+V on the same
  ordered connection.

`X11VNC_AVOID_WINDOWS=never` initializes selection ownership without waiting for a
login manager. The Computer's Xvfb session starts directly with its window manager.

## Verification and ablation

Verification on 2026-11-09 used an isolated Docker container, the official app's
screen startup commands, and a real noVNC browser connected to a Chromium textarea.
Playwright sent actual Ctrl+V and Ctrl+C events. Input covered ASCII, Latin-1,
Chinese, emoji, multilingual multiline text and 8192 Unicode characters. Additional
checks covered repeated copying, classic RFB input and forced clipboard/key writes
through the server's view-only endpoint.

| Variant | Host to desktop Unicode | Desktop to host Unicode |
| --- | --- | --- |
| Original image | Fail | Fail |
| x11vnc patch alone | Fail | Pass |
| noVNC patch alone | Fail | Fail |
| Both patches with callback and queue | Pass | Pass |
| Both patches with queue removed | Pass | Pass |
| Both patches with callback removed and stock bridge | Pass | Pass |

The selected image uses both patches and the stock clipboard bridge. The normal
input cases also pass after removing the completion callback and paste queue.

Rapid successive pastes can overlap and lose content. Timing-dependent successful
runs do not establish ordering guarantees. The minimal implementation accepts this
limit and relies on the ordinary X11 clipboard behavior.

The tested base contains x11vnc 0.9.16-9, LibVNCServer 0.9.14+dfsg-1+deb12u2 and
noVNC 1.3.0. Debian's source package supplies its existing distribution patches
and build dependencies. Compiler packages stay in the builder stage. Keep the
app release and the overlay base release aligned when upgrading.

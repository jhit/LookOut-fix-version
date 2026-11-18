# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

LookOut (fix version) is a Thunderbird MailExtension (manifest v2, Thunderbird 140–158) that decodes TNEF attachments (`winmail.dat`, `application/ms-tnef`) into regular attachments, calendar invitations (ICS) and, optionally, the original HTML body. All source lives in `src/`; there is no package.json, test suite or linter.

## Build / run

- Build the XPI: `cd src && ./build.sh` (Linux; needs `7z`/`7za`, prompts before deleting an existing `lookout_fix_version-beta-tb.xpi`). `build.bat` is the Windows equivalent. The file list is hard-coded in both scripts — add new top-level files/dirs there or they won't ship.
- Run/test manually: in Thunderbird use Tools → Developer Tools → Debug Add-ons → "Load Temporary Add-on" and select `src/manifest.json`, then open a message with a winmail.dat. Re-open the message after changing debug settings. Debug output goes to the Error Console (enabled in add-on options).
- Release bookkeeping: bump `version` (and `strict_max_version` for new TB releases) in `src/manifest.json` and add an entry at the top of `src/changes.txt`.

## Architecture

Flow (`src/background.js`): `onMessageDisplayed` (and already-open tabs at startup) → `handleMessage(tab, message)`:
1. Skips junk messages and shows a banner via `message-content-script.js` (registered with `scripting.messageDisplay`).
2. Lists attachments, fetches each candidate as a `File` via the **Attachment experiment API**, and parses it with `TnefExtractor` (`scripts/lookout.mjs`).
3. Decoded files get part names `<origPart>.<n>`; then, depending on prefs, the winmail.dat is removed, decoded files are added back as attachments, `body_part_0.html` replaces the displayed body (sent to the content script, which uses `setHTML`; charset decoded in `decodeTnefBody`), and `text/calendar` files are handed to Thunderbird's iMIP bar.

Key pieces:
- `scripts/tnef.mjs` — the core TNEF/MAPI/OLE2/RTF decoder (ported from the C TNEF project). It reads from a stream-like object and reports files through a listener (`onTnefFile`). `lookout.mjs` provides `PseudoInputStream`, a partial `nsIInputStream` over a `File`, so the decoder runs in the background script rather than privileged code. Includes OLE-embedded attachment unwrapping (ATTACH_METHOD 6 → images).
- `scripts/mapi_props.js` — MAPI property tables; loaded as a classic script in `background.html` (before the module `background.js`), not imported.
- `src/api/Attachment/` — a Thunderbird **Experiment API** (privileged `implementation.js` + `schema.json`) that manipulates the message window's attachment list (list/get/remove/add attachments, `handleCalendarInvitation`). Anything that needs chrome-privileged access goes here; schema and implementation must be kept in sync.
- `src/api/LegacyPrefs/` — vendored experiment for reading old `about:config` prefs; used only by `scripts/storage.mjs` `migratePrefs()` to move them into `storage.local`. Prefs are now read via `storage.getPrefs()`; the options UI is `src/options/`.
- `src/_locales/<lang>/messages.json` — i18n strings (en-US is default); `scripts/i18n.js` localizes the options page.

Version history and notes about LLM-assisted features are in `src/changes.txt`. Upstream FAQ/wiki: https://github.com/TB-throwback/LookOut-fix-version/wiki/F.A.Q.

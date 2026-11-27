# Passyble browser extension

Manifest V3 extension for Chrome, Edge, Brave, Opera, Opera GX, Firefox and Safari, from one source (what is Firefox's or
Safari's own lives in [`../Firefox/`](https://github.com/sensyble/PassybleFirefox/blob/main/README.md) and [`../Safari/`](https://github.com/sensyble/Passyble-Safari/blob/main/README.md)). It fills passwords, TOTP 2FA codes, passkeys,
payment cards and addresses, in one of two modes per browser profile ([The vault in this browser](#the-vault-in-this-browser-browser-mode)):

## Repository

The Passyble browser extension (Manifest V3) shared by Chrome, Edge, Firefox and Safari.

This is one of the Passyble repositories (Core, Windows, macOS, iOS, Android, Extension, Firefox, Safari, Server, Website). They are cloned side by side into one folder, each under its own name, so the relative paths between them (`../Core/core`, `../../Extension`, ...) work.

Clone next to this one:

* [PassybleCore](https://github.com/sensyble/PassybleCore): the browser vault's WebAssembly (scripts/build-wasm.mjs) and test vectors
* [PassybleFirefox](https://github.com/sensyble/PassybleFirefox): the Firefox manifest (scripts/package.mjs, scripts/make-firefox-manifest.mjs)

```sh
# One folder for all Passyble repositories, for example ~/src/Passyble
mkdir Passyble && cd Passyble
git clone https://github.com/sensyble/PassybleExtension.git Extension
git clone https://github.com/sensyble/PassybleCore.git Core
git clone https://github.com/sensyble/PassybleFirefox.git Firefox
```

Test (from this repository's root):

```sh
npm install && npm test                      # unit tests (a stand-in replaces the WebAssembly)
npm run wasm                                 # the browser vault's WebAssembly, built in ../Core
npm run package                              # store packages (needs the WebAssembly)
```

Rust builds run in the sibling Core repository; on Windows run cargo there at idle priority through `..\Core\scripts\cargo-low.cmd` (see CONTRIBUTING.md).
To build the WebAssembly from another Passyble-Core checkout, such as a sibling worktree, name it:
`npm run wasm -- --core ../Core-sec`, or set `PASSYBLE_CORE_DIR` (a relative path counts from this repository's folder).


Whenever the Passyble app is installed on the computer, the extension uses it: app mode, on its own, and the vault
in this browser isn't offered ([The Passyble app always wins](#the-passyble-app-always-wins)).

* **The Passyble app on this PC** (app mode, the default): it talks
  only to the native messaging host `com.sensyble.pass`, using the methods in
  [`Core/docs/PROTOCOL.md` §2](https://github.com/sensyble/PassybleCore/blob/main/docs/PROTOCOL.md). The vault stays in the desktop agent. The extension never
  even loads the vault's WebAssembly then.
* **The vault in this browser** (browser mode, Chrome, Edge, Brave and Firefox on a computer without the Passyble app): the
  desktop agent's own request handlers and pass-core, compiled to WebAssembly (`wasm/`, built into
  `src/wasm/`), hold the vault in the extension, encrypted in its IndexedDB. Such a vault is only on this
  device: the pages say so ("This vault is only on this device", see below) and offer a copy to download.
  **Google Drive sync needs the Passyble app on this computer**: the app signs in to Google and syncs, and
  the extension uses the app's vaults (app mode). The extension never signs in to Google itself.

In app mode the extension only ever holds the one value it is filling, a password it generated for a sign-up form
until that form is submitted (and, for 30 minutes, a SHA-256 of it with the page's origin, so the submit of a
password Passyble generated is saved without asking), a just-typed password for up to 60 s while it asks "Save login to
Passyble?", a pasted Passyble Server pairing link only while handing it to the desktop app, the login open in
the popup's editor while it is open, and the one card or identity it is filling into a form.

Plain JavaScript ES modules: no build step and no runtime dependencies. `jsdom` is a dev dependency
used only by the tests. The UI icons are vendored [Lucide](https://lucide.dev) SVG data (ISC) and the UI font is
Space Grotesk (SIL OFL 1.1); see [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md), which ships in the package.

## Load it unpacked

### Chrome / Edge

0. Build the browser vault's WebAssembly once (`npm run wasm`, needs Rust; see [Development](#development)),
   or unzip `pass-chromium-unpacked.zip` (from `npm run release`) and load that folder instead: the background imports
   `src/wasm/` and doesn't start without it, in either mode.
1. Open `chrome://extensions` (or `edge://extensions`) and turn on **Developer mode**.
2. Click **Load unpacked** and select this `Extension/` directory.
3. The card shows the ID `bmjmlhhgfbpcbfbkbndlmmockkbdcaoo` on every machine, because `manifest.json`
   pins it with a `key` (see [Stable extension ID](#stable-extension-id)). The desktop app allows this
   development ID, so no `extra_extension_ids` entry is needed. The Chrome Web Store and Edge Add-ons assign
   their own IDs, which are not allowed yet (see [Stable extension ID](#stable-extension-id)).

Chrome 111+ / Edge 111+ is required (`"world": "MAIN"` content scripts).

### Opera / Opera GX

Opera is Chromium and runs the Chromium package unchanged. People install it in one of three ways:

* **From the Chrome Web Store**: Opera installs Chrome Web Store extensions itself (the store shows
  **Add to Opera**). The extension then has the Chrome Web Store ID, which goes into
  `CHROME_STORE_EXTENSION_IDS` once assigned.
* **From Opera Add-ons** (addons.opera.com), once it is listed there ([Opera Add-ons](#opera-add-ons)). That
  copy has the ID Opera Add-ons assigns, which is in `OPERA_STORE_EXTENSION_IDS`.
* **Unpacked**, for development: `opera://extensions` → **Developer mode** → **Load unpacked** → this
  folder (or the unzipped `pass-chromium-unpacked.zip`). The `key` pins the same ID as in Chrome,
  `bmjmlhhgfbpcbfbkbndlmmockkbdcaoo`.

Pin it with the cube (Extensions) in the toolbar, then the pin next to Passyble; the setup guide says so in
Opera's words. The desktop app connects Opera and Opera GX like Chrome (see
[Native messaging host registration](#native-messaging-host-registration)). What differs in Opera:

* Opera documents only `launchWebAuthFlow` from the identity API, so `web-auth.js` builds the
  `https://<id>.chromiumapp.org/` redirect from `runtime.id` where `identity.getRedirectURL` is missing.
* `storage.sync` and `i18n.getUILanguage` are not supported; the code already falls back to `storage.local`
  and English.
* Opera has its own password manager and autofill; Passyble turns them off through `privacy.services` as in
  Chrome (Opera lists no difference for the `privacy` API). Check it after a first install.
* The sidebar (`sidebar_action`) is Opera's own manifest key; Passyble does not use it.
* Snap and Flatpak Opera on Linux can't start a native host (as for every browser).

### Firefox (140+)

Firefox gets its own manifest (an event page instead of a service worker, the gecko ID, the data consent); the
differences are data in [`../Firefox/manifest.overrides.json`](https://github.com/sensyble/PassybleFirefox/blob/main/manifest.overrides.json). Build a loadable
copy and the AMO packages there:

```sh
cd ../Firefox && npm run package    # dist/unpacked/, dist/pass-firefox.zip, dist/pass-firefox-source.zip
```

Then open `about:debugging#/runtime/this-firefox`, click **Load Temporary Add-on…** and pick
`Firefox/dist/unpacked/manifest.json`. The add-on ID is `pass@sensyble.com` (the `FIREFOX_EXTENSION_ID` the
desktop app puts in `allowed_extensions`). `node scripts/make-firefox-manifest.mjs --dist dist/firefox` still
writes the same loadable copy here, and without arguments only `manifest.firefox.json`.

On Firefox, check that "Access your data for all websites" is granted in `about:addons` → Passyble →
Permissions. Firefox treats MV3 host permissions as opt-in; the setup guide (below) then shows an
**Allow access** step. What Firefox can't do: [`../Firefox/README.md`](https://github.com/sensyble/PassybleFirefox/blob/main/README.md#what-firefox-cant-do).

### Safari

A Safari Web Extension in an Xcode project, built on macOS only: [`../Safari/README.md`](https://github.com/sensyble/Passyble-Safari/blob/main/README.md).

## Publishing to the stores

```sh
npm run release    # npm run wasm, then npm run package
```

`npm run package` alone packages what is there and stops with a message when `src/wasm/` isn't built
(a package without it would not start). `npm run release` builds the WebAssembly and packages. It
writes three zips to `dist/` (git-ignored):

| file | for | manifest |
|---|---|---|
| `dist/pass-chromium-unpacked.zip` | unzip, then "Load unpacked" in Chrome, Edge or Brave | `manifest.json` **with `key`**, so the extension gets the pinned ID the native host allows (the store zip loaded unpacked gets an ID from its folder path, and the browser can't reach Passyble) |
| `dist/pass-chromium.zip` | Chrome Web Store, Edge Add-ons | `manifest.json` **without `key`** (the stores reject it and assign their own ID; add that ID to `CHROME_STORE_EXTENSION_IDS` or `EDGE_STORE_EXTENSION_IDS`, see [Stable extension ID](#stable-extension-id)) |
| `dist/pass-firefox.zip` | addons.mozilla.org | the Firefox variant from `../Firefox/` (gecko ID, no `key`, no `offscreen`); `Firefox/scripts/package.mjs` also writes the AMO source zip |

`npm run package:opera` (`scripts/package-opera.mjs`) writes two more:

| file | for | contents |
|---|---|---|
| `dist/pass-opera.zip` | Opera Add-ons (Opera and Opera GX) | byte for byte `pass-chromium.zip`: the store manifest **without `key`** |
| `dist/pass-opera-source.zip` | Opera's reviewers | what the WebAssembly is built from (Core's `core`, `agent`, `wasm` crates, `Cargo.lock`, a cut-down workspace `Cargo.toml`), this extension, and `BUILDING.md` with the tool versions and steps |

It stops when a locale's title says "Opera" (Opera's acceptance criteria forbid it).

Both hold `manifest.json` at the top level plus `src/`, `icons/`, `_locales/` (translations) and
`THIRD_PARTY_NOTICES.md`; `test/`,
`scripts/`, `node_modules/`, `dist/`, dotfiles, `package*.json` and this README are left out. The
packager is plain Node (zlib and a minimal zip writer, no dependencies, works on Windows) and the zips
are reproducible. Before zipping, it checks that every `__MSG_…__` name in the manifest is in
`_locales/en/messages.json`. Bump `version` in `manifest.json` before each upload; the stores require a higher one.
The store listing's name and short description come from `extName` / `extDescription` in each locale's
`messages.json` (Chrome: name at most 75 characters, description at most 132; `npm test` checks both).

Permissions to justify in the store forms: the earlier set plus `identity` (Sign in to an existing Passyble
account on account.passyble.com with `launchWebAuthFlow`; no install warning). The extension never signs in to
Google: Google Drive sync is the Passyble desktop app's, so the packages carry no Google OAuth client and the
privacy answers list no Google service. The listings and privacy answers must no longer say the extension makes
no network requests (see Security model below).

### Opera Add-ons

Opera's acceptance criteria (help.opera.com/en/extensions/acceptance-criteria/): code must be readable (not
minified or obfuscated; third-party code unchanged), no remote JavaScript, no unused files, no redundant
permissions, one clear purpose, no "Opera" in the title, and the extension is tested on Windows, macOS and
Linux. Reviews can take weeks. To submit:

1. `npm run wasm`, bump `version` in `manifest.json` if it was uploaded before, then `npm run package:opera`.
2. Sign in to the Opera Add-ons developer portal (addons.opera.com/developer/) with the publisher's Opera
   account and upload `dist/pass-opera.zip` as a new extension.
3. Fill in the listing from `_locales/*/messages.json` (`extName`, `extDescription`), the icons and
   screenshots, the privacy policy URL and the support URL.
4. In the notes for the reviewers, say that `src/wasm/` is compiled from Rust, attach or link
   `dist/pass-opera-source.zip` (its `BUILDING.md` has the steps), and justify each permission (the same list
   as for the Chrome Web Store above; `nativeMessaging` talks to the Passyble desktop app).
5. Once it is published, copy the ID from its addons.opera.com page and:
   * check it is in `OPERA_STORE_EXTENSION_IDS` in `Core/desktop-app/src-tauri/src/native_messaging.rs`
     (currently `elckicaofoomkdcfbkkbnjdljhbbmkch`), and release the desktop app, so the native host allows it;
   * add it to `[apps] extension_ids` of the account service (Accounts) if that list is set (empty accepts
     any well-formed ID), so the account sign-in's `https://<id>.chromiumapp.org/` redirect is allowed.
6. Updates: bump `version`, `npm run package:opera`, upload the new `pass-opera.zip` (and the source zip
   when the WebAssembly changed).

## Native messaging host registration

The desktop installer registers the host. This section describes what it has to do.

Chrome / Edge host manifest (`com.sensyble.pass.json`):

```json
{
  "name": "com.sensyble.pass",
  "description": "Passyble password manager",
  "path": "C:\\Program Files\\Passyble\\pass-native-host.exe",
  "type": "stdio",
  "allowed_origins": ["chrome-extension://<EXTENSION_ID>/"]
}
```

Firefox uses the same file, but replaces `allowed_origins` with `"allowed_extensions": ["pass@sensyble.com"]`.

Where the manifest goes:

| browser | Windows (registry value = path to the JSON) | Linux | macOS |
|---|---|---|---|
| Chrome | `HKCU\Software\Google\Chrome\NativeMessagingHosts\com.sensyble.pass` | `~/.config/google-chrome/NativeMessagingHosts/` | `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/` |
| Edge | `HKCU\Software\Microsoft\Edge\NativeMessagingHosts\com.sensyble.pass` | `~/.config/microsoft-edge/NativeMessagingHosts/` | `~/Library/Application Support/Microsoft Edge/NativeMessagingHosts/` |
| Brave | `HKCU\Software\BraveSoftware\Brave-Browser\NativeMessagingHosts\com.sensyble.pass` | `~/.config/BraveSoftware/Brave-Browser/NativeMessagingHosts/` | `~/Library/Application Support/BraveSoftware/Brave-Browser/NativeMessagingHosts/` |
| Chromium | `HKCU\Software\Chromium\NativeMessagingHosts\com.sensyble.pass` | `~/.config/chromium/NativeMessagingHosts/` | `~/Library/Application Support/Chromium/NativeMessagingHosts/` |
| Opera | Chrome's key (Chromium reads `Software\Google\Chrome` unless a browser changes it; Opera doesn't) | `~/.config/opera/NativeMessagingHosts/` and Chrome's folder | `~/Library/Application Support/com.operasoftware.Opera/NativeMessagingHosts/` and Chrome's folder |
| Opera GX | Chrome's key | (no Linux version) | `~/Library/Application Support/com.operasoftware.OperaGX/NativeMessagingHosts/` and Chrome's folder |
| Firefox | `HKCU\Software\Mozilla\NativeMessagingHosts\com.sensyble.pass` | `~/.mozilla/native-messaging-hosts/` | `~/Library/Application Support/Mozilla/NativeMessagingHosts/` |

Opera documents Chrome's locations as the ones it reads, while upstream Chromium on macOS and Linux reads
`NativeMessagingHosts` in the browser's own folder; the desktop app writes both when Opera's own folder
exists, so a computer with Opera and no Chrome works too. On Windows the Chrome key is always written.

`allowed_origins` must list every ID the extension can have: the pinned development ID, and the IDs
the Chrome Web Store, Edge Add-ons and Opera Add-ons assign. The desktop app takes them from
`CHROMIUM_EXTENSION_IDS`, `CHROME_STORE_EXTENSION_IDS`, `EDGE_STORE_EXTENSION_IDS` and `OPERA_STORE_EXTENSION_IDS`
in `Core/desktop-app/src-tauri/src/native_messaging.rs` (plus `extra_extension_ids` in its settings.json),
and the Firefox entry from `FIREFOX_EXTENSION_ID` there. If the host is not registered, or the ID isn't allowed,
`connectNative` fails. The extension then reports `agent_unavailable`: the popup shows "Passyble isn't
running" and a page's passkey request gets the "Passyble isn't running" card (**Use another device**: the
browser's own dialog).

### Stable extension ID

An unpacked extension's ID is derived from its path unless the manifest has a `key`. `manifest.json`
has one: the base64 DER public key (SubjectPublicKeyInfo) of an RSA-2048 key pair. The ID is the
first 128 bits of SHA-256 over that DER, hex digits mapped `0-f` → `a-p`:
**`bmjmlhhgfbpcbfbkbndlmmockkbdcaoo`**, the entry in `CHROMIUM_EXTENSION_IDS`.

```sh
node scripts/extension-id.mjs            # prints the ID for manifest.json's "key"
node scripts/extension-id.mjs --check    # fails unless native_messaging.rs allows that ID
```

`npm test` (`test/manifest.test.js`) does the same check, and also checks that `FIREFOX_EXTENSION_ID`
matches the gecko ID, so the two sides can't drift apart.

The private key is not in the repo and isn't needed: unpacked loads only use the public `key`
(`*.pem` is git-ignored). It would only matter for packing a self-hosted `.crx`.
The Chrome Web Store and Edge Add-ons assign their own IDs when the extension is first published (they
sign the package themselves; remove `key` from the uploaded manifest, since the stores reject or ignore
it). Until then the desktop app does not allow store installs from Chrome or Edge. Add the Chrome Web Store ID to
`CHROME_STORE_EXTENSION_IDS` and the Edge Add-ons ID to `EDGE_STORE_EXTENSION_IDS` (both empty for now), one line
each; the Opera Add-ons ID is already in `OPERA_STORE_EXTENSION_IDS`. `npm test` checks every entry of all
lists is a well-formed ID.

To replace the key, generate a new pair (`openssl genrsa 2048 > pass-extension-key.pem`, then
`openssl rsa -in pass-extension-key.pem -pubout -outform DER | openssl base64 -A`), put the output in
`"key"`, and put the new `node scripts/extension-id.mjs` output in `CHROMIUM_EXTENSION_IDS`.

`make-firefox-manifest.mjs` strips `key`, because Firefox uses the gecko ID instead.

## What it does

| | |
|---|---|
| **Inline menu** | A small Passyble icon in the focused username/password/OTP field. Clicking it, or pressing ↓ in an empty field (Alt+↓ anytime), opens a dropdown listing the site's logins (avatar, title, username, small **2FA** / **Passkey** tags; at most about six rows, then it scrolls), in the agent's order: logins of the page's host first, then those of other subdomains of the site; within each, favorites, then the logins used most (fills, copies of a password or 2FA code and passkey sign-ins are counted in the vault, Core/docs/ARCHITECTURE.md §13; a login saved with a sign-in page's path doesn't rank higher), then by title; the popup lists them the same way. When the vault is fully locked the dropdown shows how to unlock; when it is *soft-locked* (the desktop app kept the list after an auto-lock, `logins.find` answers `locked: true`) the logins are listed as usual under "Passyble · locked", and picking one unlocks first, without leaving the browser (desktop apps with `unlock.inline`): when Windows Hello can unlock now (turned on and set up for the vault, and its policy doesn't ask for the master password first) it starts at once, its dialog over the browser window (desktop apps with `unlock.hello_browser`; older ones show their prompt window first), while the dropdown offers **Use master password instead** (`unlock.cancel` calls Hello off); otherwise, or after **Cancel** in Windows Hello, the dropdown shows a **Master password** field with **Unlock**, plus **Unlock with Windows Hello** (when offered) and **Open Passyble** (the desktop app's own prompt). The field is an extension page in an iframe (`src/unlock/frame.html`, web-accessible), so the web page can't read or script what is typed; it sends the password to the background once (`popup.unlock-password`, as the popup's form does) and tells the dropdown only how it went (`postMessage` `{ passybleUnlock: 'ready' \| 'unlocked' \| 'escape' }`, accepted only from that frame's window and the extension's origin). Where the frame can't load within 3 s, or the page draws over the dropdown or makes it see-through (`uiIntact`), it gives way for the rest of the page to **Enter master password**, which opens the extension's popup with its unlock form (or where that isn't allowed a small window of it). On a fully locked vault, opening the dropdown yourself (a click or tap into the field, the Passyble icon, ↓, the browser's shortcut or context menu) starts Windows Hello at once in the same way; once unlocked the site's logins show with the first one preselected, so Enter or a click fills. After an unlock the picked login fills at once: the fill is the next request, with no timer or re-check before it, and the unlock itself counts as that fill's Windows Hello confirmation when **Confirm each fill** is on: the unlock names the login it is for (`for` on `unlock.options` / `unlock.hello` / `unlock` / the master password frame's `popup.unlock-password`, kept per tab for 2 minutes), and with **Every time** only that login's fill counts it (Core/docs/PROTOCOL.md "Per-fill verification"). Windows Hello never starts for a dropdown that opened on its own or for the page (a focus or click by a script, a passkey request, a lock change), and not on its own for 30 s after you said no to it (Cancel in Windows Hello, **Use master password instead**, or no dialog): then the password field comes first and **Unlock with Windows Hello** is one click away. One Windows Hello at a time: the dropdown and the popup join the one in progress. Once Passyble is unlocked, however that happened, the picked login fills. Older desktop apps: their own prompt, as before. A mouse click on an entry fills like Enter, also on pages a dark-mode extension filters. On new-password fields **Generate strong password** comes first (one row: strength on the second line, the eye swaps in the password). **Suggestions open on their own**: when you click, tap or Tab into an empty sign-in field (username / e-mail or current password) on a page with saved logins, the dropdown opens by itself, once per field and page load, with nothing preselected (Enter stays the page's until you press ↓ or click a login). The page's own first focus of a sign-in field counts as yours (`autofocus`, or the site's script focusing it after it rendered, as Google, Microsoft and most single-page apps do) when it comes within 10 s of the page loading, of a single-page app navigation, or of your Enter or **Next** on the step before (the password page of an identifier-first sign-in), before any click, tap or key in the page; never a focus the page makes after that. A tab that loaded in the background opens it when you switch to it; logins that come late (Passyble still starting, or unlocked meanwhile) open it then, while the field still has the focus and is empty and you haven't clicked or typed. On a fully locked vault nothing opens on its own (a click shows how to unlock); a soft-locked one lists the logins it kept. **A click or tap on the field opens it again**, every time (after Esc, typing or a fill), also when the field has a value: then it lists all the site's logins, unfiltered until you type (to switch accounts). A click that selects text doesn't open it, and nothing opens when there is nothing to offer (no logins for the site; a locked vault shows how to unlock). Typing into the username / e-mail field filters the logins (username and title, case-insensitive; a username or title that starts with it first, the usual order otherwise; "No logins match." when none does) and never fills; a value already in the field filters the dropdown as it opens. Typing a password or Esc closes it; a focus or click by a script never opens it; nothing opens when the site has no saved logins. Options → **Open suggestions automatically** (`autoSuggest`, on by default) turns it off. On a site with no saved logins, the menu you open yourself (the icon, ↓, the shortcut or the context menu) on a sign-in field says "No logins for <site>" and offers up to 3 **Similar sites** and **Search Passyble…** (see **Other sites' logins**). |
| **Which fields** | The menu and a username fill are only for fields that have to do with the login (`src/content/detect.js`, "the username rule"). A text field is a username with positive evidence: `autocomplete` `username`, `email` or `webauthn`, `type="email"`, or words for a username, a user, a login, an e-mail, an account, a member or customer id, or a phone or mobile number in its name, id, label, placeholder or aria-label, in the nine app languages and a few more. Standing before a password field is no evidence. Words against it always win: publish, title, slug, date, time, search, subject, comment, message, URL, website, tag, category, price, address, the name of something, a code, a quantity, and `autocomplete` tokens of something else (`one-time-code`, `cc-*`, `address-*`, `organization`...; `off` says nothing). And a field with words counts only when the box holding it and its password isn't a busy form: no text area or rich-text editor, fewer than 3 other fields (up to 6 when the form says sign in), so a settings page's e-mail field isn't a username. `autocomplete="username"` and sign-up forms are exempt. **The one exception, the classic unlabelled login form:** exactly one visible text-like input right before the password field in a form or small box, no other field, select, text area or editor there, nothing against it, and buttons or wording around it that don't say something else (Save, Publish, an editor's panel). Without sign-in wording it also takes few buttons, no bare "name", and no busy form around it. **Password only:** with no username field, only the password fills; nothing goes into another text field, and the save banner records no username (the identifier step's, if there was one, or you add it in the banner). **Passwords with no sign-in around them** (a post's or page's password in a CMS, a PDF's, or a plain password in an editor or a busy form whose buttons say Save, Publish...; `quiet` in detect.js) show the icon and open on a click, the icon or a key, never on their own; on a fully locked vault a click doesn't start an unlock. A fill from elsewhere (the shortcut, the context menu, the popup) only reaches them with the focus in them, and submitting offers no login to save. A field that says "current password", a re-authentication page or a plain password step stays a normal sign-in field. |
| **Strong passwords** | When you click, tap or Tab into a new-password field (sign-up or change-password form), the menu opens on its own with **Generate strong password** first: a masked preview (the eye button reveals it inside the closed shadow root) with its strength and length. Picking it fills the password and its confirmation. It opens on its own once per form and page load, never on sign-in (current-password) fields or on focus set by a script; Esc or typing closes it, and a click on the empty field opens it again. The password comes from the agent's `generate` (20 characters, all classes, no look-alikes). If the desktop app doesn't answer within 1.5 s, the extension generates it itself with `crypto.getRandomValues` and the same rules (`src/shared/generator.js`), so this also works with the vault locked or Passyble not running. New-password fields are found by `autocomplete="new-password"`, "new"/"confirm" wording, password + confirmation pairs, and sign-up forms (submit button text such as "Create account" first, then the form text, then the page path such as `/register`). Options → **Suggest strong passwords** turns the auto-open off; the generator stays in the menu and the context menu. **Letters and numbers only** is the second row (upper- and lowercase letters and digits, no special characters, the same length: for sites that reject symbols; `generate { symbols: false }`, and a desktop app from before `symbols` gets its answer replaced by one generated here). Picked once on a site, it becomes that site's default: the next generate there (menu, dice button) puts it first with the preview and **Generate strong password** second ("With special characters"); picking the strong one again forgets the site. The site is its registrable domain as far as it can be told without a public-suffix list (the last two labels of the host, three under a two-letter country code with a common second level such as `co.uk`), kept as a plain list in `chrome.storage.local` (`generateNoSymbols`, at most 200 sites, the oldest dropped first; no secrets), written by the background from `generate.filled { symbols }` and by the popup (`src/shared/generate-prefs.js`). The field's own rules count too (`src/content/password-rules.js`, only the clear cases): `passwordrules` whose `required` / `allowed` classes leave out `special`, `ascii-printable` and `unicode`, or a `pattern` made only of letter, digit, `_` and `-` classes (lookaheads allowed, `^[A-Za-z0-9]+$`) makes no symbols the default there; `maxlength` (8 or more), `minlength` and the lengths in `passwordrules` or a one-class `pattern` set the length. |
| **Generate and save at once** | New-password fields get a small **generate** button (dice) beside the Passyble icon (fields at least 140 px wide). One click fills the password and its confirmation **and saves it right away** as a login of the site (`draft.save` → agent `logins.create` with the frame's URL; the title is the site, the username whatever the form's username field holds), so the password is never lost when a sign-up fails half-way. A toast says **Saved to Passyble · Undo**; **Undo** deletes that login (`logins.delete`; the agent keeps a copy for 60 s). Generating again in the same tab and site updates the same login. When the form is submitted, the background updates that same login (`logins.update`: the username typed since, the final password) instead of showing **Save login?**, so there is no duplicate; if that can't happen right then (vault locked meanwhile), the banner offers it as usual. On a **password change** form (a current-password field too) the draft is a separate login named "<site> (new password)", so the old password isn't overwritten before the site accepted the change; on submit the real login is updated without asking when that is clear (see **Auto-save** below; `logins.save_check` with `ignore_id`: the draft is left out), else the banner offers **Update password**, and once that is saved (or the password turns out to be saved already), the draft is deleted. A locked vault is unlocked first (the usual unlock window), and the per-fill verification policy applies (`logins.create` asks Windows Hello when it is on). The draft's id is kept in memory only: by the content script (for Undo) and per tab by the background (for 30 minutes). With a desktop app that can't edit, the button still fills and the banner offers the password on submit. The menu's **Generate strong password** entry only fills; its submit is auto-saved (below). The button generates the site's default (letters and numbers only where that was the last choice or the field's rules say so; see **Strong passwords**). |
| **Generator options** | A small chevron touches the dice button on its right (and the menu has a **Password options** row, for the keyboard: ↓ or Alt+↓, then Enter). It fills a password at once, like the dice, and opens a small card under the field's right end: the password itself (shown like the callout under the field: CSS escapes in the closed shadow root, never text the page could search), its strength, a **Password / Passphrase** switch, for a password a **Length** slider (8 to 64) and four character type chips (**A-Z**, **a-z**, **0-9**, **!@#**; the last one on can't be turned off), for a passphrase a **Words** slider (4 to 12) and **Hyphens** (off: CamelCase, letters and one digit), then **Regenerate** and **Done**. Every change fills a new one into the password and its confirmation at once. What the field forces (`src/content/password-rules.js` `passwordLimitsOf`, planned in `src/content/generate-plan.js`) shows as a short note: no special characters (that chip off and disabled), at most or at least N characters (the slider's range follows), a passphrase that can't fit (then it is a password). The choices are the user's **generator options**, the same on every site (`chrome.storage.local` `generateOptions`: mode, length, upper case, lower case, digits, words; no secrets), sent with `generate.filled { options }` and written by the background (`src/shared/generate-prefs.js`); special characters stay the per-site choice of **Strong passwords**. The dice, the menu's rows and the shortcut then make the same: a password with every letter and digit type comes from the agent's `generate` (length, symbols), anything else (a passphrase from the EFF large wordlist, `src/shared/passphrase-words.js`, loaded only then; types left out) is made in the page with `crypto.getRandomValues`. Keyboard: the switch is one Tab stop (arrow keys), the sliders take the arrow keys, Space toggles a chip, Tab and Shift+Tab stay inside the card, Escape closes it; closing (Escape, **Done**, the close button) gives the focus back to the field and shows the password under it once (**Show the generated password**). It also closes on a click outside it, typing in the field, the field going away and leaving the page. Only trusted events on a card that is really visible count; its buttons need the click guard of every in-page button. Opened from the chevron, the last password is saved like the dice's (`draft.save`) when the card closes; from the menu only when it changed and a draft was saved before. |
| **Auto-save** | Every password Passyble generates and fills in a page (menu entry, dice button, context menu) is reported to the background (`generate.filled`), which keeps a SHA-256 of it with the frame's origin, in memory only, for 30 minutes (`src/background/generated.js`). When a submit carries exactly that password for that origin, nothing is asked: with a username, or on a site with no logins yet, `logins.save` saves a new login, or updates the login with that username (or the one a password change form changes, as `logins.save_check` decides, with the same subdomain and canonical rules as the banner); the agent keeps the old password in that login's history. The next page shows **Saved to Passyble** or **Password updated in Passyble** with **Undo** (`save.undo`, the notice's one-time token, for 60 s): a new login goes to the trash, an updated one gets its earlier version back (`logins.restore`), with the draft removed along with it. This also happens on a site in **Sites without save prompts**: the user chose Passyble's password there. The banner asks as for a typed password when it can't tell which login (no username and logins of the site already, a username change), while locked (a banner shown while locked goes away once an unlock saved it), or when the save fails. A form that is never submitted saves nothing; only the dice button's and context menu's drafts keep such a password. |
| **Fill** | `logins.fill` returns username, password and the current TOTP. If the same form has an OTP field, the code is filled too (TOTP rule 1). |
| **Show the generated password** | Right after Passyble generates a password and puts it into a field (the inline menu's **Generate strong password**, the generate button beside the Passyble icon, the context menu's **Generate password**), and the focus is in that field, the password shows once in a small callout under it (above it near the bottom of the window): "Generated password", a **Copy** button, and the password in the field's font size, digits and symbols in colour, spaces as ␣. The menu's masked preview (eye button) and the callout never show together: picking the generator closes the menu, and a menu opening on the field closes the callout. It goes as soon as you leave the field (also when the window loses focus), press a key or edit in it, click or tap anywhere else, the form is submitted, the page goes away (unload, SPA navigation, the tab hidden), the field scrolls out of view or moves behind a modal dialog, the page changes the value or shows it itself, Passyble locks, or after 30 s, and it never shows again later or on refocus. Only the field that was filled, only in its own frame. **Filled logins never show their password**, whichever way they were filled. The page's field stays `type=password`: no form history, nothing for the page to watch. The callout is Passyble's own UI (closed shadow root, top layer, `pointer-events: none` except on Copy), and the password is no text in it at all, only CSS escapes in its own style sheet, because Chromium's `window.find()` searches closed shadow roots; the content script drops it when the callout goes. **Copy** has the same guard as the menu and the banner (a real click, on UI visible for 500 ms that the page hasn't made see-through or covered), keeps the focus in the field, submits nothing and says "Copied" for a moment. The desktop app copies the password (`generate.copy`: kept out of Windows clipboard history and the cloud clipboard, cleared after 45 s). With an older desktop app, or none running, or a password generated in the extension because the app was slow, it goes through the browser's clipboard API and the background clears it after 45 s (`clipboard.copied`). **Anyone who can see or record your screen can read it while it shows.** |
| **OTP on the next step** | `logins.fill` starts the agent's 3-minute pending-OTP window for the site (registrable domain, so `login.example.com` → `auth.example.com` works). On the next page or SPA view, `logins.find` returns `pending_otp`; when the 2FA step appears in the same document without a URL change (fetch + re-render), the content script that filled the login offers the code itself. When an OTP field appears, **Auto-fill 2FA code** (on by default) fills it without a click: top frame / same-origin frames only, never another site's, and never submitted for you (sites that submit on the last digit do that themselves). With it off, the menu opens on its own with the code first (typing the code by hand closes it). Code fields are found by `autocomplete="one-time-code"`, names/labels (`otp`, `totp`, `mfaCode`, `mfacode`, `verification_code`, `app_totp`, `totpPin`, `idTxtBx_SAOTCC_OTC`, `twofactor_token`, …), split one-digit boxes (4 to 8 boxes with maxlength 1; boxes without it, which the page limits in script, when they are narrow, the same size and on one line and something says code: one-time-code, OTP wording, digits or six of them; also split 3 + 3 around a dash, wrapped deep, or each in its own shadow root; never boxes named or labelled for a phone number, a date, a card number, a captcha, or a PIN without 2FA wording; one input drawn as boxes stays one field; the code goes in box by box, else as a paste on the first box, else typed key by key, the focus moving to the next box), and, with 2FA wording next to the field (its label, its small form or box with heading and buttons: `nearText` in `detect.js`), an unnamed code-shaped field (digits, maxlength 4-10) or the only text field of a form without a password. The rest of the page never makes a field a code field: a mail inbox listing "Your verification code", a help article or a chat about 2-step verification says nothing about the Subject or search box beside it. Page text counts only for a code-shaped field on a short page (about 3,000 characters of text, at most 3 inputs) that reads like a 2FA step, and nothing next to a rich-text editor or text area (mail compose, chat, comments) is ever a code field. Coupon, promo, voucher, gift card, discount, referral, invite, postal / ZIP and captcha codes are never 2FA codes, and fields named for ordinary text (subject, title, search, message, comment, reply, note, caption, description, name, amount, address, … in the supported languages) aren't either unless they also name a code ("Code from the text message"). A field none of that catches still gets the menu with the code when you click into it while the code is pending, if it could be the 2FA step's field (`adoptableCodeField`): empty, and shaped like a code, named a code, with 2FA wording next to it, or alone on a short page that is a sign-in step (sign-in URL or 2FA wording); never a username, Subject, search, chat or other ordinary box, or a field next to a rich-text editor. Without a fill moments ago, clicking into a recognised code field opens the menu when a login of the site has 2FA. |
| **Sites that skip 2FA** | Some sites don't ask for the code every time (a remembered device, "don't ask again for 30 days", risk-based checks). Nothing is shown for a pending code unless the 2FA step is detected: a code field, or 2FA wording on a sign-in URL or next to an input or frame. On a long page (more than about 3,000 characters of text) only wording next to an input or frame counts, so an app page reached after the sign-in (an inbox that lists verification-code mails) is not taken for a 2FA step and the skip check below ends the offer there. About 10 s after the page load / navigation / form submission, a page with no code field, no 2FA wording, no password field and a URL that no longer looks like a sign-in step (`login`, `signin`, `auth`, `2fa`, `verify`, `session`, …) means the user is signed in: "skipped", nothing shows, and the code is not fetched. A code field that still shows up later gets the code (within 5 minutes of the fill; after the agent's 3-minute window only through the menu). The desktop agent remembers the last 5 outcomes per site (asked / skipped / used; `otp.outcome`, hashed domains, device-local, never in extension storage); after two skipped sign-ins in a row the site is "quiet": only a code field that actually shows up gets the code, never the toast. |
| **OTP toast** | If about 3 s after the navigation (or after the fill, once the filled sign-in form is gone) there is still no detectable OTP field, but the page reads like a 2FA step ("authenticator", "verification code", "two-factor", …) on a sign-in URL, or has that wording next to an input or frame, a toast appears top-right, where the save banner and passkey prompt show (closed Shadow DOM). Not on sites that usually skip 2FA (above). It is one line: the live code with a countdown ring, **Copy**, a **Show on desktop** icon button (`totp.popup`) and close. Alt+Shift+K shows the same toast when there is no field. **Copy** asks the desktop app to copy the code itself (`totp.copy`, the agent's `logins.copy` for the frame's own URL: kept out of Windows clipboard history and the cloud clipboard, cleared after 45 s); only an older desktop app makes the toast copy the code with the page's clipboard API (cleared after 45 s, but Windows may keep it in its clipboard history). The next code when the period rolls over (also right after a resume from sleep) is fetched with `automatic: true`, which never asks to unlock, and any lock closes the toast. |
| **Passkeys** | `navigator.credentials.create/get` are wrapped in the page's main world. Passyble handles the ceremony through `passkey.create` / `passkey.get`. A page's own request gets Passyble's card first (below), and the browser's own WebAuthn runs only after **Use another device** there, a real, visible click: never because Passyble isn't running, is locked or answered with an error. The one exception: Passyble knows it has no passkey for the request (unlocked, or soft-locked with the kept list, which is the same list). Then it steps aside at once, with no card and no wait: a page's own `get()` goes straight to the browser's own WebAuthn (security keys, phones, the browser's passkeys), a conditional request is left to the browser's own conditional request (bridge.js `endConditional`), and **No passkey for <site> in Passyble** shows at the top right for a few seconds (`showPasskeyNone`: one line, a close button, never the focus; for conditional requests once per site per page and only with **Offer passkey sign-in** on). Passyble unlocks only on a click in its card, never on its own: a page that asks as it loads (a tab restored at browser start) gets the card, not a prompt. Options → **Use Passyble for passkeys in this browser** (`passkeys`, per browser, on by default) off: Passyble leaves every passkey request to the browser. Conditional mediation (`mediation: "conditional"`, passkey autofill) is the page's own sign that it offers passkey sign-in: Passyble then asks the desktop app which passkeys it has for that request (`passkey.find`: the request's RP ID and `allowCredentials`, the origin from the browser; names only, no prompt, nothing leaves the PC) and, when it has one, shows a small card at the top right, like the save banner (`src/content/passkey-prompt.js`, closed shadow root, top layer): **Sign in with your passkey?**, the site, the account and **Sign in** (several passkeys: **Choose a passkey to sign in** and the accounts). A real, visible click (the clickjacking guard below) completes the page's pending request with that passkey through the agent (`passkey.get` with `id` and `credentialId`: no second choice in the desktop app's window; the per-fill confirmation applies as for a login fill). Soft-locked: the same accounts, **Unlock and sign in** unlocks first in the card itself (Windows Hello at once, its dialog over the browser, else the master password field, the extension's own frame, inside the card), for that login, so **Every time** asks once; never the desktop app's unlock window. **Not now**, the close button or Esc leave the page's request pending (that is how passkey autofill works; the browser's own passkey autofill keeps running alongside) and keep the card away for that site until the page is loaded again; the passkeys stay in the Passyble menu of the username or `autocomplete="… webauthn"` field, one entry per passkey (picking one may unlock the same way). The card goes when the page's request ends (signed in, aborted), on a lock, and with Options → **Offer passkey sign-in** (`autoPasskey`, on by default) off. Fully locked: the card says so with **Unlock** and no account (then the passkeys, or, with none, Passyble steps aside as above); not running: no card, the menu says how to unlock. A page that hasn't called `get()` yet gets nothing: there is no challenge to sign until it asks. **A page's own `get()` and `create()`** (the site's "Sign in with a passkey" or "Create a passkey" button) are answered in the page too, never in a Passyble desktop window (`src/content/passkey-flow.js`): bridge.js parks the request and the same card shows, for `get()` the site's passkeys (**Sign in**, several: the accounts) with **Use another device** (the browser's own passkey dialog: a phone, a security key, its own passkeys) and **Cancel** (the page gets `NotAllowedError`); for `create()` **Save a passkey for <site> in Passyble?** with the account the site names, **Save with** (the site's logins and **New login**, the one with that username preselected; `passkey.create_options`), the vault when there are several, **Save passkey**, **Use another device** and **Cancel**. The pick or Save goes to the agent only then (`passkey.get` with the passkey picked, `passkey.create` with `consent`), and the desktop app's user verification follows as for a fill: Windows Hello's own dialog over the browser under **Confirm each fill** (or for a site that requires user verification), nothing else. Locked: the card says so and **Unlock and sign in** / **Unlock and save** unlock in the card first (Windows Hello at once, else the master password frame); fully locked (right after the PC starts, also for a page asking as it loads) it is **Sign in with a Passyble passkey?** with **Unlock** and **Use another device** and shows nothing of the vault until then. After the unlock: the site's passkeys (one signs in at once), or, with none, the browser's own WebAuthn at once and **No passkey for <site> in Passyble**. Unlocked without a passkey for the site: no card, the same hand-off at once. `create()` always offers to save. Passyble not running: **Passyble isn't running** with **Open Passyble** (starts the desktop app, then asks again in the same card) and **Use another device**; not installed: the same card without **Open Passyble**. A cross-origin frame: **Passyble can't answer here** with **Use another device**. Other errors: the message with **Try again** and **Use another device**. An older desktop app: its own window asks, as before. A passkey already there (`excludeCredentials`): `InvalidStateError` at once. Passkey autofill (conditional) keeps the browser's own request running alongside: fully locked, the card offers **Unlock** (with **Offer passkey sign-in** on), then the passkeys, or Passyble steps aside with **No passkey for <site> in Passyble**. |
| **Edit logins** | In the popup, each login of the current site has **Edit** in its ⋯ menu; **Add login** adds one for the site. The editor (same popup, **Back**/Esc returns) is a plain form: labels over fields for title, username, password (show/hide and **Generate** inside the field: one click fills a strong one with the user's generator options, `generateOptions` like the in-page generator, and the site's special characters choice; the chevron touching it opens the options under the field: Password or Passphrase, length or words, the four character types, hyphens, **Regenerate**; every change fills a new one and is remembered), 2FA (current code; **Add 2FA** → **Scan QR on this page** or paste a secret / `otpauth://` link; **Remove**), websites (one row each with x and **+ Add website**; with `url_rules` in `status.features` a gear per row picks its rule: **Subdomains** (default), **Same host only** or **Never show**, a badge shows one that isn't the default, `sites.rule_hint` tags Same host only as Suggested for shared hosting, and Save sends `url_rules`) and notes. A new login with `sites.suggest`: the Name lists popular sites as it is typed (picking one fills the Name and an empty first website), website rows suggest domains, and the first website names the login (`sites.name_for`) while the Name is empty or still the given one. Esc closes the editor and Ctrl/Cmd+S saves wherever the focus is (an open menu, list or the generator options take the keys first). **Cancel** / **Save** stay at the bottom; **Open in Passyble** (shows the login in the desktop app, `app.show_item`) and **Delete login** (back to the list with **Undo** for 8 s; the agent keeps the copy 60 s) are in the ⋯ menu of the editor's header. Older desktop apps: "Update Passyble to edit logins here." Lookups of another site (search box) have no Edit. |
| **Quick actions** | A compact row of three pills under the popup's search field (icons and short labels, keyboard focusable, `role="group"`), like the Android app's notification actions: **Scan QR**, **New note**, **Generate**; on the unlock card while locked (Scan QR and New note unlock with the popup's own form first, then go on; Cancel goes back to the card). Not on pages Passyble can't fill, in the read-only vault or the small unlock / question windows. **Scan QR** (shown with `edit`): one screenshot of the visible tab (`tabs.captureVisibleTab` from the popup, allowed by the toolbar click's `activeTab`; no new permission), read in the popup itself (`src/shared/qr-decode.js`: the browser's `BarcodeDetector` where it reads QR codes, else the vendored [jsQR](THIRD_PARTY_NOTICES.md) 1.4.0, loaded on demand, the screenshot read at most 1600 px on its longer side first, then up to 3200 px). The image never leaves the popup (no network, no native messaging, nothing stored) and is dropped at once. The first `otpauth://totp/` link (`src/shared/otpauth.js`) is shown with its issuer and account and goes, after **Add 2FA**, to a login of the site the user may change (the only one is chosen; several: the one whose username is the code's account first and chosen when exactly one is, else the user picks; a login with 2FA says its code will be replaced) through `popup.totp-attach` (`totp.set` with its `id`), or to **New login for <site>** (`popup.edit-save`: the issuer as the title, the account as the username). No QR code in view: "No QR code visible on this page. Scroll it into view and try again."; a QR code that isn't for an authenticator, an `otpauth-migration://` Google Authenticator export ("Import it in the Passyble app": pass-core reads those only in the app's import), HOTP and a broken link each have their own line. **New note** (shown with `notes.create` in `status.features`): a title (a **Recovery codes** suggestion fills in "Recovery codes for <site>"; empty: "Secure note") and a monospace, unwrapped text for codes, saved as a secure note (`popup.note-create`, the agent's `notes.create`, kind `note`, bound to no site), then "Note saved". **Generate**: a 20-character password from the agent (`popup.generate`; no answer or locked: generated here, same rules), **New password**, a **Letters and numbers only** checkbox under it (a new password at once; remembered for the tab's site and preset from it, like the inline menu's choice), **Copy** (`popup.generate-copy`, the agent's `generate.copy`: cleared after 45 s, kept out of clipboard history; an older desktop app: the browser's clipboard, cleared after 45 s too). App mode goes through the desktop app, browser mode through the vault in this browser (a browser vault build without `notes.create` hides New note). What the views hold lives in the popup's memory and form only, cleared on Save, Cancel, Back, Esc, lock and `pagehide`. The keyboard shortcuts for Scan QR (Alt+Shift+Q) and New note open the popup and start the same flow (see **Context menu & keys**). |
| **Save / update** | After a sign-in, sign-up or password change (submit event, a click on a submit-like button, or Enter), the next page shows **Save login to Passyble?**, **Update password in Passyble?** or **Update username in Passyble?** (`logins.save`), as `logins.save_check` decides in the agent: nothing when the username and password are already saved (or the password is the one Passyble just filled); **Update password** for a saved username with a new password; **Update username** when another username (or e-mail) comes with the password of exactly one saved login; otherwise **Save login**. Stored passwords never come to the browser for this. **Update** is only ever offered for a login of the page's own host (`www.example.com` counts as `example.com`): when the only matching logins are saved for another subdomain of the site (`example.com` while signing in on `app.example.com`, `a.example.com` on `b.example.com`), the banner offers **Save login** and saves a new login for this subdomain (a new login on a page that looks like the site of a saved one without being it, `lookalike` from `logins.save_check`, gets a red note above **Save**: "<page> is not <site>, but its address looks like it"; an international `xn--` address a quiet "International address" line); the other subdomain's login keeps its password (it is still offered for filling here). This holds for a password change form and the login just filled too. One exception: when the page names another host of its own site as canonical (`<link rel="canonical" href>` in the head of the top frame, read by the content script as the form is submitted, https pages only), the two hosts share one login: **Update** is offered for a login of either host, an updated login of the canonical host gets the page's address too, and a new login is saved for both. The background passes it as `canonical_url` (only from the top frame, only an https address of another host); the agent accepts it only for another host of the same registrable domain on https (never another site, never a public-suffix sibling such as `bob.github.io` for `alice.github.io`). Links in frames, in the body, or added after the submit don't count. A password the dice button saved right away is saved for the page's host only. "Already saved" means a login of the site (the logins the in-page menu lists: any path, `www.` and other subdomains, and for a plain-`http` page also the logins saved for its `https` address) with the same username (trimmed, any case) and exactly the same password; a password page without a username (an identifier-first sign-in whose username step wasn't captured) counts as saved when any login of the site has that password. A login in the trash doesn't count (**Save login** is offered). **Locked**: the agent can't compare passwords. Soft-locked, the background looks at the kept list (usernames only): when the site has a login with the typed username (or the one just filled, or any login when there is no username) the banner waits, and once the vault is unlocked (within 5 minutes) it shows only if there is something to save or update. Otherwise, and when fully locked, the banner shows at once; **Save** unlocks, checks again and writes nothing for a login saved as is ("Already saved in Passyble"); a banner shown while locked goes away on its own when an unlock shows the login is saved already. A password change form is never held back. The agent's `logins.save` also writes nothing for a login saved as is, so a late Save never adds a duplicate. Forms sent with fetch/XHR that stay on the same page get the banner once the form is removed or hidden (checked for about 12 s), or on the next SPA navigation. After a sign-up or password change the banner also shows when the site redirects to its sign-in page; after a plain sign-in, a sign-in form still on screen means the password was wrong, so nothing is offered. A generated password is offered even if the page clears the field before submitting. While the vault is locked the banner still appears and **Save** asks to unlock. The banner is a compact card (logo, the question, "<username> on <site>" on one line, **Not now** and **Save**, and a small **Never for this site** link under them); **Edit** unfolds **Username** and **Password** fields to correct them before saving (show/hide button; Enter saves; unfolding it stops the auto-dismiss). The fields are an extension page in a frame inside the banner (`src/save/edit.html`, web-accessible like the master password frame), never inputs in the page: a closed shadow root hides an input's value but not the keys typed into it (a page's capture listeners on `window` hear `keydown`, `input` and composition events first). The content script never has the captured password. The frame gets the banner's one-time token (postMessage to the extension's origin only) and asks the background itself (`save.edit-get`: only that extension page, in a frame of that tab, with the token): the captured username and check, and the password only when the tab's page has the origin it was typed on; elsewhere the field starts empty ("Keep the password you typed") and a value typed there replaces it. Edits go to the background from the frame (`save.edit-set`, which answers their check); the frame tells the banner only "ready", "loaded", its height, "flushed", "submit" (Enter) and "escape". **Save**/**Update** first asks the frame to store what it shows (`flush`), then sends the token only (`save.commit`); when the stored edits differ from what was captured, the background asks `logins.save_check` again and saves accordingly. A page that keeps the frame from loading (3 s) or covers the banner while it is open loses the **Edit** section, never gets in-page fields instead; **Save** still saves the captured values (for example an edited username that matches a saved login updates that login). Only real, visible clicks count (see Inline UI below). **Never for this site** (`save.never` with the banner's one-time token) stores the site the form was submitted on in `chrome.storage.local` (`neverSave`, a sorted list of origins such as `https://accounts.example.com`) and drops the captured password; from then on `save.query` shows no save / update banner for that site (the agent isn't even asked), and the options page lists the exceptions with a **Remove** button (**Sites without save prompts**). The site is the origin of the frame that submitted the form as the browser reported it when the form was captured (`sender.url`), never anything the page or the banner sends; the message carries only the token. An origin, not a registrable domain, because the extension has no public-suffix list (`co.uk`, `github.io`): `www.example.com` and `accounts.example.com` are separate entries, and so are `http` and `https`. The auto-saved generated-password draft and the auto-save of a generated password (above) are not a banner and keep working on such a site. |
| **Other sites' logins** | Sometimes the password you need is saved for another address of the same site. On a sign-in field of a site with **no** saved login, the in-page menu (opened with the icon, ↓, the shortcut or the context menu; never on its own) says "No logins for <site>" and lists up to 3 **Similar sites**, nothing preselected (Enter never picks one), then **Search Passyble…**. The popup's empty state shows the same similar sites above its focused search field. Which logins are similar is decided in the agent (`logins.similar`, pass-core `similar_logins`, Core/docs/ARCHITECTURE.md §7): a website with the site's name (its registrable domain without the public suffix) under another suffix or subdomain (`amazon.de` or `amazon.com` on `amazon.co.uk`, `login.acme.io` on `acme.com`), else a title that holds the name ("Amazon Prime" on `amazon.fr`). Never a login of the page's own site, never another site under a shared suffix (`bob.github.io` on `alice.github.io`), and never a login whose site the page looks like (pass-core's lookalike check): a lookalike only shows in an explicit vault search, with its red **Lookalike site** tag and the stronger warning. The content script gets titles, usernames and sites only, as for the menu's own rows; the background keeps which ids it offered to which frame (5 minutes). Every suggestion is another site's login, so a pick is a cross fill like **Fill on this page** in the popup: the frame sends `logins.fill_cross { id, similar: true }`, the background forwards it only for an id offered to that frame, and the desktop app asks "Fill your <site> login into <page>?" (Cancel by default). **Search Passyble…** opens the popup (`action.openPopup`) with its vault search started for the site's name; where the browser doesn't allow that, a notice says to click Passyble in the toolbar, and the popup opened within a minute starts with that search. **After a cross fill** (from the popup or the menu) the top frame shows a small card in the save banner's place: "Use this login on <site> too?", "<username> · saved for <other site>", with **Add <site> to this login**, **Save a copy for <site>** and **Not now**. Unanswered, it goes after 45 s and comes back on the next page when the form was submitted with the filled username and password, instead of "Save login?" (whose Save is the copy). **Add** (`cross.add`) has the agent add the page's origin to that login's websites (`logins.add_url`), only with the one-time token the agent returned with that very fill (`add_url_token`, 10 minutes, one use, for that login and that origin), which the background keeps and never hands to the page or the content script; the card itself only has a token of its own. **Save a copy** (`cross.copy`) saves a new login for the page with the same username and password through the usual save (`logins.save`). The trade-off: **adding the site** makes the next fills there automatic, as for the site's own logins (no prompt; one login to update when the password changes, the same password on both addresses); **a copy** keeps them separate (each its own password history, and changing one leaves the other as it was), at the price of updating both when you change the password. On a page that looks like the login's site (the fill went ahead despite the warning), the card repeats the warning in red and neither action is the highlighted one. Desktop apps without `logins.add_url` offer the copy only. The browser vault (Passyble in the browser only) needs the rebuilt pass-wasm for both features; with an older build the menu shows only **Search Passyble…** and the card only the copy. |
| **Changed before signing in** | Passyble filled a login, then the user typed another username or password over it before signing in: instead of the save banner, the next page shows **Update this login?** with the login ("<title> · <username as filled>") and what changed ("Username changed to bob@example.com", "Password changed"; never the password), then **Update login**, **Save as new login** and **Not now**. The background keeps what the tab's last fill put in (memory only, 10 minutes, dropped on lock, taken by the next submit): the login's id, the frame's origin, the username, and the password only as a SHA-256 with that origin; the content script never sees any of it, and the card holds a one-time token of its own. On the submit (a plain sign-in of the same origin; sign-up and password change forms keep their own banner) the agent still decides first (`logins.save_check` with `filled_id`): values that are another saved login of the site get that login's usual banner or nothing, values changed back to what was filled get nothing, and a password-only change of the filled login is the usual **Update password?** banner. **Update login** (`changed.update`) has the agent change only what changed (`logins.update { url, id, fields }`: the agent checks the login belongs to the page and may be changed, and verifies like any edit; the old password goes to its history). **Save as new login** (`changed.save`) saves the submitted values as a new login of the page (`logins.save` without an id). A login in a shared vault the user may not change gets no **Update** (the card says so); a fill-only member's login with only the username changed gets no card at all (its password never goes into a new login). After a **cross fill** (another site's login, above) the card stands in for "Use this login on <site> too?" (one card, not two): it says where the login came from, and **Update** first adds the page to that login with the agent's token of that fill (`logins.add_url`), then updates it; on a lookalike the warning again and no main action. Once that card was answered, a changed submit gets the usual banner. Browser mode uses the browser vault's same `logins.update`. |
| **Password check** | The **Save login?** / **Update password?** banner and the popup's login editor say, short and plain, what is wrong with the password about to be saved: "This is one of the most common passwords.", "This password is easy to guess.", "You use this password for 3 other logins.", "You used this password before for this login.", "This password contains your username.", "This password contains the site's name.", "Short: use at least 12 characters." Nothing when there is nothing to say; the editor shows a subtle **Strong** instead. It is a warning only: saving is never blocked. The desktop app checks the password on this PC (`password.check`, pass-core's local checks with its bundled common-passwords list; the reuse check compares with the unlocked vault, and the login being updated or edited is left out); the password, its hash or any part of it is never sent anywhere else, and only issue codes come back. The banner gets the check of the captured password with `save.query`; while **Edit** is open its frame shows the check of what is typed there (`save.edit-set`, 350 ms after typing pauses), and folded away again the banner asks `save.check` (the banner's one-time token only, top frame) for the check of what **Save** would save; the issues stay in its closed shadow root. There is no "use a strong password" there: the site already has that password. The editor checks as the user types (`popup.password-check`, debounced 350 ms, a late answer for an older value is dropped, never asks to unlock: locked, only the password itself is checked) and offers **Use a strong password instead** (fills a generated one, as **Generate**) when an issue is more than a hint. Content scripts can't ask for a check of their own: there is no content-script handler for it besides the banner's token-bound `save.check`. Desktop apps without `password.check` answer "unknown method": nothing is shown. |
| **Popup** | A compact header (logo, **Passyble**, a lock-state chip, a **Lock** button and a ⋯ menu with **Add Passyble Server**, **Settings**, **Setup guide** and **Help and support** (`support.open`: the desktop app's support screen at `passyble://support` when its status lists `support.link`, else the web form `account.passyble.com/support`; the Options page About card has the same row)), a search field, and the current tab's logins as rows (avatar or website icon, title, username). A click on a row fills it; hovering or focusing a row shows **Copy username**, **Copy password** and ⋯ (**Edit**, **Copy 2FA code**, **Open in Passyble**). A login with 2FA has its live code as a small pill under the username (countdown ring; a click copies it). A login with a passkey (`has_passkey`) has a small **Passkey** tag after its title, the same as in the inline menu; vault search results show **2FA** and **Passkey** tags. **Lock** (the closed-lock icon, shown while the vault is unlocked) sends `popup.lock`; the background asks the agent to `lock`, tells the active tabs (`lock_state`) and refreshes the badge, and the popup loads its state again, which is the unlock form. When Passyble isn't running the popup shows **Passyble isn't running** as usual; a desktop app without `lock` gets "Update the Passyble desktop app". **Add login** is a quiet line under the list, or the one action of the empty state, which reads "No logins for <site>" and lists up to 3 **Similar sites** under it (`popup.state` `similar`, from `logins.similar`; the same rows and actions as vault search results, never a lookalike) above the focused search field (see **Other sites' logins**). Fully locked: the unlock form (desktop apps with `unlock.inline`): **Master password** (`autocomplete="off"`), **Unlock**, **Use Windows Hello** when the desktop app allows it now, and **Open Passyble** (the desktop app's own prompt) as the fallback. Windows Hello starts by itself as the popup opens on a locked vault (the toolbar popup only: not popup.html loaded in a tab, e.g. restored at browser start) and when a fill, copy or edit needs the vault, with the field ready beside it: typing into it calls Hello off (`popup.unlock-cancel`). Not after you said no to it moments ago, and not when the in-page menu opened the popup for the master password or in the unlock-only window; a wrong password shows the desktop app's "Wrong master password. Check Caps Lock and try again.", more than 5 attempts a minute "Too many attempts…"; older desktop apps: **Unlock** (their prompt). Soft-locked (`status.soft_locked`, desktop apps with `locked.list`): the site's logins stay listed with the lock in the header (click: the unlock form); **Copy username** copies from the list without unlocking, the 2FA pill shows dots and no code (a click unlocks, then copies), and fill, **Copy password**, **Edit** and **Copy 2FA code** show the unlock form (Cancel goes back) and then finish; nothing asks to unlock on its own. Windows Hello takes the focus, which closes the popup: a fill the popup was unlocking for is finished by the background. On a page Passyble can't fill (browser pages, the new-tab page) there is just "Nothing to fill here" and **Search vault**. Passwords are generated in the page (inline menu, generate button, context menu), in the editor, and with the **Generate** quick action (below). The search box filters the site's logins and, with a desktop app that has `logins.search` (`status.features`), also searches the **whole vault** once 2 or more characters are typed (debounced 250 ms; results show title, username, site and a 2FA badge, website icons through `icons.get` with the results' ids; ↑/↓ move over the site's logins and the results, Enter runs the selected row's action). Secrets are copied by the desktop app, never by the browser's clipboard API (which can't keep them out of Windows clipboard history and the cloud clipboard): **Copy password** and the 2FA pill of the site's own logins send `popup.copy-item` with `own: true` (`logins.copy` with the tab's URL, verified like a fill; a copied password counts as a fill for the save banner), and only an older desktop app (`not_supported`) makes the popup copy the value from `popup.copy` / `popup.totp` itself. For a login of **another site**, secrets never come to the browser: **Copy username / password / 2FA code** are done by the desktop app (`logins.copy`: its clipboard, cleared after 45 s and kept out of Windows clipboard history and the cloud clipboard; Windows Hello first for password and code when the per-fill policy is on); **Fill on this page** asks in the desktop app ("Fill your <site> login into <page>?", Cancel by default, with a phishing warning) before the agent releases it for that one fill (`logins.fill_cross`, sent by the frame that holds the form, with its own URL); **Open in Passyble** shows it in the desktop app. When the tab's page looks like such a login's site without being it (pass-core's lookalike check, `logins.search` with the tab's URL marks the result `lookalike`: `paypa1.com` or `paypal.com.example.net` for a `paypal.com` login), the result has a red **Lookalike site** tag, and **Fill on this page** (the desktop app's prompt in red, or the browser vault's own question) and **Copy password** / **Copy 2FA code** ask first ("This may be a fake site", Cancel focused); a page of any site in the vault is never called a lookalike. An international (`xn--`) address gets a quiet "International address: <host>" line over the list, no warning. Older desktop apps: the filter plus "Look up logins for <site>" (`logins.find`) as before. |
| **Browser's own autofill** | The first time the popup opens, the browser's own password saving and autofill are turned off (`privacy.services`: `passwordSavingEnabled`, plus `autofillAddressEnabled` and `autofillCreditCardEnabled` where the browser has them; permission `privacy`), so the browser and Passyble don't both offer to save or fill. Only settings that are on and that the extension may control (not set by policy or another extension) change. A notice says so with **Undo**, which hands them back to the browser (`clear`). It happens once (`browserAutofillOff` in `storage.local` records when and what changed); turning a setting back on in the browser later sticks. `src/shared/browser-autofill.js`. |
| **Start Passyble** | When the desktop app can't be reached, the popup shows **Passyble isn't running** with a **Start Passyble** button (the options page's **Check connection** offers the same). The native host starts `Pass --background` only when it is spawned and finds no agent; afterwards it answers `agent_unavailable` but connects by itself as soon as Passyble is up (it tries again on each request). So **Start Passyble** sends `popup.start`, and the background closes the native port (`NativeClient.disconnect()`) and asks `status` again. The fresh host starts Passyble and waits up to ~15 s. While waiting the button reads "Starting Passyble…"; on success the popup reloads, otherwise it says "Open Passyble from the Start menu. If it's already open, open Passyble → Settings → Autofill to check the connection." If the browser reports that the host isn't registered ("Specified native messaging host not found.", Firefox "No such native application"), the popup says so instead ("The browser can't reach Passyble. Open Passyble → Settings → Autofill to register it…") and offers **Try again**, since starting Passyble can't help. Both views also have **Setup guide** (the welcome page, see **Setup guide**). |
| **Errors** | Every page uses one helper, `friendlyError()` in `src/shared/messages.js`: a translated text with the next step, chosen by the error's `reason` (set by the background for its own checks: cards and addresses turned off, not a normal web page, pick the login in the popup first, the save prompt expired, nothing to undo, the desktop app too old) and then by its protocol code ("Passyble is locked. Unlock it to continue.", "That login is saved for a different site.", "This needs a newer Passyble app. Update Passyble, then try again.", …). The `code` stays what it was, so older callers keep working. Anything else (`internal`, `invalid_params`, unknown codes, exceptions) is "Something went wrong. Try again."; the raw message only goes to the console (`console.debug`), never into the UI. The desktop app's own text is shown only where it is translated for that purpose: its reason for `not_supported` in the unlock form (Windows Hello off, the master password due, …) and the core's sentence for `invalid_params` / `internal` from `generate.*` (an alias service's answer, a bad setting). |
| **Context menu & keys** | "Fill with Passyble" on editable fields, and **Generate password** and **Generate password (letters and numbers only)** on **password fields only** (`cmd.generate-here { symbols }`; the second also becomes the site's default, see **Strong passwords**). Context menus can't be limited to one kind of input, so the items are created hidden and the content script reports each right-click (the right button's `mousedown`, then `contextmenu`; trusted events only) with `menu.target { password }`; the background shows or hides the item with `contextMenus.update` before the menu opens. A password field is `type=password` or a field detection calls a (new or current) password. Small race: a menu the browser builds before that update lands shows the previous state once; the click is checked again by the frame, and `cmd.generate-here` ignores anything that isn't a password field. **Generate password** goes to the frame that was right-clicked (the content script remembers the trusted `contextmenu` target): a new-password field gets the password with its confirmation, another password field just that field. It is then saved at once like the generate button (above). Firefox uses the same `contextMenus` API (`menus` alias); the permission is in both manifests. **Ctrl+Shift+L** fills the login (or opens the menu if there are several). **Alt+Shift+P** (**Control+Shift+P** on macOS; the `_execute_action` command, which Firefox MV3 names the same way) opens the popup (the browser does that itself; the background doesn't see it; Ctrl+Shift+P is the private window in Firefox and Chrome, so it isn't used). **Alt+Shift+K** (⌘⇧K on macOS) fills the OTP into the focused field (any text-like field that isn't e-mail, password, phone or an ordinary box named subject, search, message and the like), else into the page's code field, or shows the toast. It used to be Ctrl+Shift+K, which is Firefox's web console and Edge's "Duplicate tab". **Alt+Shift+Q** (Control+Shift+Q on macOS; `scan-qr`) opens the popup and starts its **Scan QR** quick action, and the `new-note` command its **New note** (below): the background calls `action.openPopup` before it awaits anything (Firefox allows it only while the key press is handled) and hands the action to that popup once through `popup.state` `quick_action` (one minute, that tab); the popup reads the visible tab itself as its button does, so the screenshot never reaches the background. Where the browser doesn't let the extension open its popup, the page says "To go on, click Passyble in the browser toolbar." and the popup opened within a minute starts the action. `generate-password` puts a strong password into the focused password field (`cmd.generate-here { focused: true }`: the site's **Letters and numbers only** choice decides, and it is saved at once like the generate button); with no password field in focus it copies one instead (the desktop app copies a password it generated: kept out of clipboard history; else the browser's clipboard; both cleared after 45 s) and the page says so. `lock` locks the vault (the desktop app's `lock`, or the vault in this browser) and the page says "Passyble is locked.". `sync-now` runs **Sync now** (below) and the page says how it went. The popup's own shortcut lands in its search field (on a page Passyble can't fill, on **Search vault**), so there is no separate search command. Chrome takes at most four suggested keys, so `generate-password`, `new-note`, `lock` and `sync-now` come without one; the user binds them. Browsers apply a suggested key only on first install, so an existing install keeps its old binding until it is changed (or the extension is reinstalled). Options → **Keyboard shortcuts** lists every command with its key ("Not set" when it has none, `src/shared/shortcuts.js`); **Change shortcuts** opens `chrome://extensions/shortcuts` (`edge://extensions/shortcuts` in Edge). Firefox has no such page for an extension to open, so there each row has **Change** (press the new keys: `commands.update`; Esc cancels, Backspace removes the shortcut; a combination Firefox can't use says why) and **Reset** (`commands.reset`); `about:addons` → gear menu → Manage Extension Shortcuts works too. |
| **Save selection to a note** | Right-click selected text on a web page: **Save selection to Passyble note** opens a small window (`src/clip/clip.html`, a `windows.create` popup, so Chrome, Edge, Firefox and Opera behave alike). It shows the selection, a search over the secure notes (titles only, most recently changed first) and **New note** (title prefilled with the page title, else the site). Choosing a note adds the selection as a new line at its end (`notes.append`, never replacing what is there); **New note** saves it as a new note (`notes.create`). **Add the page address** (off by default) puts the address on the next line. A locked vault is unlocked first. An older desktop app without `notes.append` gets "Update the Passyble app to add to a note" and still offers New note. Only the text the browser passes the menu (`info.selectionText`) is used; it goes to the agent only (never stored, logged or put in a URL). Core/docs/PROTOCOL.md has the details. |
| **Sync now** | A sync icon button in the popup's header (before Lock; tooltip and accessible name "Sync now"), on the list and the unlock card, and the `sync-now` keyboard shortcut (no suggested key). The icon turns while the background syncs (`popup.sync`, `src/background/sync.js`): one sync at a time for the button and the shortcut together (a second request joins the one under way; right after a sync, its answer comes again for 3 s), errors are never kept. App mode asks the desktop app's `sync.now` (params `{}`, answer `{ synced, failed: [{ vault, reason }], no_sync }`) for every vault it syncs (Google Drive, Passyble Server, WebDAV, a folder). Afterwards the popup (or the page, for the shortcut) says "Synced just now", "<vault> didn't sync: <reason>" (with several: how many and the first), "No vault syncs yet. Set it up in the Passyble app.", or, for an older desktop app ("unknown method": `not_supported` / `needs_update`), "Update the Passyble app to sync from here"; the button stays usable. A locked vault: the popup's unlock form first, then the sync runs again. Browser mode: the vault in this browser has no sync of its own (Google Drive sync needs the Passyble app), so the button shows only in a build with the shared vaults of Teams and Family (`can_sync`); it downloads them from Passyble Cloud now (`browser-teams.js` `syncNow`, signed-in account only) and, when nothing syncs, says so with **Set up sync** (the setup guide). There is no context menu entry: the context menu only has actions for the right-clicked field. Texts: `src/shared/sync.js`. |
| **Add Passyble Server** | **Add Passyble Server** in the popup header's ⋯ menu (options page: the **Passyble Server** card; `options.html#server` focuses it). One field takes either the **address of the server** or a **`pass+server://` link**, pasted, typed or dropped (Enter submits; the hint under the field says what to paste). *Address*: `pass.example.com`, `pass.example.com:8443`, `https://host[:port][/prefix]` or `http://…`, with or without a trailing `/`, and "Server: https://…" pasted from a mail. It is normalized like pass-core's `normalize_server_url` (no scheme → `https://`; http/https and a path prefix kept; trailing `/` dropped; credentials (`user@`, with a hint when it looks like an e-mail address), a query, a fragment, other schemes, an empty host or spaces are refused) and sent to `server.pair` as `{ link: "https://pass.example.com:8443" }`; the desktop app opens "Sign in to Passyble Server" with the address filled in and handles the certificate (trust on first use). *Link*: the first `pass+server://` token in the text is taken (spaces, newlines, "Link:", quotes, `<…>` and trailing punctuation are dropped) and checked locally against [`PROTOCOL.md` §3.1](https://github.com/sensyble/PassybleCore/blob/main/docs/PROTOCOL.md) (`src/shared/pairing.js`, the same rules and test vectors as pass-core's `parse_pairing_link`, except that `token` may be left out); the link type, server, account, device name and certificate (pinned self-signed / publicly trusted; a warning for `scheme=http`) are shown, the token never is. A link with a token is a **Device pairing link** ("Connect to Passyble Server" dialog); one without (`v=1`, `user`, host still required) is a **Sign-in link**, and the desktop app asks for the account's password. Links go to `server.pair` unchanged. **Continue** calls `server.pair`; nothing is saved until the user finishes in the desktop app ("Passyble opened the sign-in window. Finish there.", or "Check the Passyble window to finish connecting…" for a pairing link). The field is cleared once it was sent, whatever the answer. Typing shows problems only after a short pause; pasting shows them at once. Errors use the shared friendly messages; a desktop app from before addresses were accepted answers `invalid_params` to an address, shown as "Update the Passyble desktop app to add a server by address, or paste the pass+server:// link instead."; invalid input is refused before anything is sent. The extension never talks to Passyble Server itself: it fills from the desktop app, which syncs with the server. There is no QR scanning in the extension (no decoder small enough to vendor: jsQR, Apache-2.0, adds well over 100 KB of code). Use **Read QR from picture…** or **Read QR from clipboard** in the desktop app instead. |
| **Cards and addresses** | Payment-card fields and name / contact / address fields get the Passyble icon and menu too, on secure pages (`https://`, or `http://localhost`), with desktop apps that have the `cards` / `identities` features (`status.features`; older apps: no icon, no menu). **Cards**: the menu lists the vault's cards as brand badge, title and "Visa •••• 4242 · 04/29" (never the number); picking one asks the desktop app ("Fill <card> on <site>?", every time; then Windows Hello when the per-fill policy is on) and fills the whole card form of that frame: number (also split over 3-5 boxes, by their sizes), holder name (or given / family name), expiry in the field's own format (`MM`, `M`, `YY`, `YYYY`, `MM/YY`, `MM / YYYY`, `MMYY`, `MM-JJ`, `type=month`, read from the placeholder, a label or the size; month / year `<select>`s by value, leading number or month name), security code and card type. **Addresses**: the menu lists identities (title, "name · e-mail"); picking one fills given / middle / family or full name, company, e-mail, phone, address lines (or a street-address textarea), city, region and postal code (`<select>`s by value or text) and the country (a `<select>` by ISO code or name, either way via `Intl.DisplayNames`). Only visible fields are filled (checked again at fill time), only in the frame that asked, and only after a real click on an entry that was on screen for 500 ms (Enter on a card or address entry needs the same, not just a key press); nothing ever fills on its own. Detection (`detectFormFields` in `detect.js`): autocomplete tokens (`cc-name`, `cc-number`, `cc-exp`, `cc-exp-month`, `cc-exp-year`, `cc-csc`, `cc-type`; `name`, `given-name`, `additional-name`, `family-name`, `honorific-prefix`, `organization`, `email`, `tel`, `street-address`, `address-line1..3`, `address-level1..2`, `postal-code`, `country`, `country-name`, with `shipping` / `billing` / `section-*` prefixes), then name / id / placeholder / label wording in several languages ("Name on card", "Kartennummer", "CVV", "Expiry", "MM/YY", "First name", "ZIP", "Ville", …), then layout (split number boxes; month / year selects beside a card number). A card form needs a card number or an explicit `cc-*` token (a lone "security code" on a 2FA page stays a code field); an address form needs two address fields or three different identity fields (a sign-in e-mail, a newsletter box or a search field never qualifies). Gift / loyalty / coupon numbers are not cards. Login fields win: a field the login detection claims (username, password, 2FA code) keeps the login menu, and in a card form the security code and expiry are never taken for a password or a 2FA code. Payment forms in a **cross-origin iframe** (Stripe, Adyen, Braintree…) work in that frame: its own content script shows the menu and asks with the frame's own URL (the desktop app's prompt also names the tab's site); one frame never fills another, so a provider that puts each field in its own iframe needs a pick per field. Opening the menu on its own follows **Open suggestions automatically** (a click into an empty card or address field opens it when there is something to list; a click on a field with the icon opens it again). Options → **Offer cards and addresses** (`formFill`, on by default) turns it all off. The popup's search lists matching cards and identities under the logins ("card" lists every card); a card row copies its number (row click, or the card icon), expiry (from the list, no prompt) or security code; an identity row copies its e-mail (row click), name, phone or address. Card number and security code are copied through `cards.fill` with `purpose: "copy"` and the `field` when clicked: the desktop app asks first, then copies the value itself (out of clipboard history and the cloud clipboard, cleared after 45 s), and no card value comes to the browser; an older desktop app sends the value and the popup copies it (cleared after 45 s). |
| **Save cards** | After a checkout form with a card is submitted (submit event, Enter in a card field, or a click on a submit or pay-like button: "Pay", "Place order", "Buy now", "Bezahlen", "Payer", "Pagar", ...), the top frame shows **Save card to Passyble?** with "Visa •••• 4242 · 04/29" (never the full number) and the site, or **Update card expiry?** when that card is saved with another expiry; nothing when it is saved as is. Desktop apps (and the vault in this browser) with the `cards.save` feature only (`status.features`; older ones: no banner; the browser vault needs a build of its WebAssembly agent from a Core with `cards.save_check`, otherwise there is no banner in browser mode). **What is read** (`src/content/card-capture.js`, the card form `detectFormFields` finds for filling): the number (12 to 19 digits that pass the Luhn check, also split over several boxes), the expiry (one field in any of the formats fill writes, `type=month`, or month / year boxes and selects) and the cardholder name (or given and family name). **The security code is never read, kept or saved**: the banner says so and that it can be typed into the card in Passyble. Only on secure pages (`https://`, or `http://localhost`), only in the top frame or a frame of the top page's own origin: card fields in a payment provider's cross-origin iframe (Stripe, Adyen, Braintree...) are skipped, so on checkouts built that way nothing is offered (the background checks the frame's origin against the tab's again, from the browser's sender info). The background keeps the card in memory only (`src/background/card-save.js`): shown within 3 minutes of the submit (3-D Secure steps in between), dropped after 5 minutes, and at once once saved, dismissed, refused or replaced. **The agent decides** (`cards.save_check`, Core/docs/PROTOCOL.md "Saving cards"): it compares the number with the saved cards itself, so a saved number never comes to the browser; the banner gets the brand, last four digits and expiry only. A card saved only in a vault the user can view but not edit is not offered again; **Update** changes only a card in a vault the user can edit. A new card goes into the home vault by default, with a vault picker when the user can save into several (Teams, Family). **Edit** unfolds **Name on card**, **Expiry** (MM/YY) and **Title** (empty: "Visa •••• 4242"; not shown for an update) in an extension page in a frame inside the banner (`src/save/card-edit.html`, like the login banner's `edit.html`): the page never sees them, and the frame gets the values from the background itself with the banner's one-time token (`card.edit-get`, `card.edit-set`); the number never goes there. **Save** sends the token only (`card.commit`); the agent checks again before it writes (`cards.save`), so a second Save never adds a duplicate. **Locked**: as for logins. Soft-locked, a card with these last four digits and expiry in the kept list (`cards.list`) makes the banner wait for the unlock; otherwise it shows, and **Save** unlocks, checks again and writes nothing for a card saved as is ("Already saved"); a banner shown while locked goes away when an unlock shows the card is saved. **Never for this site** stores the origin of the frame the card was typed in, from the browser's sender info, in its own list (`neverSaveCards` in `chrome.storage.local`); the options page lists those sites under **Never save**, marked "Cards", with **Remove**. Options → **Offer to save cards** (`cardSave`, on by default, next to **Offer cards and addresses**) turns it off; it works with the card menu turned off too. |
| **Passyble account** | Optional, for people who already have one; nothing uses it yet (Passyble Cloud sync comes later) and the extension says nothing about plans or prices. **Sign in** is the first choice of the setup page's vault part and a row in Options → About (then "Signed in as <e-mail>" and **Sign out**). Browser mode: the background opens the website's own sign-in in `chrome.identity.launchWebAuthFlow` (permission `identity`; client `passyble-extension`, authorization code with PKCE S256, `state` and `iss` checked, redirect `https://<extension id>.chromiumapp.org/`) and exchanges the code at `https://account.passyble.com/oauth/token`. The refresh token and the account's e-mail are kept in the extension's IndexedDB `passyble-account` (content scripts run in the page's origin and can't reach it; `chrome.storage.local` is readable by content scripts, which need it for settings, so it isn't used for this), the 15-minute access token in `storage.session` (trusted contexts only); pages only ever get `{ signed_in, email }`. **Sign out** revokes the session (`/oauth/revoke`) and forgets both. App mode (the Passyble app owns the account): the row shows the app's account (`account.status` through the native host: signed in or not and the e-mail, never a token), **Sign in** asks the app to start its own sign-in (`app.show_account` with `sign_in`: its window comes up and the browser opens the account page) and **Sign out** opens the app's account screen, where the user signs out after its confirmation; the page then asks the status again every few seconds for a while and whenever it gets the focus back. When the app can't be reached (no native host, not running, an error, an app too old for these methods) the extension's own account stands in: **Sign in** is the browser mode one above (Firefox's data consent asked on the click). A browser without a redirect it knows (Safari): **Sign in** just opens `https://account.passyble.com/sign-in` in a tab and the extension keeps no token. An unpacked extension may point at a test service with `devAccountBase` in `storage.local` (e.g. `http://127.0.0.1:8787`); a store install always uses account.passyble.com. `src/background/account.js`. |
| **Setup guide** | `src/welcome/welcome.html`, opened in a new tab once, on a new install (`runtime.onInstalled` with `reason: "install"`; not on an update, a reload or a browser update; Firefox too, where a temporary add-on reports `install` on every load, so a `welcomeShown` flag in `storage.local` keeps it to one tab), from the popup (**Setup guide** in the "Passyble isn't running / can't be reached" view and in the ⋯ menu) and from the options page (About → **Setup guide**). A checklist that ticks itself off: it asks the background `popup.status` every 3 s (15 s once everything is done; paused while the tab is hidden), and `src/welcome/setup-steps.js` turns the answer into steps. **1 Install Passyble for Windows**: done unless the browser reports the native host as missing (`isHostMissing`); then a **Download** link (https://github.com/sensyble/pass-releases/releases/latest), "Installed? Open it, then try again." and **Try again**. **2 Start Passyble**: done once `status` answers (or any error other than `agent_unavailable`, such as a `timeout` while a Passyble dialog is open); otherwise **Start Passyble** (`popup.start`, as in the popup). The desktop app starts its agent and pipe only once it has a vault, so a Passyble on its first-run screen also answers `agent_unavailable`: the step says "It connects once it has a vault." **3 Create or unlock your vault**: done with a `status` answer, which says locked or unlocked; one line under it says where Google Drive sync is turned on ("For Google Drive sync, open Settings in the Passyble app, then Sync, and turn on Google Drive."), since the app syncs and the extension fills from the app's vaults. **Allow Passyble on websites** (only where `permissions.contains` says the manifest's `host_permissions` aren't granted, i.e. Firefox's MV3 opt-in): **Allow access** calls `permissions.request` in the click (it needs the user gesture); declined, it says where else to turn it on in this browser's words: Firefox `about:addons` → Passyble → Permissions; Chrome, Edge and Brave their extensions page (`chrome://extensions`, `edge://extensions`, `brave://extensions`) → Passyble → Details → Site access. **Pin Passyble to the toolbar**: instructions for Chrome / Brave, Edge or Firefox; done when `action.getUserSettings().isOnToolbar` (Chrome 91+, where available) says so, or when the user ticks **I pinned it** (`guidePinned: true` in `storage.local`, the page's only stored value). Pinning and website access can be done in any order; the app steps show their buttons only when they are the current one. Opening the guide, like opening the popup, spawns the native host, which starts `Pass --background` when Passyble is installed but not running. The page keeps to titles and buttons: no explanations under the steps, and no how-to cards. Below the checklist, one link **How to use Passyble** (the help center, https://help.passyble.com/) and **Open settings**; the keyboard shortcuts are in Options only. |
| **Badge** | The number of logins matching the active tab (amber background while a 2FA step is pending). |

The options page is grouped like the desktop and Android settings: **Autofill** (inline menu, open suggestions
automatically, suggest strong passwords, auto-fill 2FA codes, offer to save logins, offer cards and addresses), **Sites without save prompts**
(the "Never for this site" exceptions, each with **Remove**), **Passkeys** (use Passyble for passkeys, offer passkey sign-in), **Keyboard shortcuts**,
**Passyble Server** (the same "Add Passyble Server" form as in the popup) and **About** (desktop app connection,
**Setup guide**, extension version). A small muted footer, "Made by Sensyble", links to https://www.sensyble.net
(the options page, and the last item of the popup's ⋯ menu, which opens it in a new tab; never in the popup body). Each setting is one row: a leading icon, a one-line title, an (i) button for the details and a
switch. Settings are stored in `chrome.storage.local`, except **Open suggestions automatically** (`autoSuggest`) and
**Suggest strong passwords** (`autoGenerate`), which are in `chrome.storage.sync` so they follow the browser profile
(falling back to `local` where sync is unavailable). None of them is a secret.

The popup and the options page share one set of design tokens (`src/ui/tokens.css`, following
[`Core/docs/BRAND.md`](https://github.com/sensyble/PassybleCore/blob/main/docs/BRAND.md): neutral greys with a hint of lime, spacing on a 4 px scale, three radii, five font
sizes, colours that meet WCAG AA in light and dark); the in-page UI carries the same values in its shadow CSS
(`BASE_CSS` in `src/content/ui-common.js`). The UI font is Space Grotesk, bundled in `src/ui/fonts/` (SIL OFL 1.1, see
`THIRD_PARTY_NOTICES.md`) for the extension's own pages; the in-page UI injects no font files and uses it only where it
is installed. Icons are the Lucide set shared with the desktop app, the logo comes from `src/shared/logo.js`.

## Code layout

```
manifest.json                 Chrome/Edge manifest (Firefox variant generated by scripts/)
_locales/<locale>/messages.json  translations (English in _locales/en is the source; see Translations)
src/background.js             service worker: message router, context menu, commands, badge, passkey params
src/background/native-client.js  NativeClient: connectNative port, request ids, timeouts, events, reconnect
src/background/clipboard-clear.js  45 s clipboard clearing: alarm, an HMAC of the copy in storage.session (key in memory only), offscreen (Chrome/Edge);
                              writeClipboard: the browser vault's copies, written by the offscreen document
src/background/app-versions.js  the version check (forced and recommended updates): fetch, cache, the 6-hour alarm, Update
src/background/account-keys.js  the Passyble password in browser mode: a linked vault opened with the account key (the
                              password and Secret Key, or another device's approval), the kept wrap and Secret Key
src/account/                  the Passyble password page: sign in with it, approve this browser from another device
src/background/account.js     the optional Passyble account: Sign in on the website (launchWebAuthFlow, PKCE), the
                              tokens (refresh: IndexedDB passyble-account; access: storage.session), refresh, Sign out
src/background/local/         the vault in this browser: local-backend.js (LocalBackend: BrowserAgent with NativeClient's
                              surface; saves, effects, timers, restart), vault-store.js (IndexedDB), session.js (the key in
                              storage.session), wasm.js (loads src/wasm/: the glue, pass_wasm_bg.wasm, kdf.js); no sync
                              (Google Drive sync is the Passyble app's)
src/wasm/                     built by `npm run wasm` (git-ignored): pass_wasm.js + pass_wasm_bg.wasm (pass-agent and pass-core),
                              pass_wasm_kdf.wasm + kdf.js (Argon2id in a fresh instance per run)
wasm/                         the Rust crates of src/wasm/ (pass-wasm, pass-wasm-kdf); never packaged
src/offscreen/                offscreen document (reason CLIPBOARD) that clears the clipboard for the service worker
                              (and writes the browser vault's copies)
src/shared/                   api.js (chrome/browser), b64url.js, errors.js, settings.js, never-save.js ("Never for this site" list), otp-logic.js (pure TOTP decisions),
                              cards.js (card / identity labels: brand names, "•••• 4242 · 04/29"; secure-page check),
                              clipboard.js (read / compare / clear in a document), generator.js (local password and
                              passphrase generator), passphrase-words.js (the EFF large wordlist, generated by
                              scripts/make-passphrase-words.mjs from the core's copy),
                              generate-prefs.js ("Letters and numbers only" remembered per site, the generator options),
                              pairing.js (pass+server:// link check and server address normalization, mirror pass-core),
                              messages.js (friendlyError: plain-language errors), i18n.js (t / tp / applyI18n over
                              chrome.i18n: messages, plurals, data-i18n, Intl formatting), icons.js (Lucide icon data, same as
                              Core/desktop-app/ui/icons.js) + svg-icon.js (builds them and the logo with createElementNS)
src/shared/app-versions.js    platform keys (extension-chrome / -edge / -firefox / -opera), version rules, the stored
                              decision (appVersionGate) the content scripts and the popup read, the read-only methods
src/ui/server-pair.js         "Add Passyble Server" form used by the popup and the options page
src/ui/popover.js             ⋯ overflow menus of the popup (header, rows, editor)
src/ui/tokens.css             design tokens + buttons, inputs, chips, menus shared by the popup and the options page
src/content/bridge.js         isolated world, document_start: WebAuthn channel end, relays to background
src/content/webauthn-main.js  MAIN world, document_start: navigator.credentials wrapper (classic script)
src/content/loader.js         isolated world, document_idle: import()s main.js (content scripts can't be modules)
src/content/main.js           orchestrator: scanning, focus, menu, OTP chain, commands, save capture
src/content/passkey-prompt.js the passkey cards: "Sign in with your passkey?" and "Save a passkey for <site>?"
src/content/passkey-flow.js   a page's own get() / create() answered in its card; unlocking inside a card
src/content/detect.js         pure form detection (username / current / new password / OTP / split OTP; cards and
                              identities: detectFormFields)
src/content/fill.js           framework-friendly filling (native setter + events), split OTP
src/content/autofill.js       cards and identities into a form: expiry formats, selects, split card numbers
src/content/menu.js           inline icon + dropdown (closed shadow DOM)
src/content/password-rules.js what a new-password field allows (passwordrules, pattern, length) for the generator
src/content/generate-plan.js  the generator's options made to fit a field (with notes), and what is made in the page
src/content/gen-options.js    the generator's options card under a new-password field (closed shadow DOM)
src/content/reveal.js         a generated password, once, under its field while it has the focus, with Copy
src/content/toast.js          OTP toast, notices, "Saved · Undo" toast; banner.js save/update banner (editable;
                              the password check, shared/password-issues.js: its texts);
                              ui-common.js shadow host helpers
src/popup/, src/options/      popup (editor.js: the login editor) and options pages
src/welcome/                  setup guide opened on install: welcome.html/js/css, setup-steps.js (checklist from popup.status,
                              vaultStart: where the guide starts; the app installed skips the choice);
                              src/shared/welcome-link.js opens it; the vault part (Google Drive sync: needs the Passyble
                              app, Download, Check again; create a vault in this browser; open a vault file; Sign in)
src/shared/vault-mode.js      the mode setting and "the Passyble app always wins" (hostOutcome, nextHostState,
                              browserModeOffered, the move record), the browser vault's settings, the delete guard,
                              "only on this device" (isLocalOnlyVault, the 30-day "Not now"),
                              the master password tip's per-vault flags (masterTipAfterUnlock, ...);
                              vault-file.js: "Download a copy" and reading a picked vault file;
                              master-password.js: the texts of the master password rule (the core's check)
scripts/                      make-firefox-manifest.mjs, package.mjs, make-icons.mjs, extension-id.mjs, check-syntax.mjs
test/                         node --test suites (jsdom); local-reminder.test.js: the "only on this device" decision;
                              master-tip.test.js: the master password tip's flags
test/e2e/                     opt-in browser tests (Playwright Chromium, local pages, mock agent): the 2FA chain
                              (otp.e2e.mjs), generate + save at once (draft.e2e.mjs) and passkeys against the
                              real agent (passkey.e2e.mjs, webauthn-rp.mjs, passkey-client.js)

```

Message flow (fill):

```
content (frame F) --{type:'logins.fill', id}--> background --{id, method:'logins.fill', params:{url: <F's URL from sender.url>, id}}--> native host --> agent
content <--------------- {ok, result:{username,password,totp}} ----------------------------------------------- (agent rechecks URL ↔ item)
```

Passkeys:

```
page --navigator.credentials.get--> webauthn-main.js (MAIN) --CustomEvent on secret channel--> bridge.js (isolated)
     (a page's own get()/create(): parked; passkey-flow.js shows the card: passkey.find / passkey.create_options,
      then the user's click: the passkey picked / the login chosen; "Use another device": the browser's own)
     --runtime.sendMessage--> background: origin = sender.origin, crossOrigin from frameId/tab URL --> passkey.get
     <-- PublicKeyCredential-like object, or the browser's own navigator.credentials.get: from the card only
         after "Use another device" (answers the card decided carry final: true and never fall back)
```

## The vault in this browser (browser mode)

Design: [`Core/docs/EXTENSION_VAULT.md`](https://github.com/sensyble/PassybleCore/blob/main/docs/EXTENSION_VAULT.md) (Phase A). This build: a
vault kept in this browser, master password, autofill, save prompts, generators, 2FA codes, cards and identities,
the popup's list, search and editor, lock and auto-lock, "Confirm each fill", a downloaded copy of the vault file.
**Google Drive sync needs the Passyble app on this computer**: the app signs in to Google and syncs, and the
extension then fills from the app's vaults (app mode). The extension has no Google sign-in, no Google OAuth client
and no Drive code of its own ([`Core/docs/GOOGLE_DRIVE.md`](https://github.com/sensyble/PassybleCore/blob/main/docs/GOOGLE_DRIVE.md), "Browser extension").

* **Which mode.** `vaultMode` in `storage.local` (`app` / `browser`; unset behaves as `app`, so updating users
  see no change). On a new install the background sends one `status` to the native host: the app answers, or the
  host is there but the app isn't running: app mode, silently. No host ("host not found"): nothing is set, and the
  setup guide (`src/welcome/`) asks, with titles and one-line captions: **Google Drive sync** (needs the Passyble
  app), **Create a vault in this browser**, **Open a vault file**, **Sign in**, or (Windows) **Get the Windows app**.
  Options → **Vault** switches to the app on the user's click (the browser vault is kept, locked). While the app is
  installed, the setup guide skips the choice and Options shows one line instead of the switch; see
  [The Passyble app always wins](#the-passyble-app-always-wins).
* **Google Drive sync** (the setup guide's choice, `#sync`; also the popup's ⋯ menu, **Set up sync** and the
  Sign in part's **Use Google Drive**): "Google Drive sync needs the Passyble app on this computer", with
  **Download the Passyble app** (https://www.passyble.com/download) and **Check again** (`vault.check-app`: one
  `status` on a fresh native port; `connected`, `installed` or `missing`). Found: the extension uses the app from
  then on, and the app checklist says where Drive is turned on in the app. With a vault in this
  browser, one line says how to take it along (Download a copy in Settings, open it in the Passyble app). Older
  links (`#drive`, `#move`) open the same part.
* **Create a vault in this browser** (`#create`; also the popup's **Create a vault**): the master password twice,
  "Passyble can't reset your master password", and the "only on this device" notice below. The new master
  password must pass the **master password rule**, defined once in pass-core (`check_master_password`,
  Core/core/src/password_check.rs, "The master password rule"): at least 12 characters, not a common password
  (also with capitals, leetspeak, digits or symbols added), and strong enough (score 3, "good", about 60 bits)
  with the signed-in Passyble account's e-mail address and the word "passyble" counted as known to an attacker.
  As it is typed (150 ms after typing pauses) the page asks the background (`vault.check-master`, extension
  pages only), which runs pass-wasm's free function `checkMasterPassword(password, namesJson)` in the browser
  vault's WebAssembly; no vault is needed and the answer has no secrets (`ok`, `score`, `label`, `entropy_bits`,
  `issues`, `name`, `min_length`). Under the field: every reason as a red list in the core's order while it is
  refused (Use at least 12 characters; one of the most common passwords; leave out your e-mail address,
  username and the vault's name; avoid repeated characters, sequences and keyboard rows; too easy to guess,
  with how to fix it), "Good: strong enough for a master password." (or "Strong: ...") once it passes, nothing while
  the field is empty. **Create vault** checks again and, while it is refused, shows the first reason and goes
  no further; the background's `vault.create` checks the length, then the rule with the account's e-mail, and
  pass-wasm's `create` refuses a password the rule refuses (`invalid_params`) as well. The texts are in
  `src/shared/master-password.js`; the rule is never re-implemented here. Existing master passwords are never
  judged by it: unlocking, **Open a vault file**, "Confirm each fill" and **Delete vault** take the password as
  it is, so a vault whose master password predates the rule opens as before. The typed password goes only to
  the background (for the check while it is typed, never logged or kept, and once to create the vault) and the
  fields are cleared once the vault is created.
* **The master password tip.** Such an older master password gets a calm, dismissible note instead. Right after
  a successful unlock with the master password typed by the user (the popup's unlock form, the in-page menu's
  master password frame, **Open a vault file**), the background runs the same `checkMasterPassword` on it (the
  vault's name and the signed-in account's e-mail address as known names) and keeps one per-vault flag in
  `storage.local` `vaultMasterPasswordTip` (`{ [vaultId]: { weak?: true, dismissed?: true } }`, device-local,
  never synced, never the password, score or reasons; `src/shared/vault-mode.js`). Each such unlock sets
  `weak` to the answer, so a master password changed on another device clears it at the next unlock here. A
  restarted service worker's session restore (the key from storage.session), a failed unlock and "Confirm each
  fill" leave the flags alone, as does a check WebAssembly from an older core can't run. Options' Vault card
  shows "Master password: Your master password is easier to guess than Passyble now allows for new ones. It
  still works." as a plain row at the top of Locking while `weak` and not `dismissed` (`master_tip` in
  `vault.state`, browser mode only; with the Passyble app the app shows its own), with **Dismiss**
  (`vault.master-tip-dismiss`, extension pages only). Nothing in the popup, nothing on the unlock screen.
  **Delete vault** removes the vault's flags and **Create a vault** starts with none. The browser vault has no
  Change master password screen yet (EXTENSION_VAULT.md Phase B), so the row has no button for it; once there
  is one, it clears both flags (`clearMasterTip`).
* **This vault is only on this device.** A vault kept in this browser has no sync (`local_only` in `vault.state`
  and `popup.state`; `isLocalOnlyVault` in `src/shared/vault-mode.js`). The create step shows "Only on this
  device. If you lose this device you lose these passwords. You can turn on Google Drive backup now or later."
  with **Set up sync**; **Your vault is ready** repeats it. The popup shows the reminder card above the list
  while it is due (`reminder`): **Set up sync** (the setup guide at `#sync`) and **Not now**, which hides it for
  30 days for that vault (`vault.reminder-dismiss`: the time per vault id in `storage.local`
  `vaultLocalReminderDismissed`), after which it shows again while the vault is still only here. It never blocks
  anything and is never on the unlock screen. Options' Vault card has the same notice as a row with **Set up
  sync**. Export stays the manual way (**Download a copy**), named in the text.
* **Sign in** (a Passyble account) signs in, then says "Passyble Cloud storage is coming soon." with **Use Google
  Drive** (which needs the Passyble app, above).
* **One protocol, two backends.** `agent()` in background.js goes to `NativeClient` (app mode) or `LocalBackend`
  (browser mode, `src/background/local/`), which wraps pass-wasm's `BrowserAgent` with the same requests and
  answers. After every call it writes `takeSave()` to IndexedDB, carries out `takeEffects()` (`lock_state`:
  the key to or from storage.session and the usual `lock_state` event; `copy`; `vault_changed`) and schedules
  `tick()` at `nextTickMs()` (a timer, and the `pass-vault-tick` alarm that wakes a stopped worker). Calls run
  one at a time. Without a vault every method but `status` answers `no_vault` (the popup shows **Create a
  vault**, **Google Drive sync** and **Sign in**). Methods that need the desktop app: `unlock` is the popup's
  unlock form here, `totp.popup`, `app.show_item` and `server.pair` are `not_supported` and hidden in the UI.
* **Asking the user.** What the desktop app asks in its windows, the extension asks in its popup (or a small window
  of it, `popup.html?ask=1`, where the popup can't be opened): `unlock`, the master password again with
  **Confirm each fill** (`verification_required`: the background gets a grant with `verifyPassword` and sends the
  request once more with it), a card fill from the page, and a login of another site ("Fill on this page"). Only
  the background attaches `grant` and `confirmed`, never from a content script's message; a popup that closes
  without answering cancels the request. An unlock the user started for one login counts as that fill's check.
* **Storage.** The encrypted vault file (the apps' format) in IndexedDB `passyble` (`unlimitedStorage`), with the
  previous copy; nothing vault-related in `storage.sync`. The unlocked vault's 32-byte key in `storage.session`
  (`setAccessLevel` TRUSTED_CONTEXTS: memory only, not for content scripts) with the time of the last use, so a
  stopped service worker opens the vault again without Argon2id and the auto-lock carries on. Removed on every lock.
* **Locking.** After 15 minutes without use by default (Options → Vault: 1, 5, 15, 30, 60 minutes, or only when
  the browser closes), when the computer locks (`idle`, a soft lock; can be turned off), when the last browser
  window closes, and always on browser exit and extension update or reload (the browser clears storage.session).
* **Argon2id** runs in the service worker through `src/wasm/kdf.js`: a fresh 38 KiB instance per run (about 0.13 s
  with the vault defaults), zeroed and dropped, so its 64 MiB never stays. The popup and pages stay responsive (other
  processes).
* **Copies.** The vault releases a copied secret to the background only (`copy` effect); the offscreen document writes
  it and the usual alarm clears it after 45 s. A browser can't keep it out of Windows clipboard history or the cloud
  clipboard (the UI doesn't repeat this).
* **A copy.** **Download a copy** (Options) saves the encrypted vault file (`Passyble-<date>.vault`, opens in the
  Windows app, imports on Android). **Delete vault** needs a copy downloaded in the last hour and the master password.
* **Vaults the extension synced with Google Drive itself** (older builds; rare, as no released build had Drive set up): the record still
  says `source: 'drive'`. It opens and saves here as before (nothing syncs any more, `drive_needs_app` in
  `vault.state` and `popup.state`); the popup shows **Google Drive sync needs the app** under the list and the
  setup guide and Options say "This vault synced with Google Drive in an older version. It keeps working in this
  browser; to sync it again, use the Passyble app." Its Drive copy stays where it was, for the Passyble app.
* **Not yet.** Sync of the vault in this browser (Google Drive goes through the Passyble app), Passyble Cloud,
  passkeys (the passkey card says they need the app for now and offers **Use another device**), quick unlock with a
  passkey (PRF), recovery code and changing the master password, the full vault page (creating cards and
  identities), import and export, website icons. After a service worker restart a pending 2FA code, grants, the
  vault search's grants, rate limits and the 2FA outcome history start empty, and a soft lock becomes a full lock.

To try browser mode, use a computer (or a virtual machine) without the Passyble app: its native host is registered for
the whole computer, so every browser profile there finds it and uses the app.

## The Passyble app always wins

Design: [`Core/docs/EXTENSION_VAULT.md` §2.6](https://github.com/sensyble/PassybleCore/blob/main/docs/EXTENSION_VAULT.md).
The rule: whenever the Passyble app is installed on this computer (its native host is registered: `connectNative`
answers, or answers "the app isn't running", rather than "host not found"), the extension uses it.

* **What counts.** One look for the host has three outcomes (`hostOutcome` in `src/shared/vault-mode.js`):
  `connected` (the app answered), `installed` (the host is there; the app is closed, has no vault yet, or is slow) and
  `missing` ("host not found"). Installed is as good as connected: the extension never leaves app mode because the
  app is closed or slow; the popup shows the usual "Passyble isn't running" with **Start Passyble**.
  `nextHostState` turns the outcome into the stored state (`vaultHostSeen`, `vaultHostMisses`, `vaultAppGone`).
* **A fresh install with the app** (running or not): app mode at once; the setup guide skips the mode choice and shows
  the app checklist with one line, "The Passyble app is installed on this computer, so Passyble uses it here." The
  same line replaces the switch in Options → **Vault**, and `vault.create`, `vault.open-file` and
  `vault.set-mode` to `browser` answer `invalid_state` while the app is installed.
* **Browser mode, then the app appears.** The background looks for the app in browser mode when the popup opens (at
  most every 10 minutes), when the browser starts, after an update, and when Options or the setup guide opens (at
  most every 15 s). Found: app mode on its own. If the vault in this browser has items, the popup offers once
  "Move 23 items from this browser into the Passyble app" (the count comes from the vault's `vaults.list` while it is
  open; locked: "Move the items in this browser into the Passyble app"). **Move** opens Options' move card, the
  close button is **Not now**; either way Options keeps the card (`vaultMove` in `storage.local`).
* **The move** (Options → **Vault**, `vault.move`): the master password of the vault in this browser (the Passyble
  password for a vault linked to an account), and the app's vault when it has more than one it can edit
  (`vault.move-targets`). The background opens a second instance of the browser vault's file with that password
  (`LocalBackend.exportForMove`: the vault in this browser stays locked, no event fires), then sends its encrypted file
  and vault key once to the app (`items.import`, PROTOCOL.md §2; the app's own unlock prompt first when it is locked).
  The app adds every item (logins with 2FA and passkeys, notes, cards, identities) and skips exact duplicates (same
  site, username and password, and nothing the app's copy lacks). Answer: added, skipped, and the titles of the items
  that could not move: "Some items could not move" lists them, they stay in this browser, and **Try again** moves
  only what is still missing. An app without `items.import`: "This needs a newer Passyble app".
* **After a move** the vault in this browser is untouched, marked as moved, with **Delete the vault in this browser**
  (a confirmation first: "Deletes the vault in this browser for good. Its items are in the Passyble app now.";
  `vault.delete-moved`, only after a complete move). Declining keeps everything as it was.
* **The app uninstalled.** In app mode every request tells: a "host not found" counts at most once a minute, and only
  three of them in a row mark the app as gone (one failure, a burst of failures, or a failure followed by an answer
  never do). Options then says "The Passyble app is no longer on this computer. You can keep a vault in this browser
  instead." with the switch back; nothing is created on its own.
* **Firefox and Safari.** The same where native messaging exists; the browser vault's own feature detection
  (`browserVaultSupported`) is unchanged. Safari's extension ships inside the Passyble app, so it is always app mode.

## One Passyble password (browser mode)

Design: `Passyble-Password-Design.md` section 8 "Extension" (next to the repositories), Core
[`docs/ARCHITECTURE.md`](https://github.com/sensyble/PassybleCore/blob/main/docs/ARCHITECTURE.md) §18 and
[`docs/EXTENSION_VAULT.md`](https://github.com/sensyble/PassybleCore/blob/main/docs/EXTENSION_VAULT.md) §17.
**App mode is unchanged**: the Passyble app on this computer holds the account key, and the extension offers none
of this (the account page says to sign in in the app).

* **A vault linked to a Passyble account** (file version 3, `vaultFileAccess`) opens with the Passyble password,
  never with a master password. The popup's unlock form (`state.account_unlock`) says "Passyble password", shows the
  **Secret Key** field with "This is a shared computer: don't keep my Secret Key" while this browser keeps none,
  the e-mail field while the address isn't known here, and **Approve from another device**. The in-page menu's
  frame says "Passyble password" too (`account.keys-state`); the first Secret Key goes through the popup. **Open a
  vault file** reads the picked file first (`vault.file-access`) and asks the same way. "Confirm each fill" and
  **Delete vault** take the Passyble password for a linked vault. No master password tip for it.
* **Where the account key comes from** (`src/background/account-keys.js`, every cryptographic step pass-core's in
  the WebAssembly, Argon2id in the short-lived kdf module): the password wrap kept in this browser (offline); else
  the account service's, after a sign-in signature: `POST /v1/account-key/unlock` (after `GET /v1/account-key` and
  an `unlock` nonce) when this browser is signed in to that account, else `POST /v1/prelogin` and `POST
  /v1/session/password` (client `passyble-extension`, platform `extension`), which also signs this browser in. The
  address and device name are signed exactly as the service normalizes them. The password and the Secret Key never
  leave the browser. A wrong password from the kept wrap asks the service once (the password may have changed on
  another device); offline it is simply wrong. The account key is used for the one unlock and freed; a restarted
  service worker opens the vault with the vault key in `storage.session`, as every browser vault.
* **Kept in this browser**: the extension's IndexedDB `passyble-account` (which content scripts can't reach),
  record `keys`: per account the address, `ak_id`, salt, Argon2id parameters, the password wrap and, unless "This
  is a shared computer" was ticked, the Secret Key. None of them helps anyone without the Secret Key and the
  password.
* **The account page** (`src/account/account.html`; Options' Account row **Passyble password**, the setup guide's
  **Use your Passyble password**, the popup's **Approve from another device**): **Sign in with your Passyble
  password** (`account.password-sign-in`: the address, the password, the Secret Key once the background asks for
  it; the linked vault here opens at once), and **Approve from another device** (`account.approval-start`: a
  request without a session, `POST /v1/account-key/requests`; the page shows its QR, then polls
  `account.approval-poll` every 2 s: `GET /v1/account-key/requests/<id>` with `X-Pass-Request-Token`; once the
  other device accepted without a camera, the background reveals `r_n` and the page shows the 6 digits to type
  on that device; when sealed, the account key and the Secret Key arrive with a new session and are held in
  memory only: the page shows the arrived account key's fingerprint to compare with the other device's Settings,
  Passyble password, and only **It matches** (`account.approval-confirm`) keeps them, signs in with that session,
  keeps the Secret Key and opens a linked vault here; **It doesn't match**, Cancel or 5 minutes drop them (the
  WebAssembly objects freed, the session revoked) with a loud warning, design D4 of the breach audit). The pending
  request's one-time key waits in `storage.session`. Firefox asks for the data consent on
  the click, as for Sign in. Changing the password, the Secret Key or the recovery code, and resets, stay in the
  apps (the page says so, with the download link).
* **Not yet**: the per-browser passkey quick unlock (PRF) wrapping the account key (the WebAssembly has `prfWrap`
  and `AccountKey.openPrf`; the quick unlock itself, EXTENSION_VAULT.md §5.2, isn't built for any vault yet), making
  a vault linked here, approving another device from this browser, reading the Emergency Kit's QR with a camera.

## Same-computer sign-in with the Passyble app

The extension and the Passyble desktop app on the same computer sign each other in, so the person signs in once
(the account service's `POST /v1/session/handoff`; the desktop's side: Core `docs/PROTOCOL.md`,
`account.peer` and `account.handoff_*`). Mostly browser mode; in app mode only a session of the extension's own
is handed over or dropped (the second-last point). `syncWithApp` in `src/background/account.js` runs on
`account.status` (Options, the setup guide; the page waits
at most 2.5 s, then asks again while `app_syncing`), when the popup opens, on the 6-hour alarm and right after a
sign-in here; at most every 30 s, one at a time with Sign in and Sign out, and never when no native host was
found for this browser (`vaultHostSeen` 'missing') or the app was just found unavailable.

* **The app signed in, the extension not:** the background makes a PKCE verifier, sends only its S256 challenge
  and the device name (`account.handoff_to_extension`), and redeems the code the app got from the service at
  `/oauth/token` (client `passyble-extension`, `redirect_uri=urn:passyble:handoff`). Options and the setup guide
  then say "Signed in through the Passyble app on this computer".
* **The extension signed in, the app not:** `account.handoff_begin` gives the app's challenge, the background asks
  `POST /v1/session/handoff` (client `passyble-windows`) with its own access token, and `account.handoff_finish`
  hands the app the code. Not while the user signed out of the app and kept the extension (`accepts_handoff`
  false).
* **The app signed out with "also sign out of the browser extension":** the extension signs out (revoke, as Sign
  out) if it is signed in to that address, then `account.peer_signed_out`. Different addresses on each side:
  nothing.
* **Sign out here** while the app is signed in to the same address: Options asks with "Also sign out of the
  Passyble app on this computer" ticked (`account.sign-out` with `also_app`: the app signs out through
  `account.sign_out`). Unticked, or the app couldn't be asked: "paused" (the `peer` record of the IndexedDB
  `passyble-account`), so the app doesn't sign the extension in again until the user signs in here or the app
  is seen signed out.
* **Safety:** each side gets its own session from the service (on the device list, revocable on its own); the app
  only ever receives a challenge or a code, never a token or a verifier; the code lives 60 s, is single use and
  works only with the verifier that never left the side signing in. A handoff gives a session, never a key: the
  Passyble password with the Secret Key (or a device approval) is still needed to open anything. An older app
  answers "unknown method" and nothing happens; a handoff that fails waits 10 minutes before the next try (the
  service allows 20 codes per account an hour). Firefox needs the account data consent first, as for Sign in.
* **App mode with a session of the extension's own** (signed in here while the app couldn't be reached: not
  running, or too old). The account shown is the app's, so that session shouldn't linger unseen. Only while it
  exists is the app asked (`account.peer`): signed out and taking a handoff, the session goes to the app as above
  (`account.handoff_begin`, `POST /v1/session/handoff`, `account.handoff_finish`), and once the app says it is
  signed in the extension's own is revoked and forgotten; a handoff that fails keeps it and waits 10 minutes. The
  app signed in to the same address: revoked and forgotten quietly, nothing else asked of the app. Signed out with
  "also sign out of the browser extension" for that address: revoked and forgotten, then `account.peer_signed_out`.
  The app signed in to another address: nothing automatic; `account.status` adds `browser_email` and Options says
  "The Passyble app is signed in as X; this browser as Y" with "Use the app's account"
  (`account.use-app-account`: this browser's own session is revoked and forgotten). App mode never signs the
  extension in from the app: it shows the app's account already.
* **Note:** browser mode is chosen only while no app was found (`vaultHostSeen` 'missing'), and the next look that
  finds the app switches to app mode ("The Passyble app always wins"), so browser mode seldom has an app to talk
  to; app mode is the usual case, through the point above.

## Teams and Family (linked vaults)

Shared vaults of a Team, a Family or a direct share, beside the user's own vault (Core/docs/PROTOCOL.md §2 "Linked
vaults", EXTENSION_VAULT.md §16). Nothing changes for a user without them.

* **App mode** (`status.features` has `vaults`): answers name each item's vault (`vault_id`, `vaults`, `vault`);
  `src/shared/vaults.js` turns them into what the UI shows. The in-page menu and the popup tag each login with its
  vault when several are open. A fill-only member's login is filled, never shown or copied: no copy button, no
  editor, `popup.copy` refuses it (`fill_only`), and the save banner's Edit never shows a password just filled from
  it. A view member's login opens without Save or Delete (the agent answers `read_only` anyway). The save banner and
  the passkey card offer **Save in** for a new login (the vaults the user may change, the agent's default first). The
  popup's "Shared with me" line (`vaults.list`) counts the open shared vaults, those that can't be opened and the
  people to confirm, with **Open in Passyble** (`app.show_teams`) for the desktop app's Teams and Family screen.
* **Browser mode** (`src/background/teams/`): with the extension's own Passyble account sign-in, the shared vaults on
  Passyble Cloud open with the keyset in this browser's vault (WebAssembly `openShared`), their encrypted files are
  kept in IndexedDB (`passyble-teams`), requests run across vaults (`multiForUrl`, `multiSearch`,
  `multiDefaultSaveVault`), roles are enforced as in app mode, and the popup offers quick key confirmations (the
  fingerprint the person reads out is typed and compared) and links to the web console for the admin screens.
  Secure send (`send-client.js`) and the organisation routes (`org-api.js`) list the routes they assume while the
  account service doesn't have them yet.
* **Quick Cloud checks** (browser mode, `cloud-cadence.js`, `browser-teams.js` `pull`): with a Passyble Cloud
  device (only a paid plan gets one) and the vault unlocked, the shared vault files are checked with conditional
  GETs (304 when nothing changed) on unlock, when the browser is back online, on popup open after 15 s, and every
  minute (alarm `pass-cloud-sync`, cleared on lock, sign out, without a device, and when every shared vault's plan
  refuses uploads). Errors back off from 30 s, doubling to 10 minutes with jitter; a 429 (`busy`) waits at least its
  `Retry-After`; offline checks are skipped. The account service is still asked every 5 minutes at most. The device
  and the check times are kept in `storage.session` for a sleeping service worker. Changes upload on each write as
  before. App mode: the desktop app syncs on its own.
* **Team password rule** (`policy.password_mode`: `choice`, the default, `second_password` or `one_password`; a
  Team admin sets it for the team or one member, a Family is always `choice`). The extension has no share action
  and no "Use a separate password" or link flow (password changes stay in the apps), so it shows the rule and keeps
  the Team's vaults unchanged where the apps would:
  * App mode: the desktop app's `vaults.list` `password_policy` (`{ mode, action_needed, title, message }`, texts
    in the app's language) shows under the "Shared with me" line while `action_needed` is true, with **Open in
    Passyble** (`app.show_teams {}`), where the fix is. While `second_password` isn't met, the agent reports the
    Team's vaults with `can_edit: false` and answers `read_only` to any change: the popup offers no Edit, the save
    banner no update and no **Save in** for them, and the editor says "You can use this vault's logins but not
    change them." An older desktop app without the field: nothing new.
  * Browser mode (`browser-teams.js`, `org-api.js` `passwordMode`): the effective mode is `second_password` if any
    Team says so, else `one_password` if any, else `choice` (a missing key, from an older service, is `choice`).
    This browser's vault is the home vault; it has a second password when it opens with its own master password
    (not linked to the Passyble account: `vaultAccess` kind `password`). `second_password` while it is linked: the
    vaults of the Teams that require it are read-only here (the view member's path: no save, update, delete,
    member change or key confirmation in them, **Save in** offers the other vaults, a copy changed before stays
    here until the rule is met), with "Your team requires a second password" and that this browser can't give
    its vault one: use the Passyble app on Windows or Android, or ask the team admin. `one_password` while it has
    its own password: "Your team uses one password", a notice only, as this browser can't switch it either. The
    rule is kept in `teamsSeen`, so a restarted service worker opens those vaults read-only before its first
    refresh. For an account in a Team, the answer (`{ "second_password": bool }`) goes to `PUT
    /v1/me/second-password` with the extension's own account session at a Teams refresh, when it differs from
    the last one sent for this sign-in. `sharing_blocked` isn't read: there is no share action here.
* **Vault filter and pinned vaults** (`src/popup/vault-filter.js`, both modes, two or more vaults open): over the
  list, a button with the vault shown ("All vaults" or a vault's name; the home vault without a name is
  "Personal") whose menu picks one, and a chip per pinned vault (a click shows only it, a second click all again).
  The site's logins, similar sites, the search's results and cards and addresses follow it. Pinned vaults come first,
  in pin order; the menu's **Pin vault** / **Unpin vault** acts on the vault shown. The pins are the agent's
  (`vaults.pins` / `vaults.set_pins`, `status.features` has `vaults.pins`; kept in the home vault's synced settings
  `ui.pinned_vaults`): the whole list goes back with the change, so pins of vaults this browser doesn't show stay;
  a sixth pin only says "You can pin up to 5 vaults. Unpin one first." Pins of vaults not in the list aren't drawn.
  Browser mode adds `known`, the shared vaults `src/background/teams/` opens beside the agent's own, so it keeps
  their pins.
  An older agent ("unknown method"): the filter without pins.
* **Vault switcher** (`src/popup/vault-switch.js`, app mode): with more than one vault of the user's own
  (`vaults.own`), the header shows the open one's name; its menu lists them, pinned first (by `vault_id`), the open
  one checked. Another one: `vaults.switch`, then the desktop app asks for that vault's unlock itself (the
  extension never takes that master password) and the popup says "Unlock Work in the Passyble app" until it sees
  that vault open (it loads again on `lock_state`). Team and Family vaults open with the personal vault: they are in
  the filter, not the switcher. `not_supported` (one vault, always so in browser mode, or an older app): none.
* **Moving a login** (popup editor, `status.features` has `items.move`): **Vault** offers the open vaults the user
  may edit (`can_edit: true`, not a view or fill member's), pinned first, for a login of one of them, when there is
  another. Save updates the fields, then `items.move` (`popup.edit-move`); the login comes back with a new id, which
  the list selects. Into a shared vault the popup asks first ("Everyone with access to <vault> will see it"); the
  desktop app may ask too (`cancelled`: the fields stay saved, the login where it was).
* Firefox and Safari repackage these files as they are (no new permission).

## Forced and recommended updates

The owner sets a **minimum** and a **recommended** version per platform in the admin console; every Passyble app
and this extension read them from one public list (Core's shared contract, "forced and recommended updates"). The
extension's platform keys are `extension-chrome`, `extension-edge`, `extension-firefox` and `extension-opera`
(`src/shared/app-versions.js` `detectPlatform`: Firefox by `runtime.getBrowserInfo`, Edge and Opera by their
`navigator.userAgentData` brands "Microsoft Edge" / "Opera", else the `Edg/` / `OPR/` user agent tokens; Brave and
every other Chromium browser are `extension-chrome`). The Safari extension ships inside the macOS and iOS apps and
follows `macos` / `ios`: it isn't checked here.

**The check** (`src/background/app-versions.js`): `GET https://account.passyble.com/api/app-versions` from the
background only, with `credentials: "omit"`, no referrer and no headers of its own; nothing about the account, the
device, the browser profile or the vault is sent (the answer is the same public list for everyone). 10 s timeout.
It runs as the background starts (at most every 6 hours then, 15 minutes after a failed try, since a Chromium service
worker starts many times a day), at browser start, on install and right after an update (so a block clears at
once), and every 6 hours by the `pass-app-versions` alarm. The last good answer is kept with its time in
`storage.local` (`appVersionsCache`); a failed fetch uses it, and with nothing ever fetched nothing is blocked. The
answer must be `v: 1`; a missing platform or field, or `null`, is no requirement; versions are `MAJOR.MINOR.PATCH`,
compared numerically, a `-prerelease` / `+build` suffix ignored on both sides; a malformed value is ignored (never a
block). The current version is `runtime.getManifest().version`. The decision (`appVersionGate` in `storage.local`,
read by the content scripts and the popup) names the version it was made for, so a decision from before an update
never blocks the new version. An unpacked extension with `devAccountBase` asks that service instead (as for Sign in).
No permission was added: `host_permissions` is `<all_urls>` and the extension pages' CSP has no `connect-src`; the
endpoint also answers with `Access-Control-Allow-Origin: *`, which covers Firefox, where host access is opt-in.

**Below the minimum** the extension is read-only until it is updated:

* Content scripts stay quiet: no icon, inline menu, autofill, 2FA code, save or update banner, generator or toast
  (`main.js` doesn't start; a block that comes while a page is open stops it at once). Passkeys go to the browser's
  own WebAuthn: `bridge.js` answers every request with `fallback` and `isConditionalMediationAvailable` is the
  browser's own, so the MAIN-world hook takes no part.
* The background enforces it whatever asks: every content-script message except the in-page UI's font and the
  right-click report answers `update_required` (reason `extension_update`); so do the popup's and the save edit
  frame's fill, edit, save and sync handlers (`popup.fill`, `popup.fill-cross`, `popup.edit-*`, `save.edit-*`,
  `vault.create`, `vault.open-file`, ...); and `agent()` refuses every agent method outside the read-only list below, in
  app mode and browser mode alike. Keyboard shortcuts and the context menu do nothing.
* The popup opens on **Update required** with the owner's message (plain text through `textContent`, at most 280
  characters). **Update** calls `runtime.requestUpdateCheck()` and, when the browser has an update, reloads the
  extension with it (`runtime.reload()` once `runtime.onUpdateAvailable` says it is downloaded); otherwise (no update
  yet, throttled, or no `requestUpdateCheck`, as in Firefox) it opens the store page: Chrome Web Store
  `https://chromewebstore.google.com/search/Passyble`, Edge Add-ons `https://microsoftedge.microsoft.com/addons/search/Passyble`,
  Firefox `https://addons.mozilla.org/firefox/addon/passyble/`, Opera `https://addons.opera.com/en/search/?query=Passyble`
  (the store searches until the listings have their own address, as in Website/README.md). **View my passwords**
  opens the read-only vault: a vault search (`logins.search`, 2 or more characters) whose rows copy the username,
  the password or the 2FA code through the desktop app (`logins.copy`), with a line that stays at the top ("The
  Passyble extension is read-only until you update it. You can still view and copy your passwords.") and Back to the
  Update screen. Nothing there fills, edits, adds, deletes, saves or uses passkeys. Unlocking works as usual.

**Below the recommended version** (and at or above the minimum) the popup shows a banner over the list, "A new
version of Passyble is available." with **Update** (as above) and a close button; closing it is remembered for that
recommended version (`appVersionDismissed`), so it shows again when the owner raises it.

**The desktop app below its own minimum** is read-only too and says so to the extension:

* `status` gains `"update_required": true` while it is read-only (absent or `false` otherwise).
* Every refused request answers the error code `update_required` (message "Update the Passyble app to keep
  filling"). Refused: `logins.find`, `logins.fill`, `logins.fill_cross`, `logins.save`, `logins.save_check`,
  `logins.get_for_edit`, `logins.update`, `logins.create`, `logins.delete`, `logins.restore`, `totp.get`,
  `totp.popup`, `totp.set`, `totp.from_image`, `otp.outcome`, `cards.fill` (unless `purpose: "copy"` with a
  `field`), `identities.fill`, `passkey.find`, `passkey.create_options`, `passkey.create`, `passkey.get`,
  `server.pair`, `generate.email` and any unknown future method. Still answered: `status`, `unlock`,
  `unlock.options`, `unlock.password`, `unlock.hello`, `unlock.cancel`, `lock`, `logins.search`, `logins.copy`,
  `cards.list`, `identities.list`, `cards.fill` with `purpose: "copy"` and a `field`, `app.show_item`, `icons.get`,
  `password.check`, `generate`, `generate.options`, `generate.username`, `generate.copy`. This extension's own
  read-only list is the same (`READ_ONLY_METHODS` in `src/shared/app-versions.js`).
* The extension words the code "Update the Passyble app to keep filling." (`friendlyError`, `errAppUpdateRequired`):
  the popup shows the same read-only vault with that line (`popup.state` answers `read_only: "app"` and skips
  `logins.find`), the in-page menu shows it where it would have listed logins (on its own once per page, and on a
  click), a fill shortcut or the context menu shows it as a notice, and a page's passkey request goes to the
  browser's own WebAuthn at once with that notice (no Passyble card; a conditional request is left to the
  browser's own).

## Security model and limitations

* **Network requests.** In every mode the version check (see [Forced and recommended updates](#forced-and-recommended-updates)):
  `GET https://account.passyble.com/api/app-versions`, a public list, with no account, device or vault data, no
  cookies and no referrer. Otherwise in app mode none (Sign in is the Passyble app's, and Google Drive sync is the desktop
  app's own), except Sign in while the app can't be reached, which is the browser mode one below. In browser mode only one service: `https://account.passyble.com` after the user clicks
  **Sign in** (the sign-in window, the token exchange, a refresh while signed in, the revoke on **Sign out**). The
  extension never talks to Google: the vault in this browser has no sync, and Google Drive sync needs the Passyble
  app on this computer. Nothing else goes out (no analytics; the vault, settings and pages never go to the account
  service). The store listings and the privacy text must say so before this ships.

* **No secrets at rest.** Nothing secret goes into `storage`. The background holds a submitted password
  in memory only until the user answers the save banner, and for at most 60 s (5 min until commit).
  The banner gets a one-time token, never the password: its edit frame (an extension page) asks for the
  password itself, and gets it only when the tab shows the origin the password was typed on. A password generated in the inline menu stays
  in the content script's memory (same frame) until the form is submitted or for at most 10 min, then
  follows the same rule.
* **Showing a generated password** (`src/content/reveal.js`) puts it on screen for anyone who can see or
  record it, once, right after it was generated, until you leave the field (at most 30 s). Filled logins
  never show. The page already has that password (it is in its field), so the callout gives it nothing
  new; still it never goes into the page's DOM, an attribute, the console or a window message, the field
  is never switched to `type=text` (which would put it into the browser's form history and change how
  keyboards and other tools treat the field), and inside the closed shadow root it is CSS escapes, no
  text, so the page can't select, copy or `window.find()` it. Copy takes only a real click on the visible
  callout (the clickjacking guard below) and sends the password to the background and the desktop app,
  never through anything the page can read. The agent's `generate.copy` copies only what its own
  `generate` returned in the last 10 minutes, so it can't be used to write anything else to the clipboard.
* **Passyble Server pairing links** carry the account's token. The extension only holds one in the input field
  and in the single `popup.server-pair` message to the background, which checks it and forwards it once to
  `server.pair`. It is never logged, written to `storage`, or shown (the preview lists everything but the
  token), and the field is cleared after sending. The field has `autocomplete="off"` and `spellcheck="false"`
  (Chrome's enhanced spell check sends text to a web service). `popup.server-pair` is a `popup.*` handler,
  so content scripts can't call it. A plain server address is only normalized and forwarded: the
  extension never contacts the server; the desktop app does, after the user confirms there.
* **Editing** (`popup.edit-*`, `popup.totp-check`, `popup.totp-scan`, `popup.show-item`) is popup-only as well.
  The page is the tab's URL from `tabs.get`, and the agent only lets a login be read or changed from a page it
  belongs to, so the editor can't reach another site's login. Opening the editor releases the password (Windows
  Hello first when the per-fill policy is on, which then covers saving that login for 10 minutes); the 2FA
  secret itself never comes to the browser (only its settings and current code). The background and the agent
  both check field types and lengths, websites must be http(s) without credentials, 2FA secrets must parse
  (base32 of 80+ bits or `otpauth://totp/`). Edits are rate-limited in the agent. What the editor holds lives
  in the popup's memory and its form only: never in `storage`, cleared on Save, Cancel, Back, lock and
  `pagehide`. **Scan QR on this page** takes one screenshot of the visible tab (`tabs.captureVisibleTab`, allowed
  by `activeTab` / `<all_urls>`, which the manifest already has: no new permission; Firefox has the same API)
  and sends it to the agent once; the QR code is decoded in the desktop app (pass-core, Rust), not in the
  extension. A scanned or pasted secret stays in the popup until Save. Clicking an `otpauth://` link on a page isn't
  the extension's: the browser hands it to the app registered for it (the desktop app, Core/desktop-app/README.md), and no
  content script reads or intercepts these links.
* **Quick actions** (`popup.totp-attach`, `popup.note-create`, `popup.generate-copy`) are popup-only like editing: the
  page is the tab's URL from `tabs.get` (the agent checks the login belongs to it), the background checks the 2FA
  value and the note's title (one line, at most 256) and text (at most 20000) before the agent checks them again.
  **Scan QR** decodes in the popup: the background never sees the screenshot, only the `otpauth://` link.
* **Vault search** (`popup.search`, `popup.copy-item`, `popup.fill-cross`) is popup-only. `logins.search` needs
  an unlocked vault, at least 2 characters, is rate-limited in the agent and returns no secrets (title,
  username, site, has-2FA). Copies of another site's login are made by the desktop app, never in the browser.
  "Fill on this page" for another site's login is the one place a login leaves the vault for a page it isn't
  saved for: the agent only does it for an item its recent searches returned, after the user confirms in the
  desktop app (default Cancel), and the background only lets the frame ask right after the popup requested
  exactly that login in that tab (one fill per request).
* **Drafts** (`draft.save` / `draft.undo`) create or change only a login of the frame's own site (the URL comes
  from `sender.url`; the agent checks it), and only after the trusted click on the generate button (shown for
  at least 500 ms, not covered) or the browser's own context menu.
* **Lock** (`popup.lock`) is popup-only too: no content script (so no web page) can lock the vault through it.
* **Cards and addresses** belong to no site, so instead of the URL binding of logins: secure pages only (`https://`,
  `http://localhost`; checked by the content script, the background and the agent), lists without secrets (brand,
  last four digits, expiry; name, e-mail, company), and the desktop app asks before **every** card fill or copy
  ("Fill <card> on <site>?", then the per-fill policy). An identity fills once the vault is unlocked. The request
  carries no URL from the page: the background takes the asking frame's own `sender.url` (a payment iframe's own
  origin, never the tab's; `top_url` from `sender.tab.url` only labels the prompt) and answers that frame alone. The
  card stays in the content script's memory only while it is written into the visible fields; the popup gets no
  number or code at all when the desktop app copies it (`field`), and from an older one only on its way to the
  clipboard (cleared after 45 s). Nothing is stored.
* **Never for this site** (`save.never`) is the one storage write a page's action leads to, and it can only add the
  origin of the frame whose submit the background captured itself, after checking the banner's token and the top
  frame; the options page can remove entries. The list holds origins only, no secrets.
* **Start Passyble** (`popup.start`) is a `popup.*` handler too: a web page can't make the browser spawn the
  native host or start the desktop app through it.
* **Setup guide** (`src/welcome/`) is an extension page: it only sends `popup.status` and `popup.start` (no vault
  data, no secrets), asks the browser about its own permissions and toolbar pin, and stores only `guidePinned`.
* **Icons** are built with `createElementNS` from static data (no `innerHTML`), so they also work in
  the closed shadow roots on pages with Trusted Types.
* **The browser names the site, not the page.** Every URL or origin sent to the agent comes from
  `sender.url` / `sender.origin` / `tabs.get`. Messages from content scripts can't choose a URL, and
  `popup.*` handlers only accept messages from extension pages. The agent rechecks URL ↔ item binding
  for `logins.fill`, `totp.get` and the passkey calls, so a compromised page can't fetch another site's
  credential by item id.
* **Opaque origins.** A sandboxed iframe or a page served with `Content-Security-Policy: sandbox`
  (often used for user-uploaded HTML) has a URL that names a site but an opaque origin: it is not that
  site. The content scripts don't start there (`document.domain === ""`), the bridge leaves its
  passkey requests to the browser, and the background refuses any content-script request whose
  `sender.origin` is `"null"` or differs from the origin of `sender.url`.
* **Frames.** Fills run in the frame that asked and use that frame's URL (never the tab's). Auto-fill of OTP codes is
  limited to the top frame and same-origin frames. Passkey requests from cross-origin iframes fall
  through to the browser, because only the browser can check the `publickey-credentials-*` permissions
  policy. That includes a frame with the top's origin under a cross-origin parent (A > B > A), which
  bridge.js reports from `location.ancestorOrigins` / the parent chain.
* **Inline UI** lives in closed shadow roots and ignores untrusted (script-generated) clicks and keys.
  The page can still see that the host elements exist and restyle or cover them, so clicks that fill,
  save or copy (menu entries, **Save**/**Update**, the toast's **Copy** / **Show on desktop**) also need
  the UI to have been on screen for 500 ms (entries swapped in under the cursor don't count), no
  opacity / filter / clipping on the host or `<html>` (dark-mode `invert`/`hue-rotate` filters are
  fine), no page popover / modal dialog / fullscreen element shown over it after it, and, where the
  browser has IntersectionObserver v2 (Chromium), not occluded. Enter in the menu needs the same, minus
  the delay. Firefox can't detect a `pointer-events: none` overlay in the normal stacking order. A menu
  that opens for something the page did (its passkey request plus a focus) preselects nothing, so the
  Enter that submits a typed username never picks a login or passkey. The passkey card follows the same
  rules (closed shadow root, trusted and visible clicks only, never takes the focus). Like the other hosts,
  the page can see that a `pass-passkey` element appeared, which tells a site that asked for passkey
  sign-in that this Passyble has a passkey for it (one of its own; never which one or for whom).
* **In-page UI on top of the page's modals** (`ui-common.js`, "layering"): every host (the field icon and
  generate button, the dropdown with its master password frame, the toasts, the save banner) is a
  `popover="manual"` element shown with `showPopover()`, so it sits in the browser's top layer above any
  z-index and above a `<dialog>` opened with `showModal()`, a page popover or the fullscreen element that
  was there first. When the page puts something into the top layer later that overlaps it (`beforetoggle`,
  a dialog's `open` attribute, `fullscreenchange`; checked on focus changes and every 500 ms), the host is
  shown again (hide, show) on top, and its clicks count only 500 ms after that. While a modal dialog is open
  everything outside it is inert, so UI for a field inside a top-layer element (modal dialog, open popover,
  fullscreen element) is mounted inside that element as its last child (still a popover, above the dialog's
  content; a click on it is no "click outside" for the page's light dismiss), toasts and the banner go into
  the topmost modal dialog or fullscreen element and back to `<html>` when it closes, and the icon and menu of
  a field behind a modal dialog it isn't in are hidden. The page's own `::backdrop` rules are kept off our
  popovers (an adopted stylesheet). Pointer, mouse, focus and key events from inside the UI stop at the
  host, so a modal library's "pointerdown / focus outside" handler doesn't close the sign-in dialog when a
  login is picked (the extension's own listeners are capture listeners and still see them). Without
  popovers (Chrome < 114, Firefox < 125) the host keeps `z-index: 2147483647` and, inside a dialog with a
  transform, the menu corrects its position by the measured offset. The page can restyle a host it holds
  in its dialog like any other, so the checks above apply there unchanged. A page focus trap that runs in
  the capture phase can still pull the focus out of the master password frame (the popup then takes it).
* **MAIN-world interception can be observed by the page.** `webauthn-main.js` runs in the page's own JS
  world, so a page can tell the functions are wrapped and can call the wrappers itself. This grants
  nothing beyond calling `navigator.credentials` directly: the origin is derived by the background,
  and the agent shows its own consent dialog. The channel name is a random secret announced at
  `document_start` with a synchronous event, before page scripts run. Private copies of
  `dispatchEvent`, `addEventListener`, `CustomEvent` and `JSON` are taken at the same moment.
  Requests carry random ids. A page that learns the channel (possible only in the rare fallback
  handshake when the bridge loads second) can at most feed fake results to its own code, or ask Passyble
  about its own origin.
* **WebAuthn fidelity.** The returned credential is a plain object whose prototype is
  `PublicKeyCredential.prototype`, and all members are own properties. `instanceof` checks pass, but a
  page that calls the native prototype getters directly will see "Illegal invocation". Passyble makes only
  ES256 platform credentials. Requests without alg -7, or with `authenticatorAttachment: "cross-platform"`,
  go to the browser, as do `get` with `mediation: "silent"` / `"immediate"` and `create` with
  `mediation: "conditional"` (they must not show Passyble's windows). `getClientExtensionResults()` reports `credProps` and the WebAuthn PRF extension (`prf`: `enabled` at creation,
  `results.first` / `second` as ArrayBuffers, base64url in `toJSON()`); the inputs go to the desktop app as
  base64url (docs: Core/docs/PROTOCOL.md, "WebAuthn PRF extension"), and an older desktop app that sends no
  answer is reported as an authenticator without PRF (`enabled: false`, `prf: {}`). Cancellation can't be
  passed to the agent (the protocol has no cancel method): an aborted request rejects with
  `AbortError` at once, and any late agent reply is dropped. Checked against what client libraries
  use (`test/e2e/passkey.e2e.mjs`): `toJSON()` and `JSON.stringify`, `PublicKeyCredential.parse*OptionsFromJSON`,
  `getTransports()`, `getPublicKey()` (SPKI DER), `getPublicKeyAlgorithm()`, `getAuthenticatorData()`,
  `authenticatorAttachment: "platform"`, `credProps.rk`, and `InvalidStateError` for an excluded credential.
* **Injected twice.** On install, update or reload the background injects the content scripts into open
  tabs again. After a reload the earlier copy's bridge is dead, so every wrapper records the browser's
  own method (`Symbol.for('passyble.webauthn.native')`) and a copy with a live bridge falls back to that,
  never through the dead copy (which would never settle: `getClientCapabilities()` used to hang there).
  The bridge confirms each request synchronously (`:got`); without that receipt the browser takes over.
  A copy injected next to a still-live one (no handshake answer) leaves the live one in charge.
* **Fingerprinting.** The content modules are `web_accessible_resources` (needed for `import()`), so a
  page can detect that Passyble is installed. The pinned ID makes that easier (store builds have fixed IDs too).
* **Browser mode** (the vault in this browser): the browser *is* the vault while it is unlocked: the decrypted vault
  is in the service worker's WebAssembly memory and its key in `storage.session` (memory only, trusted contexts; a
  copied profile has only the encrypted file, like any synced copy). Malware running as the user can read the
  browser's memory, as it can read the desktop app's; locking on the computer locking and the auto-lock bound that.
  Uninstalling the extension or removing the browser profile deletes the vault, which is only on this device (the
  pages say so and remind the user to download a copy or move it into the Passyble app). Copies can't be kept out of Windows clipboard history. A malicious extension update could read the vault
  when it is next unlocked (the trust placed in the store channel).
* **Clipboard.** Passwords, 2FA codes and generated passwords copied from the popup or the OTP toast
  are cleared after 45 s, like the desktop and Android apps (usernames aren't). The copying page tells
  the background, which keeps only a SHA-256 of the value in `storage.session` (memory only) and sets a
  `chrome.alarms` alarm, because the service worker / event page may be suspended before the timer
  would fire. When it fires, Chrome and Edge open an offscreen document (`offscreen` permission, reason
  `CLIPBOARD`) that clears the clipboard with `execCommand('copy')` and closes again. Firefox has no
  offscreen API but its event page has a DOM, so it clears there with `navigator.clipboard`
  (`clipboardWrite`). A newer Passyble copy restarts the 45 s.
  Passyble doesn't ask for `clipboardRead` (it would add a "read data you copy and paste" warning), so it
  usually can't check that the clipboard still holds its value. It then clears anyway, because its own copy
  is the last one it knows about. Something copied in another app within those 45 s is therefore
  cleared too. If the browser does allow reading, a clipboard that no longer matches is left alone.
  Chrome before 120 rounds the alarm up to 1 min for packed (store) installs.

## Protocol notes (assumptions the desktop agent should match)

* `passkey.create`: the extension maps error `invalid_state`, or `cancelled` with a message containing
  "invalid_state", to `InvalidStateError`. Plain `cancelled` maps to `NotAllowedError` and
  `url_mismatch` to `SecurityError`. `not_found`, `agent_unavailable`, `timeout`, `extension_reloaded`, `locked`
  (after one unlock attempt), `invalid_params`, `not_supported` and `internal` fall back to the browser's WebAuthn. "Use another device"
  in the desktop app's passkey window is therefore reported as `not_found`; in the extension's card it hands the
  request to the browser without asking the agent. A Save in the card sends `consent: { itemId }` (the login chosen,
  `null` for a new one): no consent window.
* `passkey.create_options` (`{ origin, rpId, userName, excludeCredentials }`, background method): what the save card
  offers (the site's logins, the suggested one, the vault, whether the passkey is there already). "Unknown method"
  (an older desktop app): the request goes as `passkey.create` without `consent`, and the desktop app asks.
* `passkey.get` may carry an extra `id` (item id) and `credentialId` when the user picked a specific passkey
  in the inline menu or the passkey card (conditional mediation). The agent limits the choice to that item's
  passkey and signs without asking which one again. Unknown extra params should be ignored, not rejected.
* `passkey.find` (`{ origin, rpId, allowCredentials }`, background method) lists the passkeys for a pending
  conditional request; an older desktop app ("unknown method", `not_supported` here) means no card, and the menu
  goes by `logins.find`'s `has_passkey` as before.
* Optional fields are left out rather than sent as `null` (`id` in `totp.get` / `logins.save`, `length` in `generate`; `symbols` in `generate` only as `false`).
* Passkey calls on a `locked` vault: the background calls `unlock` once and retries, except for a choice made in
  the page (`inPage`: the cards and the menu unlock there first), which gets `locked` back for the card to unlock
  again in the page (desktop apps with `unlock.inline`; older ones still get `unlock`). The same `withUnlock` wraps
  fills, copies, `totp.get` with an `id` (a login the user picked), `totp.popup`, the editor's calls and saves, so on
  a soft-locked vault (the list is shown, PROTOCOL.md "Soft lock") the user's one click unlocks and completes the
  action. `totp.get` without an `id` (the pending code after a fill) never pops up the unlock window.
* Soft lock: `popup.state` lists the tab's logins when `status.soft_locked` (with `locked: true, soft_locked: true`);
  without it (fully locked, or an older desktop app) it answers `{ locked: true }` as before. The badge counts the
  site's logins while soft-locked too and its tooltip says "(locked)".
* Request timeouts: 30 s; 3 min for calls that can wait for Windows Hello or a confirmation (`logins.fill`,
  `logins.fill_cross`, `logins.copy`, `totp.get`, `totp.popup`, `cards.fill` and the editor's calls); 10 min for
  `unlock` / `unlock.hello` / `unlock.password` / `passkey.*` / `server.pair`. A timeout is reported as `timeout`
  ("Passyble didn't answer in time. Check for an open Passyble window, then try again."), not as
  `agent_unavailable`: no "isn't running", no Start Passyble, no back-off for the badge.
* A request lost with its native port (the host exited: Passyble restarted, or the browser ended the host) is
  rejected as `agent_unavailable` with `portClosed`, unlike the host's own "The Passyble desktop app is not
  running" answer. The background sends such a request once more on a fresh port, but only for methods that
  only read (`status`, `logins.find`, `logins.search`, `logins.save_check`, `cards.list`, `identities.list`,
  `generate`, `unlock.options`, `icons.get`); fills, saves, edits, unlocks, passkeys and prompts are never sent
  twice, and neither is anything when the host isn't registered.
* Content scripts: when the extension was updated or reloaded under an open page, requests fail with
  `extension_reloaded` ("Passyble was updated. Reload this page to use it."). A message that didn't reach the
  background ("Receiving end does not exist", while its service worker starts) is sent again after 250, 500
  and 1000 ms (Brave's service worker can take that long to wake) before it counts as `agent_unavailable`.
* An in-page menu that says Passyble isn't running (also the generator's "Start the Passyble app before you
  submit" note) asks again every 2 s for up to 30 s with its own list call (`logins.find`, `cards.list`,
  `identities.list`), without showing "Loading" each time, and shows its content in place as soon as Passyble
  answers. It stops when the menu closes, opens on another field, or the page's scripts stop.
* `server.pair`: the extension sends the trimmed link (≤ 4096 bytes, already checked), or a normalized
  plain server address (`https://host[:port][/prefix]`, no trailing `/`) in the same `link` field, and
  shows `{ host, user, sign_in }` from the result (`popup.server-pair` answers
  `{ shown, host, user: null, sign_in: true, address: true }` for an address). `not_supported`, and
  `invalid_params` whose message contains "unknown method" (desktop builds from before `server.pair`),
  are shown as "update the Passyble desktop app"; `invalid_params` for an address (builds from before
  addresses) as "Update the Passyble desktop app to add a server by address, or paste the pass+server://
  link instead."
* `lock` (no parameters, result `{ "locked": true }`) locks the vault; it is sent only for the popup's **Lock**
  button (`popup.lock`). "unknown method" from an older desktop app is shown as "Update the Passyble desktop app";
  `agent_unavailable` shows "Passyble isn't running". The agent's `lock_state` event follows, but the background
  doesn't wait for it.
* Editing needs `status.features` to contain `edit` (`popup.state` reports `can_edit` / `can_scan`); an agent
  that answers "unknown method …" is shown as "Update Passyble to edit logins here." `rate_limited` is shown as
  "Too many changes. Wait a minute."
* Website icons: when `status.features` contains `icons.get` (`popup.state` reports `can_icons`), the popup
  sends `popup.icons` after listing the logins, and the background asks the agent `icons.get` for the tab's URL.
  The icons come from the desktop app's cache ("Show website icons" in the app), only for the logins that match
  the page, as `data:image/png` URLs; the extension never fetches an icon or loads a remote image. No icons, an
  error or an older desktop app keep the first letter. The in-page inline menu keeps letters (a `data:` image
  there would fall under the page's CSP). The setup guide says so under its checklist (`welcomeIconsNote`): the
  Passyble app downloads the icons directly from each website, and where to turn them off.
* Start Passyble relies on the native host starting the desktop app when it is spawned and finds no agent
  (Core/native-host), so `popup.start` just closes the port and sends `status` again.
* Cards and addresses: `cards.list` / `identities.list` (`{ url }`, no secrets, `locked: true` from a soft lock's list)
  when a card or address field is focused or its menu opens; `cards.fill` (`{ url, id, top_url? }`, plus
  `purpose: "copy"` and `field: "number" | "security_code"` from the popup: the desktop app copies that value itself
  and answers `{ copied: true, field }` without card values; an older one ignores `field` and answers the card) and `identities.fill` (`{ url, id }`) when one is picked, with the usual
  unlock-and-retry. "unknown method" from an older desktop app is `not_supported`, which hides the menu for the page;
  the content scripts first ask the background `forms.features` (from `status.features`, cached 30 s, and the
  `formFill` setting) and show nothing without them. `url_mismatch` (not a secure page) and `cancelled` (declined in
  the desktop app) are shown as messages (a cancel quietly in the popup).
* The only event handled is `lock_state` (refreshes the badge, open menus and the popup; its `soft` flag needs no
  handling, the popup asks `popup.state` again). Others are ignored.
* Forced updates: `status.update_required: true` while the desktop app is below its minimum version, and the error
  code `update_required` for every request a read-only app refuses (the lists are under
  [Forced and recommended updates](#forced-and-recommended-updates)). The extension falls back to the browser's
  WebAuthn on `update_required` like on `not_supported`.

## Translations

Guide for translators (all four apps: formats, glossary, `node ../Core/scripts/i18n-check.mjs`): [Core/docs/TRANSLATING.md](https://github.com/sensyble/PassybleCore/blob/main/docs/TRANSLATING.md).

Every user-visible string (popup, options page, in-page menu, toasts, save prompt, context menu, keyboard
shortcut descriptions, toolbar tooltips, error texts, tooltips, `aria-label`s and placeholders) is a
message in the standard WebExtension catalog. The browser picks the catalog for its UI language and falls
back to English message by message, so a partial translation works.

```
_locales/en/messages.json      English: the source of every text, with a description for translators
_locales/<locale>/messages.json  one per language: es, fr, de, zh_CN, ja, ko, hi, id, …
```

To add a language, copy `_locales/en/messages.json` to `_locales/<locale>/messages.json` (Chrome's locale
codes: `de`, `zh_CN`, `pt_BR`, …), translate every `message`, and run `npm test`. Rules:

- **`catalogLocale`** is not shown: set it to the folder name with `-` instead of `_` (`de`, `zh-CN`).
  Plural rules and the pages' `lang` (which picks CJK / Devanagari fonts) follow it.
- **Placeholders** have names: `"message": "Look up $SITE$"` with `"placeholders": {"site": {"content":
  "$1", "example": "github.com"}}`. Move `$SITE$` wherever your language needs it; keep the
  `placeholders` block exactly as in English (same names, same `$1`, `$2`).
- **Plurals** are separate messages with a suffix: `badgeLogins_one`, `badgeLogins_other`. Provide the
  categories your language uses (`Intl.PluralRules`: `_zero`, `_one`, `_two`, `_few`, `_many`, `_other`;
  Japanese, Chinese, Korean and Indonesian only `_other`; French and Hindi `_one` and `_other`, French may add `_many`). `$COUNT$`
  is the number, already formatted for the browser's language.
- `description` is for you and is not shown; you may drop it from translated files or keep it.
- Don't translate Passyble, Passyble Server, Sensyble, Windows Hello, card brands, `pass+server://`,
  `otpauth://`, example addresses (`pass.example.com`) or `about:addons` / `chrome://…` addresses. Names of
  the desktop app's menus and buttons (Settings → Autofill, Read QR from picture…) should match the translated desktop app.
- Keep `extName` within 75 characters and `extDescription` within 132 (Chrome Web Store limits).

`npm test` (test/i18n.test.js) checks that every key the code and the HTML use exists in English, that no
English message is unused or lacks a description, that names are valid (`A-Z a-z 0-9 _`, unique ignoring
case), and that every other locale has only English keys, no empty messages, the same placeholders per
message and complete plural sets for its language. It also flags user-visible English left hard-coded
in the UI code.

In code (`src/shared/i18n.js`; `chrome.i18n` works in the background, the extension pages and the
content scripts, so also inside the in-page UI's closed shadow roots):

```js
import { t, tp, applyI18n, formatNumber } from '../shared/i18n.js';
t('popupLookUp', site);                // "Look up github.com"
tp('badgeLogins', n);                  // badgeLogins_one / _other for n; $1 is the formatted n
applyI18n(document);                   // static HTML: data-i18n, data-i18n-title, -placeholder, -aria-label
```

Static HTML keeps its English text as the fallback, with `data-i18n="key"` (text) and
`data-i18n-title` / `data-i18n-placeholder` / `data-i18n-aria-label`. Build sentences as whole messages
with placeholders, never by joining translated pieces. The page's MAIN-world script
(`webauthn-main.js`) shows no text; its `DOMException` messages mirror the browser's own WebAuthn errors.
Under Node, `test/helpers/i18n-setup.js` (loaded by `npm test`) provides `getMessage` backed by
`_locales/en`, so tests assert the English text.

## Development

```sh
npm install        # jsdom, for tests only
npm test           # node --test test/*.test.js (with test/helpers/i18n-setup.js: chrome.i18n from _locales/en)
npm run test:e2e   # opt-in: the 2FA chain in Playwright's Chromium (npm i --no-save playwright; HEADED=1 to watch)
npm run check      # node --check on every JS file
npm run icons      # regenerate icons/ (svg + png) from the logo in ../Core/desktop-app/scripts/make-icons.mjs
npm run firefox-manifest
npm run package    # dist/pass-chromium.zip and dist/pass-firefox.zip for the stores, dist/pass-chromium-unpacked.zip to sideload
                   # (needs src/wasm/ built; stops with a message otherwise)
npm run wasm       # the browser vault's WebAssembly into src/wasm/ (git-ignored), see below
npm run release    # npm run wasm, then npm run package
```

**Google Drive.** Nothing to set up here: the extension has no Google OAuth client. Google Drive sync is the
Passyble desktop app's (Core/docs/GOOGLE_DRIVE.md); the extension uses the app's vaults in app mode. A
`google-oauth.env` or `src/google-client.json` left over from older builds is git-ignored and never packaged; it can
be deleted.

**The browser vault's WebAssembly** (Core/docs/EXTENSION_VAULT.md): `wasm/` is the Rust crate
`pass-wasm` (pass-agent's request handlers and pass-core, exported with wasm-bindgen) and `wasm/kdf/` the
separate Argon2id module `pass-wasm-kdf`. `npm run wasm` (`scripts/build-wasm.mjs`) needs Rust with
`rustup target add wasm32-unknown-unknown`; it builds both in ../Core (through `..\Core\scripts\cargo-low.cmd` on Windows;
`--core <dir>` or `PASSYBLE_CORE_DIR` picks another Passyble-Core checkout, and its own `scripts\cargo-low.cmd`), runs
the wasm-bindgen CLI of exactly the crate's pinned version (it installs it with
`cargo install wasm-bindgen-cli --version <v> --locked` if missing; `--no-install` only prints that command),
runs `wasm-opt` when Binaryen is on the PATH, and writes `src/wasm/pass_wasm.js`, `pass_wasm_bg.wasm`,
`pass_wasm_kdf.wasm` and `kdf.js` (the host side of the Argon2id module). The background imports them statically (an
MV3 service worker can't `import()` later), so the extension needs them built to start; `npm run package` requires
them and never ships `wasm/`. Under `npm test`, `test/helpers/wasm-hooks.js` resolves `src/wasm/` to a JavaScript
stand-in (`test/helpers/fake-wasm.js`) when it isn't built, or when a test asks for it
(`globalThis.__passFakeWasm`, the background's browser-mode tests); `test/local-wasm.test.js` runs the LocalBackend on
the real modules. Tests: `cargo test -p pass-wasm` (natively), the same tests as
WebAssembly in Node with `CARGO_TARGET_WASM32_UNKNOWN_UNKNOWN_RUNNER=wasm-bindgen-test-runner cargo test
--target wasm32-unknown-unknown -p pass-wasm`, and `test/wasm-agent.test.js` against the built `src/wasm/`
(skipped when it isn't built).

`test/e2e/browser-vault.e2e.mjs` runs browser mode in Playwright's Chromium with a throwaway profile and no Passyble
app (`test/e2e/no-native-host.js` makes `connectNative` answer "host not found", so no native host on the test PC is
ever started): the setup guide's choice, creating a vault in this browser (with the "only on this device" notice), a
login saved from the save prompt and filled from the menu, Options and a backup download, lock and unlock, a service
worker stopped over CDP that comes back unlocked, and deleting the vault and opening the backup again. `SHOTS=<dir>`
saves screenshots of the setup guide, the popup and Options in light and dark.

`test/e2e/otp.e2e.mjs` loads a copy of the extension into Playwright's Chromium whose service worker first imports
`test/e2e/native-stub.js`: it replaces `runtime.connectNative('com.sensyble.pass')` with a port that posts each
request to a mock agent (`test/e2e/mock-agent.mjs`: real RFC 6238 codes, the agent's pending-OTP and URL rules).
Everything else (content scripts, background, NativeClient) is the shipped code. `test/e2e/pages.mjs` serves the
sign-in and 2FA pages on `http://*.example.localhost`. Without Playwright the test is skipped.

`test/e2e/passkey.e2e.mjs` registers and signs in on local pages the way sites do (webauthn.io's
SimpleWebAuthn flow with a conditional request on load, GitHub's `allowCredentials`, passkey autofill
through the Passyble menu, `excludeCredentials`, a cross-origin iframe, RSA-only, an aborted signal,
the content scripts injected twice, the cards of a page's own `get()` / `create()` clicked like a user would,
**Use another device** and Cancel), with a SimpleWebAuthn-style client (`passkey-client.js`: hand-converted
options and the getters, `toJSON()`, or the browser's `parse*FromJSON`). A relying party in the test
(`webauthn-rp.mjs`) verifies every response like a server: clientDataJSON, rpIdHash, flags, the `none`
attestation, the COSE key against `getPublicKey()`, and each ES256 signature. The authenticator is the
**real desktop agent** when its harness is built (`cargo build -p pass-agent --example e2e_agent`; found
under `$CARGO_TARGET_DIR` or `target/debug/examples`, or set `PASS_E2E_AGENT`), else the mock agent's
software authenticator. The browser's own WebAuthn never runs for real: every page and frame gets a stand-in for
its methods (it records the call and says no), behind it a CDP virtual authenticator (one CDP session per page,
never detached: a detached second session can switch it off), and Chromium runs with
`--disable-features=WebAuthenticationUseNativeWinApi`, so no Windows Security dialog can open. With the mock agent
it also covers a vault fully locked after boot (the card's **Unlock**, a page asking as it loads), no passkey for
the site and Passyble not running (**Open Passyble**). `PASSKEY_SHOTS=<dir>` saves screenshots of those cards.
`BROWSER=<path to brave.exe>` runs it in Brave (a temporary profile).

## Reporting a security issue

Please don't open a public issue for a vulnerability. How to report one privately, what to include and how
fast we aim to answer are in [SECURITY.md](SECURITY.md), also published at
https://passyble.com/vulnerability-disclosure/.

Dependencies are checked for known vulnerabilities on every push and pull request and every Monday
(`.github/workflows/dependency-audit.yml`: npm audit and osv-scanner), and Dependabot (`.github/dependabot.yml`)
proposes updates weekly.

## Licence

Copyright (C) 2026 Sensyble LLC

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General
Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option)
any later version. This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See [LICENSE](LICENSE)
for the full text.

Third-party parts keep their own licences: [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). Contributions
are welcome under a Contributor License Agreement ([CONTRIBUTING.md](CONTRIBUTING.md)). The Passyble and
Sensyble names and logos are trademarks of Sensyble LLC ([TRADEMARKS.md](TRADEMARKS.md)).

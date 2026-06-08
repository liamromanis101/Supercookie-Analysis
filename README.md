# Favicon Supercookies — What They Are, Which Browsers Are Affected, and How to Find Them

So you've probably heard about tracking cookies. You clear them, you use incognito mode, maybe you run a VPN — and you feel pretty good about your privacy. There's a problem though. There's a tracking technique that laughs at all of that.

It's called a **favicon supercookie**, and it exploits something you'd never think twice about: those tiny little icons in your browser tab.

---

## What's Actually Going On

When you visit a website, your browser downloads the site's favicon (the small icon you see in the tab bar) and stores it in a local database called the **favicon cache**, or F-Cache. This is completely normal behaviour — it means your browser doesn't have to re-download the icon every time you visit.

The problem is that the F-Cache is separate from your regular browser cache and cookies. It has its own database file, and it doesn't get cleared when you wipe your cookies or browsing history.

A researcher named [Jonas Strehle](https://github.com/jonasstrehle/supercookie) demonstrated how this can be turned into a tracking system. Here's the rough idea:

1. When you first visit a site, the server sends your browser on a series of redirects to different subpaths (think `/a/`, `/b/`, `/0/`, `/1/`). Some of those paths return a favicon, some return nothing.
2. The pattern of hits and misses — which paths your browser requested a favicon for and which it didn't — encodes a unique binary ID.
3. That ID gets baked into your F-Cache.
4. When you come back, the server watches which favicon requests come in and which don't (the cached ones are silent). It reconstructs your ID from that pattern and recognises you — no cookie needed.

The technique can theoretically distinguish **2^N unique users**, where N is the number of redirects used. It works across regular and incognito sessions, survives reboots, and isn't touched by ad blockers or VPNs.

This was originally described in an academic paper from the University of Illinois Chicago, presented at NDSS 2021: *"Tales of Favicons and Caches: Persistent Tracking in Modern Browsers"*.

---

## What Doesn't Protect You (Contrary to Popular Belief)

| Method | Stops Favicon Supercookies? |
|---|---|
| Clearing cookies | ❌ No — F-Cache is a separate database |
| Clearing browser cache | ❌ No — F-Cache is separate |
| Incognito / Private mode | ❌ No — most browsers share F-Cache across modes |
| VPN | ❌ No — changes your IP, not your local cache |
| Ad blockers (uBlock, AdBlock Plus) | ❌ No — they block known tracker domains, not this |
| Restarting your computer | ❌ No — F-Cache persists on disk |

---

## Browser Protection Status

The actual fix has to come from the browser vendors themselves, through a technique called **cache partitioning** — essentially, each site gets its own isolated F-Cache bucket so one site can't read what another site wrote.

| Browser | Protected? | Notes |
|---|---|---|
| 🦊 Firefox 85+ | ✅ Yes | Full cache partitioning shipped in Jan 2021 |
| 🦁 Brave (Desktop) | ✅ Yes | Passes all state partitioning tests |
| 🧭 Safari (Desktop) | ✅ Yes | Favicon cache partitioned |
| 🔒 Tor Browser | ✅ Yes | Firefox base + aggressive isolation |
| 🌐 Chrome | ⚠️ Partial | Storage partitioning expanding but not complete |
| 🔵 Edge | ⚠️ Partial | Mirrors Chrome's behaviour |
| 🦁 Brave (iOS) | ⚠️ Partial | Open issue to partition favicon cache on iOS |
| 🧭 Safari (iOS) | ⚠️ Partial | ITP helps, but protections are inconsistent |
| 🦊 Firefox (Android) | ⚠️ Partial | Favicon cache not yet partitioned on mobile |

Firefox's response is worth highlighting — in January 2021 they shipped partitioning across the HTTP cache, image cache, favicon cache, HSTS, OCSP, font cache, DNS cache, and more, all in one go. It was a comprehensive answer to the whole class of supercookie attacks, not just this one.

---

## How to Check Your Own System

The F-Cache is a SQLite database file sitting on your disk right now. Here's how to inspect it.

### Linux

**Find the database files:**

```bash
# Chrome / Chromium
find ~/.config/google-chrome -name "Favicons" 2>/dev/null
find ~/.config/chromium -name "Favicons" 2>/dev/null

# Edge
find ~/.config/microsoft-edge -name "Favicons" 2>/dev/null

# Brave
find ~/.config/BraveSoftware -name "Favicons" 2>/dev/null

# Firefox (uses a different format)
find ~/.mozilla/firefox -name "favicons.sqlite" 2>/dev/null
```

**Query the contents:**

```bash
# Install sqlite3 if you don't have it
sudo apt install sqlite3

# Dump favicon URLs from Chrome / Edge / Brave
sqlite3 ~/.config/google-chrome/Default/Favicons \
  "SELECT url FROM icon_mapping ORDER BY last_updated DESC;"

# Firefox
sqlite3 ~/.mozilla/firefox/*.default-release/favicons.sqlite \
  "SELECT url FROM moz_pages_w_icons ORDER BY expire_ms DESC;"
```

### Windows

**Find the database files (PowerShell):**

```powershell
# Chrome
Get-ChildItem "$env:LOCALAPPDATA\Google\Chrome\User Data" -Recurse -Filter "Favicons"

# Edge
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\Edge\User Data" -Recurse -Filter "Favicons"

# Brave
Get-ChildItem "$env:LOCALAPPDATA\BraveSoftware\Brave-Browser\User Data" -Recurse -Filter "Favicons"

# Firefox
Get-ChildItem "$env:APPDATA\Mozilla\Firefox\Profiles" -Recurse -Filter "favicons.sqlite"
```

**Query the contents (requires sqlite3.exe):**

```powershell
# Install via winget
winget install SQLite.SQLite

# Chrome / Edge / Brave
sqlite3 "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Favicons" `
  "SELECT url FROM icon_mapping ORDER BY last_updated DESC;"

# Scan all Chrome profiles at once
Get-ChildItem "$env:LOCALAPPDATA\Google\Chrome\User Data" -Recurse -Filter "Favicons" |
ForEach-Object {
    Write-Host "`n=== $($_.FullName) ===" -ForegroundColor Cyan
    sqlite3 $_.FullName "SELECT url FROM icon_mapping ORDER BY last_updated DESC LIMIT 50;"
}
```

### What Does a Suspicious Entry Look Like?

Normal favicon entries look like this:

```
https://github.com/favicon.ico
https://google.com/favicon.ico
https://bbc.co.uk/favicon.ico
```

A supercookie attack leaves a very different pattern — the **same domain** appearing across many sequential subpaths:

```
https://tracker.example.com/0/
https://tracker.example.com/1/
https://tracker.example.com/a/
https://tracker.example.com/b/
https://tracker.example.com/id/00001/
https://tracker.example.com/id/00010/
```

That sequential subpath structure is the giveaway. Each path represents one bit of your encoded fingerprint.

---

## Clearing the Cache

If you want to wipe it, close your browser first, then:

**Linux:**
```bash
# Chrome
rm ~/.config/google-chrome/Default/Favicons
rm ~/.config/google-chrome/Default/Favicons-journal

# Edge
rm ~/.config/microsoft-edge/Default/Favicons
rm ~/.config/microsoft-edge/Default/Favicons-journal
```

**Windows (PowerShell):**
```powershell
# Chrome
Remove-Item "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Favicons"
Remove-Item "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Favicons-journal"

# Edge
Remove-Item "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Favicons"
Remove-Item "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Favicons-journal"
```

The browser will recreate the file cleanly on next launch.

---

## The Bigger Picture

This specific technique is a proof-of-concept — Jonas Strehle built it to demonstrate the vulnerability, not to deploy it at scale. But it's part of a broader family of **cache-based supercookies** that have been actively exploited over the years: ETags, HSTS flags, Flash storage, and now favicons.

The lesson isn't really about favicons specifically. It's that browsers are extraordinarily complex pieces of software with dozens of caching layers, and any one of them can potentially be abused for tracking. The fixes require browser vendors to rethink their caching architecture from the ground up — which is exactly what Firefox did in 2021, and what others have been slowly following.

If you're on Firefox, Brave desktop, or recent Safari, you're in good shape. If you're on Chrome or Edge, the protections are improving but you're not fully covered yet. On mobile, the picture is patchier across the board.

---

## References

- [jonasstrehle/supercookie](https://github.com/jonasstrehle/supercookie) — the original proof-of-concept
- [NDSS 2021 Paper](https://par.nsf.gov/servlets/purl/10268961) — Tales of Favicons and Caches: Persistent Tracking in Modern Browsers
- [Mozilla Security Blog](https://blog.mozilla.org/security/2021/01/26/supercookie-protections/) — Firefox 85 supercookie protections
- [PrivacyTests.org](https://privacytests.org) — ongoing automated browser privacy testing
- [Cover Your Tracks (EFF)](https://coveryourtracks.eff.org) — test how trackable your browser is

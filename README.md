# FIXED: Photoshop CS3 crashing / won't reinstall — Error 1920 "Bonjour Service failed to start" caused by Nikon Wireless Transfer Utility

**TL;DR:** If Adobe Photoshop CS3 suddenly stops launching on Windows 10, and reinstalling it fails with **MSI Error 1920: Service 'Bonjour Service' failed to start**, check whether you have a **Nikon wireless transfer utility** installed. Its bundled mDNS component conflicts with Bonjour and can cause `mDNSResponder.exe` to fail to start with the generic, unhelpful error code **4294967295 (0xFFFFFFFF)**. Uninstalling/disabling the Nikon utility resolved it for me.

---

## The setup

- Windows 10 (build 19045.x)
- Adobe Photoshop CS3 (10.0), installed since 2007, working fine until one Friday
- No manual changes made to the system beforehand

## Symptom 1: Photoshop CS3 suddenly stopped launching

No warning, no update, no new install — it just stopped opening. Checked Windows crash reports and found a real CS3 crash dump (not to be confused with a *different* crash dump for Photoshop 2022 that also showed up — make sure you're reading the report for the right version if you have multiple Photoshop installs).

The CS3 crash report showed:

```
Application: Adobe Photoshop CS3
Path: C:\Program Files (x86)\Adobe\Adobe Photoshop CS3\Photoshop.exe
Faulting module: C:\Windows\System32\comdlg32.dll
Exception code: 0xC000041D
```

Photoshop.dll, PSViews.dll, and various other CS3 components were loading fine — the crash was specifically in the interaction with `comdlg32.dll`, a Windows common-dialog component. This pointed toward a **Windows-side change**, not a CS3-specific issue, especially given it had worked for years without incident.

## Confirming Windows corruption

Ran, as Administrator:

```
sfc /scannow
```

Result: **Windows Resource Protection found corrupt files and successfully repaired them.** So there was genuine system-file corruption, likely explaining the sudden `comdlg32.dll`-related crash.

The CBS/SFC log also flagged overlapping ownership/security warnings on directories like `C:\Program Files (x86)\` and the Start Menu folders. These turned out to be a secondary concern and not the main blocker — noted here in case others see the same thing in their logs.

## Symptom 2: Reinstalling CS3 made things worse

While troubleshooting, I tried reinstalling Photoshop CS3 to test against the (now-repaired) Windows files. The installer **failed partway through**:

```
Adobe Photoshop CS3 — Component install failed
Shared components — Component install failed
```

Checking installed programs afterward showed a bunch of CS3 sub-components (Bridge, Device Central, Camera Raw, etc.) with today's install date, but the main **Adobe Photoshop CS3** entry still showed its original 2007 install date — meaning the reinstall was incomplete, and now CS3 wasn't runnable at all.

## Symptom 3: MSI Error 1920 — Bonjour Service

Running the installer again produced a much more specific error:

```
Error 1920: Service 'Bonjour Service' failed to start.
```

CS3's Adobe Version Cue component bundles Apple's old Bonjour service for network discovery. Checking it:

```
sc query "Bonjour Service"
```

showed the service existed but was **stopped**, and its binary path (`C:\Program Files (x86)\Bonjour\mDNSResponder.exe`) **didn't actually exist on disk** — an orphaned service entry. Deleted it:

```
sc delete "Bonjour Service"
```

Re-running the CS3 installer still hit Error 1920 — it was trying to install/start Bonjour fresh and still failing.

## The real fix path

1. Ran a full Windows repair pass:
   ```
   DISM /Online /Cleanup-Image /RestoreHealth
   sfc /scannow
   ```
   then rebooted. (DISM repairs the underlying component store that SFC repairs *from* — running both, in that order, catches cases where a first SFC pass can't find good source files locally.)

2. Installed **Apple's official Bonjour Print Services** installer separately (from apple.com — not the outdated copy bundled in the CS3 installer), so a working Bonjour service would already exist before CS3's installer tried to touch it.

3. Tried the CS3 install again. New error:
   ```
   'Bonjour Service' failed to start.
   Verify that you have sufficient privileges to start system services.
   ```
   This message is misleading — it's the Service Control Manager's generic fallback text, not necessarily an actual permissions problem.

4. Tried starting the service manually:
   ```
   net start "Bonjour Service"
   ```
   Got error code **4294967295** (which is `0xFFFFFFFF`, i.e. -1 as unsigned 32-bit) — a signature of the service process crashing/terminating before it could report a real Win32 error, not a genuine privilege issue.

## Root cause: Nikon wireless transfer software

Checking installed software for anything else touching mDNS/Bonjour-style network discovery turned up a **Nikon wireless transfer utility** (used for Wi-Fi image transfer from a Nikon camera). It runs its own mDNS responder component in the background, and it was conflicting with Bonjour trying to bind/start — hence the instant crash with no meaningful error code from the SCM.

**Removing/disabling the Nikon utility's background service resolved the Bonjour startup failure immediately**, and the CS3 installer completed successfully after that.

## Summary of the actual root cause chain

1. Windows system files got corrupted (cause unknown/unconfirmed) → caused the original `comdlg32.dll` crash in CS3.
2. Attempting to fix it by reinstalling CS3 broke the existing install without completing a new one.
3. The reinstall then got blocked by an unrelated, pre-existing conflict: **Nikon's camera transfer software was silently squatting on mDNS resources that Bonjour needed**, which had nothing to do with Photoshop, Windows corruption, or Adobe at all.

If you're dealing with **any** software that bundles Bonjour/mDNSResponder (this isn't just an Adobe/CS3 thing — anything using Bonjour for network discovery can hit this) and you get a generic "insufficient privileges" or error code **4294967295** when the service tries to start, check for other mDNS-capable software running in the background — camera transfer utilities (Nikon, Canon), printer software, iTunes, and similar tools are common culprits.

---

*Posting this in case it saves someone else the same multi-day rabbit hole.*

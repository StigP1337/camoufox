# Firefox 140.5.0 ESR Upgrade Notes

## Platform-Specific Notes

### macOS: BSD Patch vs GNU Patch Compatibility

**Issue:** macOS ships with BSD patch (v2.0-12u11-Apple) which is significantly stricter about context matching than GNU patch (v2.7+). Some patches may fail on macOS due to context line mismatches, even when the patches apply cleanly on Linux.

**Root Cause:** BSD patch requires exact context matches, while GNU patch handles "fuzzy" matching and line offsets more gracefully. When Firefox adds/removes lines near patch contexts (e.g., new `SPHINX_TREES` entries in build files), BSD patch fails while GNU patch succeeds.

**Solution:** Install GNU patch via Homebrew:
```bash
brew install gpatch
export PATH="/opt/homebrew/opt/gpatch/libexec/gnubin:$PATH"
make git-dir
```

**Alternative:** Regenerate patches with updated Firefox 140.5.0 context (see config.patch fix in this document for example workflow).

## Current Status

✅ **Fixed Patches:**
- `patches/playwright/0-playwright.patch` - Playwright base integration (applied cleanly)
- `patches/playwright/1-leak-fixes.patch` - Stealth fixes (fixed manually, navigator.webdriver + enterprise policies)
- `patches/ghostery/Disable-Onboarding-Messages.patch` - Applied with offset
- `patches/all-addons-private-mode.patch` - Applied with fuzz
- `patches/anti-font-fingerprinting.patch` - Applied with offset
- `patches/audio-context-spoofing.patch` - **FIXED** - Updated for new SPHINX_TREES line in dom/media/moz.build
- `patches/font-hijacker.patch` - **FIXED** - Updated for Firefox 142 build system changes (CONFIGURE_SUBST_FILES block removed from layout/style/moz.build)
- `patches/force-default-pointer.patch` - Applied with 23-line offset (forces browser to always report fine pointer capabilities)
- `patches/geolocation-spoofing.patch` - **FIXED** - Updated for Firefox 142 GTK configuration block changes and corrected LOCAL_INCLUDES alphabetical ordering ("/camoucfg" must come before "/dom/base" and "/dom/ipc") in dom/geolocation/moz.build
- `patches/global-style-sheets.patch` - Applied with 20-line offset
- `patches/librewolf/ui-patches/handlers.patch` - Applied cleanly (removes 190 lines of default handlers)
- `patches/librewolf/ui-patches/hide-default-browser.patch` - Applied with 6-line offset
- `patches/locale-spoofing.patch` - Applied with fuzz 2 and 22-line offset (implements locale spoofing)
- `patches/media-device-spoofing.patch` - Applied with 3-line offset
- `patches/librewolf/mozilla_dirs.patch` - Applied with fuzz and offsets (modifies Mozilla directory paths)
- `patches/network-patches.patch` - Applied with significant offsets (25-153 lines)
- `patches/no-css-animations.patch` - Applied with 1-line offsets (disables/modifies CSS animations)
- `patches/no-search-engines.patch` - **FIXED** - Updated for Firefox 142 UrlbarProviderInterventions.sys.mjs line number shifts
- `patches/pin-addons.patch` - Applied with significant offsets (937 and 163 lines)
- `patches/librewolf/ui-patches/remove-branding-urlbar.patch` - Applied with 203-line offset

- `patches/librewolf/ui-patches/remove-cfrprefs.patch` - **FIXED** - Updated for Firefox 142 dynamic settings system (removes CFR preferences from main.js config)
- `patches/librewolf/ui-patches/remove-organization-policy-banner.patch` - Applied with 8-line offset
- `patches/camoufox-branding.patch` - Added new patch to fix the MOZ_APP_VENDOR, MOZ_APP_PROFILE build error by adapting to Firefox 142's configuration system changes (Bug 1898177), following the "broken patch" workflow for creating patches.
- `patches/config.patch` - **FIXED (macOS compatibility)** - Updated patch context for Firefox 142 build system changes. Firefox added two new SPHINX_TREES entries (`content-security` and `jsloader`) to root `moz.build`, shifting line numbers from 220 to 232. Old patch context expected `update-infrastructure` as last SPHINX_TREES entry, but new context includes the additional entries. This caused BSD patch (macOS default) to fail due to strict context matching, while GNU patch succeeded with fuzzy matching. Regenerated patch with correct Firefox 142 context - now applies cleanly with both BSD and GNU patch.
- `patches/webgl-spoofing.patch` - **FIXED** - Added explicit `static_cast<uint8_t>()` casts for GetShaderPrecisionFormat to fix narrowing conversion errors (Firefox 142 changed ShaderPrecisionFormat fields to uint8_t while MaskConfig returns int32_t)

✅ **Removed/Obsolete Patches:**
- `patches/librewolf/sed-patches/allow-searchengines-non-esr.patch` - **DELETED** - Firefox 142 natively supports SearchEngines in non-ESR builds (Bug 1961839, April 2025)
- `patches/librewolf/remove_addons.patch` - **DELETED** - LibreWolf privacy patch that conflicts with Camoufox stealth mission (removing "Report Broken Site" makes browser detectably different from vanilla Firefox).  Plus the entire subsystem in firefox has changed and updating to that is out of scope for this mission.
❌ **Remaining Patches (need testing):**
- All other Camoufox patches (40 remaining - testing in progress)

**Next Step:** Continue testing remaining patches with `make dir`.

## The Problem

Playwright's `bootstrap.diff` patch for Firefox 140.5.0 ESR is designed for a specific git commit from the Firefox GitHub mirror, **NOT** Mozilla's official release tarball.

- **Mozilla's tarball**: `https://archive.mozilla.org/pub/firefox/releases/140.5.0esr/source/firefox-140.5.0esr.source.tar.xz`
- **Playwright's source**: GitHub mirror commit from their release branch

Even though both are "Firefox 140.5.0 ESR", they're slightly different, causing **multiple reject files** when applying Playwright's patch to Mozilla's tarball.

## The Solution

**Use Playwright's exact Firefox source** instead of Mozilla's tarball.

### What We Did

1. **Cloned Playwright's exact Firefox commit:**
   ```bash
   git clone --filter=blob:none --no-checkout git@github.com:mozilla-firefox/firefox.git camoufox-140.5.0-fork.27
   cd camoufox-140.5.0-fork.27
   git checkout <PLAYWRIGHT_COMMIT_HASH>
   ```

2. **Created the `unpatched` tag** (for `make revert`):
   ```bash
   git tag -a unpatched -m "Initial commit"
   ```

3. **Copied Camoufox additions** (spices & herbs):
   ```bash
   bash ../scripts/copy-additions.sh 140.5.0 fork.27
   ```

4. **Committed additions and updated tag:**
   ```bash
   git add -A
   git commit -m "Add Camoufox additions"
   git tag -f -a unpatched -m "Initial commit with additions"
   ```

5. **Applied patches:**
   ```bash
   make dir  # Should now work cleanly with Playwright's patch
   ```

## Why This Works

- Playwright's `bootstrap.diff` expects specific line numbers and code structure from their exact commit
- By using the same source Playwright uses, the patch applies cleanly (no rejects)
- The other Camoufox patches (fingerprinting, spoofing, etc.) still apply fine since they target Firefox APIs, not specific line numbers

## The Trade-off

**Normal approach (what Camoufox used before):**
- Download Mozilla's official tarball
- Manually fix patch conflicts when upgrading
- More work, but using "official" Firefox releases

**Our approach (what we're doing now):**
- Use Playwright's git commit
- Patches apply cleanly
- Less work, but using GitHub mirror instead of Mozilla's official source

## Finding the Right Commit

For future upgrades, find Playwright's Firefox commit:

1. Check Playwright's `UPSTREAM_CONFIG.sh`:
   ```bash
   curl -s https://raw.githubusercontent.com/microsoft/playwright/release-1.XX/browser_patches/firefox/UPSTREAM_CONFIG.sh
   ```

2. Look for `BASE_REVISION`:
   ```
   REMOTE_URL="https://github.com/mozilla-firefox/firefox"
   BASE_BRANCH="release"
   BASE_REVISION="<COMMIT_HASH_HERE>"
   ```

3. Use that commit hash when cloning.

## Backup Plan

If this approach causes issues, we still have:
- **Backup directory**: `camoufox-140.5.0-fork.27.bak` (Mozilla tarball source)
- **Downloaded tarball**: `firefox-140.5.0-esr-playwright.tar.gz` (Playwright's commit as tarball)
- Can go back to manual patch fixing if needed

## Understanding the Dual-Repo Structure

**Camoufox uses TWO git repositories:**

1. **Camoufox Project Repo** (outer repo at `/home/azureuser/camoufox/`)
   - Contains: patches, scripts, Makefile, documentation
   - `.git/` tracks: patch files, build scripts, configuration
   - **Commit patch changes here** after fixing them

2. **Firefox Source Repo** (inner repo at `camoufox-140.5.0-fork.27/`)
   - Contains: Firefox source code
   - `.git/` tracks: Firefox code + Camoufox additions
   - Used for: generating patches via `git diff`, reverting with `make revert`

**Key workflow:**
- Fix broken Firefox files → `cd camoufox-140.5.0-fork.27 && git diff > ../patches/foo.patch`
- Commit the patch file → `cd .. && git add patches/foo.patch && git commit`

For the general workflow of applying patches, fixing broken patches, and using checkpoints, see [WORKFLOW.md](./WORKFLOW.md).


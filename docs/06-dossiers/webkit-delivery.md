# Dossier — Web applet / WebKit and its delivery rig

Tier C on score, reserve on strategy. Best *complement* to a kernel bug.

## 1. Reframing

The bug is not the hard part. WebKit is the most fuzzed code on Earth, so the bug you find will be a variant of a public one, and Nintendo does backport for it. The hard part is:

1. **Privilege.** The `web` applet is an applet. No `fsp-srv`, no `sockets`, no `am` in its own handle table, heavily reduced syscall mask. JSC gets JIT, nothing else does.
2. **Delivery.** Stock trusts only the system CA store, so HTTPS interception needs a Nintendo name or a weak-validation path.
3. **Pivot.** Getting out of the applet is a second exploit. Historically this was always a kernel bug.

Get a kernel bug first and this track becomes trivial to weaponise. That is the strategy: keep it warm, do not lead with it.

## 2. Delivery rig (the part people get wrong)

The friction is solvable and the solution is "be the router."

- ESP32 as a fake AP on the `nintendo`-style SSID pattern, with DHCP, DNS, and a captive portal.
- Answer the connectivity probe. **Verify whether `ctest.cdn.nintendo.net` on 20.x is still plaintext HTTP and whether a 200 from it is trusted enough to open the portal page.** If it is HTTPS-validated, the rig must fall back to a user-initiated link (News, eShop, NSO) and the friction rises to "tap this news item".
- Portal page fingerprints the WebKit build, then serves a version-matched payload.
- DNS: answer the probe name, `*.nintendo.net`, and everything else to the AP.

User action required: join a Wi-Fi network. That is lower friction than a soldering iron and it needs no key material and no physical contact.

Fallback channels, in order of friction: news item link, eShop page, NSO login, a QR from the Album share flow (console serves to *phone*, so that one attacks the phone — useful for the phone, not the console).

## 3. Build tracking

- Identify the `nn_webkit` / WebKit build per firmware. Record it in a table keyed by version.
- Fingerprint from the page: UA string, JS feature probes (JSC version, WASM features, `SharedArrayBuffer`, `BigInt`, `RegExp` v-flag), CSS support, error-string differences.
- Diff `nn_webkit` across versions the same way `04-methods.md` §4.1 diffs `Kernel`. A JSC commit reverted or a DOM fix backported is a queue entry.

## 4. Bug classes, by historical yield

- **JSC:** DFG/FTL JIT type confusion (the classic; `Array.prototype`, `arguments`, `Math` builtins, `RegExp` IR). Needs JIT privilege, which the applet has.
- **DOM:** node lifetime, `Range`, `Selection`, `cloneNode`, `adoptedStyleSheets`, `MutationObserver` re-entrancy.
- **WASM:** memory sizing, trap handlers, table growth.
- **Media:** `AudioContext`, `MediaSource`, decoder paths — shares surface with the Album track.
- **Networking:** `fetch`, `XMLHttpRequest`, `Range`, MIME sniffing.

## 5. Harness

- Laptop with `dnsmasq` + `nginx`, AP, console joined once. Iterate on hardware, not in a desktop browser — the build is years behind and the sandbox differs.
- Serve a corpus page that runs one variant per load and reports back over `fetch` (the applet has network).
- Capture `fatal` with camera + `twili`.
- Port the rig to ESP32 only once a payload works on the laptop.

## 6. Known-unknowns

- Current WebKit branch and backport cadence.
- Whether the probe is plaintext and whether its result can open a page.
- Whether the `web` applet still has `fsp-srv` for cache (affects post-exploitation).
- Whether `SharedArrayBuffer` and JIT primitives are enabled for the applet's content security policy.
- Whether eShop/NSO embed a separate WebKit or the same `nn_webkit`.

## 7. First three experiments

1. Join the fake AP on a stock unit, confirm portal open, record the exact probe behaviour.
2. Fingerprint the build and pin it to a WebKit commit.
3. Diff `nn_webkit` between the two most recent firmwares and label every JSC-shaped change.

## 8. Status

Reserve. Start the device-side rig build in P1, but do not chase a bug until either K0 yields or a tier-A track dies.

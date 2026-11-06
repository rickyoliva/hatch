# Dossier — Peripherals, wireless and library applets

A bundle of mid-tier tracks, kept together because they share hardware (RP2040/ESP32 fleets) and because several are sysmodule-level.

## 1. `usb` — host side (tier A)

Plug-in, no keys, no interaction, sysmodule privilege. Two parsers in series:

1. **Core descriptor parsing** — device descriptor, config/interface/endpoint descriptors, string descriptors, `bMaxPacketSize0`, HID report descriptors, class-specific descriptors. Length fields, `wTotalLength`, interface counts, endpoint counts.
2. **Class driver logic** — AX88179 / RTL8153 LAN adapters (bulk IN/OUT, `nifm` handoff, MAC/PHY register reads over the vendor control pipe) and the GameCube adapter (37-byte input frames, origin/stick calibration, a calibration state machine).

An RP2040 can present as either. Start with the GC adapter: the protocol is documented, tiny, and the driver is simple enough to read in an afternoon. LAN adapters are the richer target but need more of the stack emulated to reach the interesting code.

Bug classes: `wTotalLength` vs actual buffer, endpoint count overflow, descriptor type confusion, control-transfer `wLength` mismatch, calibration blob assumed to be 3×3 bytes.

## 2. `usb` — device side (tier A, cheaper delivery)

When the console is plugged into a PC it is the USB *device*. A hostile host owns every setup packet: `GET_DESCRIPTOR`, `SET_CONFIGURATION`, class requests, stall/recovery sequencing, and the enumeration timing. This needs the user only to "plug into this laptop", and it attacks the console's device stack rather than a dock peripheral.

Bug classes: setup-packet handling before configuration, descriptor request for an index that does not exist, control-transfer data-stage length mismatch, re-enumeration storms.

## 3. `nifm` / 802.11 / DHCP (tier A)

The only contactless entry.

- **Beacon and probe-response IEs** — SSID, supported rates, RSN/WPA2/WPA3, vendor IEs, malformed lengths, duplicate IEs, `RSN` cipher-suite count arithmetic.
- **Association and EAP** — association response parsing, EAP method negotiation, EAP-TLS certificate chain handling.
- **DHCP** — option 119 domain search (compression pointers are a classic), option 121 classless routes, overload, option length arithmetic.
- **DNS** — name decompression pointer loops, EDNS0, oversized response.

Bug classes: IE length vs remaining frame, compression pointer loops, option length overflow, EAP fragment reassembly.

## 4. `nfp` / amiibo (tier B)

A modded console writes arbitrary bytes to an NTAG215 with a valid HMAC (amiibo keys are public), then hands the tag to a stock unit. No network, no target keys, and games parse the app area as a rich struct — SSBU Figure Players are a serialised AI blob.

Also reachable: the APDU layer (`nfp:user`), tag-type detection, multi-read state machine, removal/re-tap handling, and the app-area HMAC path itself.

Bug classes: app-area length vs declared block count, APDU `Le`/`Lc` handling, re-entrancy on rapid re-tap, tag-type confusion.

## 5. `gc` / gamecard (tier B)

A MIG-class card presents a spoofed card. `gc` parses `KIO`, header, ID set, and a certificate chain. Insert-to-trigger, sysmodule privilege, and — like the card slot — no interaction beyond inserting. Constraint: card auth needs a cloned cert, and per-card uniqueness means a shared cert is a fingerprint.

Bug classes: header length vs declared region size, ID-set count, cert chain depth, command-response size mismatch.

## 6. `hid` (tier C)

Wired controllers parse over USB with less pairing friction than Bluetooth: report descriptors, `0x21`/`0x30`/`0x3F` reports, SPI flash reads, and stick calibration as a **float blob** with per-axis centre/min/max consumed by driver math. Joy-Con firmware update is a HID-level state machine.

Bug classes: SPI read length, calibration array bounds, rumble/amp packet size, report-ID confusion.

## 7. Library applets (tier C, but structurally interesting)

`swkbd`, `myPage`, `web`, `controllerSupport`, `playerSelect`, `photoViewer`, `netConnect`, `cabinet`.

These take a serialised argument blob (`LibAppletArguments` plus a per-applet payload) from the calling **game**, and they exchange `IStorage` over `ILibraryAppletAccessor` that is mapped **writable in both processes**. Consequences:

- The caller is an Application, so the payload is attacker-influenced by any game content the player controls — a save string, a stage name, a URL.
- A bug in the applet yields corruption in a block the game also maps. That is a cross-process primitive with no kernel involvement.
- `swkbd` dictionary paths and inline word lists, `myPage`'s HTML generation from a caller template, and `web`'s `WebArg` (URL, whitelist, footer) are all parsers fed by the caller.

This is the least-documented and most structurally novel corner of the matrix.

## 8. Shared notes

- One RP2040/ESP32 image per persona, slot-driven descriptors from flash.
- Plug/reboot rig; analyser capture on every fault.
- Differential oracle where a reference implementation exists (Linux `usb`/`cfg80211`, `dhcpcd`, `exfatprogs`).
- These tracks cross over: a single gadget presenting as a LAN adapter *and* an SD reader *and* a controller exercises three parsers per insertion.

## 9. Status

`usb` host, `usb` device and `nifm` are tier A and share the P2 harness phase. `nfp` and `gc` are tier B. `hid` and library applets are opportunistic.

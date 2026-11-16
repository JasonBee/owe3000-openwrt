# Stock firmware security review — INEAUTO / OIBIRO OWE3000

This is a summary of what was found in the **stock vendor firmware** of the
OWE3000 (board CPE-AX3000 V3.1, MediaTek MT7981B) during the process of
porting OpenWrt to it. We are publishing this so that other owners of this
device, and of the same white-label board sold under other names, can make an
informed decision about running the stock firmware.

**Short version: do not trust the stock firmware on an untrusted network.
Reflash it with OpenWrt (see the main README) or keep it isolated.**

Live credentials found in the firmware (password hashes, the cracked root
password, a hardcoded Wi-Fi PSK, and a mesh key) are intentionally **redacted**
here. The point is the class of problem, not handing out working secrets for
units still running stock.

## Responsible disclosure

A full, unredacted report is available to the manufacturer and to legitimate
researchers on request. In keeping with responsible-disclosure practice, no
technical details or credentials have been published here, and the unredacted
report has not been sent over any insecure channel.

The manufacturer has been contacted through their official channels with a
brief, findings-free notification offering the detailed report to the
appropriate security or engineering contact:

- the official contact form at https://ineauto.com/pages/contact
- the support email printed in the product insert (ineauto@outlook.com)

The notification deliberately contains no vulnerability details; the full
report will be shared only once a legitimate point of contact responds.

**Disclosure status:** contacted 2026-11-16 — awaiting response. The
redacted summary below was published in parallel; because it contains no
working credentials, it exposes no unit while the vendor is given time to
respond.

---

## Background: why this device was reviewed

This started as basic due diligence, not a hobby teardown. A local nonprofit
was planning to deploy an inexpensive, unlabelled white-box wireless device
bought from a marketplace listing. With prior experience in this space, I tend
to trust recognised market certifications as a baseline of accountability, and
an uncertified, unbranded network device with no clear maker did not meet that
bar. Rather than simply refuse the device and leave the nonprofit to deploy it
anyway, I set out to validate whether it was safe to use.

The review escalated in stages:

1. **Open-source investigation.** Researching the make, model, and board
   characteristics (silkscreen CPE-AX3000 V3.1, the OIBIRO/INEAUTO branding,
   the MT7981B platform) to understand what the hardware actually was.
2. **Administrative and network review.** Looking at the device the way any
   operator could, admin interface, exposed services, and security scanning,
   to see what it exposed on a network.
3. **Firmware analysis.** This is where an AI assistant (Anthropic's Claude /
   Claude Code) materially changed the pace. The stock firmware's root
   filesystem was obfuscated; the key factor was not that the assistant could
   author firmware, but that it **deconstructed the obfuscated firmware binary
   faster than I was achieving with my own security tools on Kali Linux.** The
   accelerated turnaround is what made a deeper review practical on a realistic
   timeline.

Once the obfuscated image was opened quickly and the findings below became
clear, the exercise turned into one of convenience and thoroughness: given how
fast the risks were surfacing, it was worth exploring how complete a
remediation could be, i.e. whether a clean, compatible OpenWrt image could be
built and flashed to overwrite the vendor's compromised build entirely. That
is what the rest of this repository documents.

---

## What the stock firmware is

Under the vendor's "Extender WiFi Settings" web UI, the stock firmware is
itself an OpenWrt build (a relabelled OpenWrt 6.6.x snapshot derived from the
Cudy WR3000 v1 profile), frozen and shipped as the vendor's own version. It
does not receive upstream security updates.

The SquashFS root filesystem had its superblock lightly obfuscated (the magic
bytes changed and the compressor field altered) so that standard tools won't
mount it without a fix. That is not a vulnerability by itself, but it is
consistent with firmware that was not expected to be examined.

---

## Findings

### 1. Hidden second root account (critical)

`/etc/passwd` contained **two** accounts with UID 0. The normal `root`
account was given a non-interactive shell, but an **added** account (a
developer-style name, not a standard OpenWrt account) also had UID 0, GID 0,
and a **real interactive shell**. Anyone who can authenticate to it has full
control of the device. An undocumented root-shell account is backdoor-
equivalent regardless of intent.

### 2. Telnet enabled by default (critical)

The boot script unconditionally starts the BusyBox telnet daemon on every
boot. Telnet is cleartext, there is no option in the UI to disable it, and
port 23 was confirmed open on the running device.

### 3. Trivial root password (critical)

The `root` password hash was MD5-crypt and was cracked **instantly** to a
common default. Although `root`'s own shell was disabled, this password is
what the web/ubus authentication backend checks, so the admin interface and
API accepted it.

### 4. Undocumented always-on service (high)

A non-standard binary runs at every boot and is auto-respawned. It opens a
UDP socket and answers discovery probes, contains command-execution calls,
reads the device's own Wi-Fi config and MAC, and carries identifiers tying it
to an external IoT-CPE auto-provisioning platform (including strings and an
email address from the original ODM). It is an unauthenticated, always-
listening service with command-execution capability and provisioning
heritage.

### 5. Hardcoded Wi-Fi PSK compiled into the firmware (high)

The above binary contains a hardcoded WPA2-PSK string. A credential compiled
into the firmware image is identical on every unit and cannot be rotated by
the user.

### 6. Frozen, unmaintained fork (high)

The firmware will not receive OpenWrt security updates. Any vulnerability in
its kernel or packages after the build date stays unpatched for the life of
the device.

### 7. Hardcoded mesh key (medium)

The roaming/mesh daemon config shipped a hardcoded shared key and IV,
identical across all units of this model.

---

## Interpretation and caveats

The combination, an added UID-0 shell account, telnet forced on, a default
root password, and an undocumented listening service with command-execution
and provisioning heritage, is described here by effect, not intent. A vendor
could argue each element is legitimate manufacturing or auto-provisioning
tooling. **Whether deliberate or careless, the effect on the owner's security
is the same:** parties other than the owner can plausibly obtain control of a
stock unit on the same network.

Confidence: the findings above were read directly from the extracted
filesystem and binaries and are reproducible from the stock image. The
internal behaviour of the provisioning binary was characterised from strings
and imported symbols, not a full decompilation, so its complete capability
may be broader than described. The identification of the extra account as a
"backdoor" is a characterisation of effect, as noted.

---

## Recommendation

- Do not run this device on a trusted network in its stock firmware state.
  Treat a stock unit as potentially accessible to third parties.
- Reflash with OpenWrt (see the main README for the full procedure), which
  removes the telnet default, the extra account, and the provisioning binary.
- If any identical units exist, assume they share the same accounts, PSK, and
  mesh key, and treat them as a group.
- Change the Wi-Fi passphrase and any reused passwords on networks a stock
  unit has touched.

---

*No working credentials are published in this document. The specific hashes,
the cracked password, the hardcoded PSK, and the mesh key were recorded
privately during the review and are deliberately omitted here.*

---

## Methodology / how this was done

This port and review were done by a human operator working on the physical
device, using an AI assistant (Anthropic's Claude) as a **thinking partner
rather than the primary tool**. The operator set the goals, made every
decision, and did all the hands-on work; the assistant accelerated analysis,
suggested approaches, and helped draft documentation. Its stand-out
contribution was speed, deconstructing the obfuscated firmware faster than
the operator's own tooling, which is what made a deeper review practical. The
split of work was roughly:

**Human, on real hardware:**
- Opened the enclosure, identified components, and located the unlabelled
  UART header.
- Confirmed the serial pinout and the 3.3 V logic level with a multimeter.
- Attached a USB-TTL adapter and drove the serial console and U-Boot.
- Ran every command on the device, performed the flash, and verified the
  result on the live unit.

**AI-assisted:**
- Interpreting boot logs, chip markings, and the flash/partition layout.
- Reconstructing the flash backups from serial `md.b` hex dumps and
  verifying them against the device's own CRC32 values.
- Un-obfuscating and extracting the stock SquashFS (restoring the altered
  superblock), and reviewing the extracted filesystem for the findings above.
- Writing the OpenWrt device tree, image recipe, and sysupgrade changes, and
  drafting this documentation.

Every AI-suggested command was run by the human operator, and all results,
backups, checksums, and the final working device were verified on real
hardware. Nothing in this repository was taken on trust from the model
without being checked against the device.

The credential findings were cracked/confirmed locally during the review and
are deliberately omitted from this public document.


---

## Contact / about the author

This review and the OpenWrt port were done by **Jason Bunston**
(Elite Conceptual Consulting Ltd.).

- Website: https://eliteconceptual.com
- Email: jbunston@eliteconceptual.com
- LinkedIn: https://www.linkedin.com/in/jasonbunston/
- Book a 30-minute consulting call: https://calendar.app.google/6eEY9v4XAqnKiCJdA

For the full unredacted report (manufacturer or legitimate researchers),
please reach out by email.

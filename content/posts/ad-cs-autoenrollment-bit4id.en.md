---
title: "AD CS AutoEnrollment and CryptoAPI: a real-world case of enrollment blocked by cryptographic middleware"
date: 2026-06-25T14:00:00+02:00
draft: false
description: "A domain-joined Windows 11 client was failing to renew its machine certificate for EAP-TLS. The CA was working. The template was correct. The block was local, silent, and caused by a cryptographic middleware intercepting CryptoAPI before certreq could complete the request."
translationKey: "ad-cs-autoenrollment-bit4id"
tags:
  - PKI
  - AD CS
  - AutoEnrollment
  - Windows
  - CryptoAPI
  - EAP-TLS
  - 802.1X
  - Security
  - Troubleshooting
  - Enterprise Architecture
slug: "ad-cs-autoenrollment-bit4id"
cover:
  image: "img/ad-cs-autoenrollment-bit4id.png"
  alt: "ad-cs-autoenrollment-bit4id"
  relative: false
---

## The context

A fully patched, domain-joined Windows 11 client was failing to complete the renewal of a machine certificate used for EAP-TLS authentication on an 802.1X network. The certificate was issued against a dedicated AD CS template, using the Microsoft RSA SChannel provider, with Client Authentication EKU and `LocalMachine\My` as the target store.

The CA was issuing without issues. The template was correctly configured. Enroll and autoenroll permissions for computer objects were in place. And yet the client had no certificate.

This is the kind of problem that, in an enterprise environment with hundreds or thousands of clients, can take a long time to isolate — especially when operational pressure pushes toward aggressive intervention before the actual failure point has been identified.

## The problem

The symptom looked simple: `certreq -enroll -machine -q "Machine_AutoEnroll"` was not completing. No useful output, no explicit error, no certificate in the store.

Before looking at the client, all upstream components were verified: CA reachability, correct FQDN, template publication, enroll and autoenroll permissions, EKU, SAN, renewal period, chain, revocation, `LocalMachine\My` and `Request` store state, Microsoft RSA SChannel provider availability, cryptographic services, execution under the `NT AUTHORITY\SYSTEM` context. Nothing to report on any of those points.

The block was therefore local. To isolate it, a more granular approach was used: `certreq -new` with a minimal INF file, explicit provider declaration, stdout/stderr capture, and a process tree wrapper with a 120-second timeout.

The INF was kept deliberately sparse:

```ini
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=CLIENT-01"
MachineKeySet = TRUE
KeyLength = 2048
Exportable = FALSE
KeySpec = 1
KeyUsage = 0xa0
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
Silent = TRUE
```

No template in the INF: the template was to be passed only at submit time, to avoid involving the template-driven path during local key generation. The provider was declared explicitly. The test was reduced to the absolute minimum.

Result after 120 seconds: timeout. Empty stdout, empty stderr, no `.req` file created, no pending request in the `Request` store.

## The evidence that changed direction

The `certreq.exe` process tree at the moment of timeout showed this:

```
cmd.exe (wrapper)
└── certreq.exe
    └── kchain.exe [C:\Program Files (x86)\Bit4id\UKC\UKC\bin\kchain.exe]
        └── kchain_gui.exe [C:\Program Files (x86)\Bit4id\UKC\UKC\bin\kchain_gui.exe]
```

`certreq.exe` was waiting on `kchain.exe`, which had spawned `kchain_gui.exe`. A Bit4id UKC popup had also appeared during the test. The kill tree at timeout confirmed the chain: `kchain_gui.exe` child of `kchain.exe`, `kchain.exe` child of `certreq.exe`.

The CSP list, captured before removal, showed two Bit4id providers registered in the local CryptoAPI layer, both typed as `PROV_RSA_FULL` — the same type as the Microsoft provider declared in the INF:

```
Bit4id UKC Service Provider          - PROV_RSA_FULL
Bit4id Universal Middleware Provider - PROV_RSA_FULL
```

Alongside those, a Bit4id Key Storage Provider on the CNG layer and `bit4p11.dll` registered as a PKCS#11 provider were also present.

The installed software consisted of three components:

- Bit4id Universal MW 1.4.10.645
- Bit4id UKC 1.17.7.5
- Bit4id Firma4ng-InfoCamere 1.6.14

At this point, **Process Monitor** revealed that `certreq.exe` was attempting to write to a `C:\4log` path that did not exist on the system. I created the path to collect further evidence: once present, `bit4p11.dll` began writing binary logs to it. The file produced during the test was `bit4p11.dll.06_25-09_50_19.pid00002434.log.bin`, created at 11:50:19 — the exact second `certreq.exe` had started (StartTime=2026-06-25T11:50:17). The temporal correlation confirmed that `bit4p11.dll` was active during the operation.

The diagnosis was clear: the Bit4id middleware was intercepting the CryptoAPI path before `certreq.exe` could complete key generation, stalling while waiting for a token or user interaction via `kchain_gui.exe` — in a context (`NT AUTHORITY\SYSTEM`, silent mode) where that interaction could never happen.

## The decision

With an authorized change, the three Bit4id/UKC components were cleanly uninstalled, followed by a controlled reboot. Nothing else: no TPM reset, no domain rejoin, no manual deletion of MachineKeys or registry provider entries.

After the reboot, the CSP list showed only native Microsoft providers. No Bit4id process or service was running. Bit4id UKC and Universal Middleware registry keys were gone; a few residual entries under `HKLM\SOFTWARE\bit4id` related to logs and PKI manager components remained, with no functional impact.

## The result

With no further intervention, AutoEnrollment on the first cycle after reboot acquired the machine certificate automatically. The `CertificateServicesClient` log events were precise:

- **12:05:39** — Authentication to LDAP policy server completed
- **12:05:39** — Policy loaded from enrollment server
- **12:05:40** — Authentication to CA `CASERVER.ACME.LOCAL\CA` completed
- **12:05:40** — Certificate Machine_AutoEnroll received with Request ID 124077 from CA
- **12:05:41** — AutoEnrollment for Local System completed

Certificate present in `LocalMachine\My` at completion:

```
Template:    Machine_AutoEnroll
Issuer:      CN=CA, DC=ACME, LOCAL
NotBefore:   25/06/2026 11:55
NotAfter:    25/06/2027 11:55
Thumbprint:  A7F3C91E4B8D2065F0C9A13B7E42D58C6A90F1D3
Provider:    Microsoft RSA SChannel Cryptographic Provider
PrivateKey:  Non-exportable, bound, cryptographic test passed
```

`Request` store empty. No residual pending requests.

## What not to do

The pressure to quickly fix a client that fails network authentication tends to skip the analysis phase entirely. The actions that get applied first — usually without any evidence they are needed — are domain rejoin, TPM reset, and manual MachineKeys deletion.

A domain rejoin is not a neutral operation: it resets the secure channel, can trigger regeneration of all machine certificates, and has side effects on local configurations. In environments with BitLocker and TPM-bound keys, a TPM reset requires a dedicated maintenance window.

The principle is the inverse of instinct: first rule out everything outside the client (CA, template, permissions, network, chain), then isolate the problem locally with non-mutative tools, and only then apply the minimum necessary change. In this case, that minimum change was uninstalling three software packages.

## Trade-offs and risks of removal

Removing Bit4id from a corporate client is not neutral. In many environments the middleware is required for smart card access, CNS, digital signature tokens, or access to public administration services. Removal must be coordinated with whoever manages those needs, and in some cases the right answer is not to remove but to find a compatible version of the middleware, or reconfigure it to avoid interfering with the native CryptoAPI path.

In this specific case, removal was the appropriate decision because the middleware was not needed for the client's primary functions in that context. The assessment must be made case by case.

## Architectural implications

This case exposes something that is rarely considered in the design of enterprise environments running AD CS with AutoEnrollment: coexistence with third-party cryptographic middleware is not transparent.

Token and smart card middleware registers providers at the system level, sometimes on both CryptoAPI (CAPI) and CNG. This has global effects — not only on explicit PKI operations, but on any path that traverses the cryptographic layer, including AutoEnrollment, TLS, and Kerberos PKINIT. When a middleware registers as `PROV_RSA_FULL`, it can be invoked by any cryptographic operation of that type, regardless of how explicitly the provider is declared in the INF or in the request.

The questions an architect should ask before rollout — and that belong in any assessment:

- Which third-party cryptographic middleware is present across the fleet? Is there a standardized, defined version?
- Has AutoEnrollment behavior been tested in their presence, under the `NT AUTHORITY\SYSTEM` context and in silent mode?
- Are AutoEnrollment GPOs monitored for enrollment failure events, or does the problem only surface when a user can no longer authenticate?
- Is there a coordinated change process between whoever manages PKI and whoever manages clients?

Monitoring `CertificateServicesClient` events with alerts on enrollment failures is a straightforward measure that is not active in most environments. Setting it up costs little and significantly reduces detection time for problems like this one.

## Lessons learned

The problem was not in the PKI. Not in the template. Not in the network. It was in software installed on the client that was silently intercepting a system mechanism, in a context where it could not complete its own operation.

The lesson is not specific to Bit4id: the evidence collected reflects a particular combination of versions, local configuration, and operating system, without comparative testing on alternative versions. This is not a product advisory.

The lesson is that third-party cryptographic middleware has far broader visibility and scope than those who install it typically consider — and in an enterprise PKI architecture, that cannot be taken for granted. It needs to be mapped, tested, and monitored.

---

*Environment: Windows 11 25H2 (patches current as of 25/06/2026). Enterprise AD CS CA. Template Machine_AutoEnroll for EAP-TLS/802.1X.*
*Bit4id components identified: Universal MW 1.4.10.645, UKC 1.17.7.5, Firma4ng-InfoCamere 1.6.14.*
*The case has been anonymized with generic values. No comparative testing was performed with alternative versions of the middleware.*

---

**Update, June 30, 2026:** further analysis has confirmed that updating the Bit4id middleware resolves the issue described above. Anyone running this middleware in production should plan to update it.
# License and Certificate Audit

This document maps the Sandboxie-Plus license, supporter certificate, serial number, and activation-related code paths for compliance review and maintenance. It is descriptive only and does not document bypass or removal steps.

## Summary

Sandboxie-Plus uses a signed supporter certificate model rather than a single centralized `license` module name. The main persisted artifact is `Certificate.dat`. The kernel driver validates that file, exposes the resulting certificate state through driver APIs, and higher-level components use that state to gate supporter features.

The repository also contains many open-source license headers and third-party license texts. Those are separate from the product supporter certificate mechanism described here.

## Primary Files

| Area | File | Purpose |
| --- | --- | --- |
| Driver certificate validation | `Sandboxie/core/drv/verify.c` | Reads `Certificate.dat`, parses fields, verifies signature, computes certificate state and feature options. |
| Driver certificate state | `Sandboxie/core/drv/verify.h` | Defines `SCertInfo`, certificate types, levels, and helper macros. |
| Plus certificate UI | `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp` | Certificate settings page, serial entry, evaluation certificate request, certificate apply flow. |
| Plus certificate API integration | `SandboxiePlus/SandMan/OnlineUpdater.cpp` | Builds certificate server requests and downloads supporter certificates. |
| Plus certificate storage | `SandboxiePlus/SandMan/SandMan.cpp` | Writes `Certificate.dat` through the API and checks feature access in UI flows. |
| Classic certificate apply flow | `Sandboxie/apps/control/AboutDialog.cpp` | Reads clipboard, detects serial numbers, fetches certificates, writes `Certificate.dat`, reloads config. |
| CLI/update certificate tooling | `SandboxieTools/UpdUtil/UpdUtil.cpp` | Supports `get_cert` / `get_cert_lr`, certificate downloads, and update utility flows. |
| Localized messages | `Sandboxie/msgs/Sbie-English-1033.txt` and translated `Text-*` files | License Manager, Product Key, Activation Key, and certificate user-facing strings. |

## Certificate Format and Storage

The certificate file is named `Certificate.dat`.

Relevant paths and functions:

| Reference | Role |
| --- | --- |
| `Sandboxie/core/drv/verify.c:540` | Driver constructs the `(HomePath)\\Certificate.dat` path. |
| `Sandboxie/core/drv/verify.c:574` | Driver reads the certificate file from the home path. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3179` | Plus UI loads the certificate for display. |
| `SandboxiePlus/SandMan/SandMan.cpp:3461` | Plus writes certificate bytes to `Certificate.dat`. |
| `Sandboxie/apps/control/AboutDialog.cpp:539` | Classic apply flow targets the home `Certificate.dat`. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1799` | CLI tool defaults certificate output to `base_dir\\Certificate.dat`. |

Common parsed fields include:

| Field | Reference | Notes |
| --- | --- | --- |
| `DATE` | `Sandboxie/core/drv/verify.c:687` | Certificate issue/check date and optional validity days. |
| `DAYS` | `Sandboxie/core/drv/verify.c:706` | Optional explicit validity duration. |
| `TYPE` | `Sandboxie/core/drv/verify.c:713` | Certificate type, optionally with `TYPE-LEVEL`. |
| `LEVEL` | `Sandboxie/core/drv/verify.c:726` | Feature level when separate from `TYPE`. |
| `OPTIONS` | `Sandboxie/core/drv/verify.c:733` | Explicit feature options such as `SBOX`, `EBOX`, `NETI`, `DESK`, `NoCR`. |
| `UPDATEKEY` | `Sandboxie/core/drv/verify.c:740` | Update key used for renewals, upgrades, and blocklist checks. |
| `AMOUNT` | `Sandboxie/core/drv/verify.c:747` | Seat or amount metadata. |
| `SOFTWARE` | `Sandboxie/core/drv/verify.c:750` | If present, must match `Sandboxie-Plus`. |
| `HWID` | `Sandboxie/core/drv/verify.c:756` | Optional node-locking to the current hardware ID. |
| `SIGNATURE` | `Sandboxie/core/drv/verify.c:660` | Base64 signature over hashed certificate fields. |

## Signature Verification

The certificate validation path is in `KphValidateCertificate()`.

Key references:

| Reference | Role |
| --- | --- |
| `Sandboxie/core/drv/verify.c:41` | Trusted ECDSA P-256 public key blob. |
| `Sandboxie/core/drv/verify.c:660` | Extracts and decodes the `SIGNATURE` field. |
| `Sandboxie/core/drv/verify.c:675` | Hashes certificate tag names. |
| `Sandboxie/core/drv/verify.c:678` | Hashes certificate values. |
| `Sandboxie/core/drv/verify.c:767` | Finalizes the hash. |
| `Sandboxie/core/drv/verify.c:775` | Verifies hash/signature with `KphVerifySignature`. |
| `Sandboxie/core/drv/verify.c:777` | Checks hard-coded blocked `UPDATEKEY` values. |
| `Sandboxie/core/drv/verify.c:792` | Checks secure `CertBlockList` entries. |
| `Sandboxie/core/drv/verify.c:826` | Enforces node-locking when `HWID` fields are present. |

Related generic verification helpers:

| Reference | Role |
| --- | --- |
| `Sandboxie/core/drv/verify.c:289` | `KphVerifyBuffer()` verifies external buffers and signatures. |
| `Sandboxie/core/drv/verify.c:390` | `KphVerifyCurrentProcess()` verifies the current process image. |
| `Sandboxie/core/drv/verify.h:103` | Public declaration for `KphVerifyBuffer()`. |
| `Sandboxie/core/drv/verify.h:104` | Public declaration for `KphVerifyCurrentProcess()`. |

## Certificate State

Certificate state is represented by `SCertInfo` in `Sandboxie/core/drv/verify.h:20`.

Important fields:

| Field | Meaning |
| --- | --- |
| `active` | Certificate is valid and usable for feature decisions. |
| `expired` | Certificate is expired. It may still be active during grace handling. |
| `outdated` | Certificate is not valid for the current build date. |
| `grace_period` | Certificate is expired or outdated but temporarily accepted. |
| `locked` | Certificate has matched a hardware lock. |
| `lock_req` | Refresh or locking is required. |
| `type` | Encoded certificate type. |
| `level` | Encoded feature level. |
| `opt_sec` | Enables security/privacy enhanced and app box features. |
| `opt_enc` | Enables encrypted/protected sandbox features. |
| `opt_net` | Enables network interception features. |
| `opt_desk` | Enables isolated Sandboxie desktop features. |
| `expirers_in_sec` | Seconds until expiration, or negative if expired. |

Certificate types are defined in `Sandboxie/core/drv/verify.h:51`:

| Type | Notes |
| --- | --- |
| `eCertEternal` | Eternal certificate class. |
| `eCertContributor` | Contributor certificate. |
| `eCertBusiness` | Business certificate class. |
| `eCertPersonal` | Personal/supporter certificate class. |
| `eCertHome` | Home/subscription certificate class. |
| `eCertFamily` | Family certificate class. |
| `eCertDeveloper` | Developer certificate. |
| `eCertPatreon` | Patreon certificate class. |
| `eCertGreatPatreon` | Higher Patreon tier. |
| `eCertEntryPatreon` | Entry Patreon tier. |
| `eCertEvaluation` | Evaluation certificate. |

Feature levels are defined in `Sandboxie/core/drv/verify.h:87`:

| Level | Notes |
| --- | --- |
| `eCertStandard` | Standard level. |
| `eCertStandard2` | Alternate standard level. |
| `eCertAdvanced1` | Intermediate advanced level. |
| `eCertAdvanced` | Advanced level. |
| `eCertMaxLevel` | Maximum level. |

Subscription and insider helpers:

| Reference | Role |
| --- | --- |
| `Sandboxie/core/drv/verify.h:96` | `CERT_IS_TYPE`. |
| `Sandboxie/core/drv/verify.h:97` | `CERT_IS_SUBSCRIPTION`. |
| `Sandboxie/core/drv/verify.h:98` | `CERT_IS_INSIDER`. |

## Validation Rules

The following rules are applied in `KphValidateCertificate()`:

| Reference | Rule |
| --- | --- |
| `Sandboxie/core/drv/verify.c:889` | `TYPE` is mandatory for normal classification. |
| `Sandboxie/core/drv/verify.c:891` | Maps textual certificate types to `SCertInfo.type`. |
| `Sandboxie/core/drv/verify.c:925` | Eternal/developer/evaluation classes receive maximum level handling. |
| `Sandboxie/core/drv/verify.c:936` | Default or missing level maps to standard. |
| `Sandboxie/core/drv/verify.c:982` | Explicit `OPTIONS` override default option derivation. |
| `Sandboxie/core/drv/verify.c:1015` | Without explicit `OPTIONS`, options are derived from certificate level. |
| `Sandboxie/core/drv/verify.c:1031` | Eternal certificates do not expire. |
| `Sandboxie/core/drv/verify.c:1038` | Detects subscription certificate types. |
| `Sandboxie/core/drv/verify.c:1045` | Marks expired certificates. |
| `Sandboxie/core/drv/verify.c:1050` | Non-subscription certificates can become outdated for newer builds. |
| `Sandboxie/core/drv/verify.c:1056` | Inactive status is applied when expiration/outdated rules fail outside grace period. |
| `Sandboxie/core/drv/verify.c:1069` | Checks secure `RequireLock` parameter. |
| `Sandboxie/core/drv/verify.c:1076` | Enforces lock requirement for Simplified Chinese install UI language. |
| `Sandboxie/core/drv/verify.c:1080` | Applies refresh/lock rules unless `NoCR` applies or certificate type is exempt. |

## Online Certificate Retrieval

Certificate retrieval uses `https://sandboxie-plus.com/get_cert.php`.

Plus UI and updater path:

| Reference | Role |
| --- | --- |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:274` | `COnlineUpdater::GetSupportCert()` entry point. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:281` | Adds `SN` when a serial is provided. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:287` | Adds `UpdateKey` for renewals/upgrades. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:290` | Builds `HashKey` from random ID and user settings name. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:295` | Evaluation certificate requests use `Name` and `eMail`. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:301` | Adds `LR=1` when lock is required. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:306` | Adds `HwId` when needed. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:316` | Builds the `get_cert.php` URL. |

Plus settings UI path:

| Reference | Role |
| --- | --- |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3311` | `OnGetCert()` handles the serial entry button. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3322` | Validates that serials start with `SBIE`. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3349` | Extracts existing `UPDATEKEY` when available. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3353` | Calls `GetSupportCert()`. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3360` | Evaluation certificate flow entry. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3388` | Receives certificate data or error. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3407` | Applies certificate. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3514` | Static `ApplyCertificate()` implementation. |

Classic UI path:

| Reference | Role |
| --- | --- |
| `Sandboxie/apps/control/AboutDialog.cpp:191` | Detects serial text by `SBIE` prefix. |
| `Sandboxie/apps/control/AboutDialog.cpp:203` | Retrieves hardware ID from driver info class `-2`. |
| `Sandboxie/apps/control/AboutDialog.cpp:215` | Reads `UPDATEKEY` from an existing certificate. |
| `Sandboxie/apps/control/AboutDialog.cpp:248` | Downloads certificate data via WinHTTP. |
| `Sandboxie/apps/control/AboutDialog.cpp:365` | Builds certificate server query from a serial. |
| `Sandboxie/apps/control/AboutDialog.cpp:370` | Adds `HwId` for node-locked serials. |
| `Sandboxie/apps/control/AboutDialog.cpp:378` | Adds `UpdateKey` for renewal/upgrade serials. |
| `Sandboxie/apps/control/AboutDialog.cpp:391` | Adds `HashKey`. |
| `Sandboxie/apps/control/AboutDialog.cpp:410` | Calls `sandboxie-plus.com/get_cert.php`. |
| `Sandboxie/apps/control/AboutDialog.cpp:459` | `ApplyCertificate()` entry point. |
| `Sandboxie/apps/control/AboutDialog.cpp:552` | Reloads certificate config after writing. |

CLI/update utility path:

| Reference | Role |
| --- | --- |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1795` | Handles `get_cert` and `get_cert_lr` commands. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1818` | Builds `SN` query for `get_cert`. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1822` | Adds `HwId` when required. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1834` | Reads or accepts `update_key`. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1851` | Downloads certificate from update domain. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1876` | Prints certificate when no output path is selected. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1878` | Writes `Certificate.dat` when using the default file path. |

## Certificate Reload and Driver Exposure

Certificate state is consumed by other components through driver APIs.

| Reference | Role |
| --- | --- |
| `SandboxiePlus/QSbieAPI/SbieAPI.cpp:2176` | Reloads certificate via `SBIE_CONF_FLAG_RELOAD_CERT`. |
| `SandboxiePlus/QSbieAPI/SbieAPI.cpp:2184` | Calls `API_RELOAD_CONF`. |
| `Sandboxie/core/svc/DriverAssistStart.cpp:423` | Service-side certificate reload. |
| `Sandboxie/core/svc/DriverAssistStart.cpp:437` | Queries driver info class `-1` into `SCertInfo`. |
| `Sandboxie/core/svc/UserServer.cpp:649` | Queries `SCertInfo` to authorize encrypted feature behavior. |
| `Sandboxie/core/svc/MountManager.cpp:1128` | Queries `SCertInfo` to authorize file image and RAM disk behavior. |

## Feature Gates

The product uses `SCertInfo` fields to gate selected features at the UI, service, DLL, and driver layers.

UI gates:

| Reference | Gate |
| --- | --- |
| `SandboxiePlus/SandMan/SandMan.cpp:3469` | `CSandMan::CheckCertificate()` central UI check. |
| `SandboxiePlus/SandMan/SandMan.cpp:3474` | Checks `opt_enc` or `opt_net` for advanced features. |
| `SandboxiePlus/SandMan/SandMan.cpp:3487` | Checks `active` or `opt_sec` for supporter-only feature sets. |
| `SandboxiePlus/SandMan/Windows/OptionsGeneral.cpp:95` | Adds supporter badge/check for security feature options. |
| `SandboxiePlus/SandMan/Windows/OptionsGeneral.cpp:102` | Adds supporter badge/check for encryption feature options. |
| `SandboxiePlus/SandMan/Windows/OptionsNetwork.cpp:74` | Checks network feature availability. |

Service and DLL gates:

| Reference | Gate |
| --- | --- |
| `Sandboxie/core/svc/UserServer.cpp:649` | Requires `active && opt_enc`. |
| `Sandboxie/core/svc/MountManager.cpp:1128` | Requires `active && opt_enc` or `active && opt_sec`, depending on feature. |
| `Sandboxie/core/dll/net.c:2116` | Requires `active && opt_net` for network proxy/interception behavior. |
| `Sandboxie/core/dll/dns_filter.c:577` | Requires `active && opt_net` for DNS filtering. |

Driver gates:

| Reference | Gate |
| --- | --- |
| `Sandboxie/core/drv/process.c:783` | Checks `active && opt_sec` for security-related process handling. |
| `Sandboxie/core/drv/process.c:814` | Checks `active && opt_enc` for encryption/protection-related process handling. |
| `Sandboxie/core/drv/api.c:1216` | Includes `opt_sec` in API behavior. |
| `Sandboxie/core/drv/api.c:1222` | Includes `opt_enc` in API behavior. |
| `Sandboxie/core/drv/api.c:1225` | Includes `opt_net` in API behavior. |

## User-Facing License and Activation Text

The primary English message file contains License Manager and activation strings:

| Reference | Text Area |
| --- | --- |
| `Sandboxie/msgs/Sbie-English-1033.txt:3912` | License Manager section begins. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3930` | Registered and license-active messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3934` | Registered but reactivation-needed messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3939` | Unregistered copy messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3951` | Product key expiration messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3965` | License Manager network access messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3969` | Registration key not recognized messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:3996` | License activation instructions. |
| `Sandboxie/msgs/Sbie-English-1033.txt:4040` | Invalid product key messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:4048` | Invalid activation key messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:4056` | Product activation failure messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:4077` | Product key not recognized by server messaging. |
| `Sandboxie/msgs/Sbie-English-1033.txt:4085` | Product key activated on too many computers messaging. |

Translated equivalents appear in `Sandboxie/msgs/Text-*` and `Sandboxie/msgs/report/Report-*` files.

## Evaluation Certificate Flow

Evaluation certificate UI limits are documented as UI-side limits only.

| Reference | Role |
| --- | --- |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.h:252` | `EVAL_MAX` is `3`, noted as UI-only because actual limits are server enforced. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.h:253` | `EVAL_DAYS` is `10`. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3300` | Displays evaluation certificate availability. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3365` | `StartEval()` collects name and email. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3381` | Requests an evaluation certificate through `GetSupportCert()`. |
| `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp:3427` | Increments UI-side evaluation count after a successful evaluation certificate apply. |

## Related Installer and Update Code

These files are related to installation/update flow but are not the core certificate verifier:

| Reference | Role |
| --- | --- |
| `Installer/Sandboxie-Plus.iss` | Plus Inno Setup installer. Includes `/PORTABLE=1`, service/driver install, update utility invocation, and optional UI launch. |
| `Sandboxie/install/SandboxieVS.nsi` | Classic NSIS installer. Includes install/upgrade/remove modes and bundled install behavior. |
| `SandboxiePlus/SandMan/OnlineUpdater.cpp:1176` | Runs update installers, optionally silently. |
| `SandboxieTools/UpdUtil/UpdUtil.cpp:1402` | Runs setup with `/open_agent`, optionally adding `/SILENT` for embedded update runs. |

## Audit Notes

- The actual certificate trust decision is made in the driver validation path, not only in the UI.
- `Certificate.dat` is text-like and parsed by tag name/value pairs, but authenticity depends on the signature verified against the embedded public key.
- UI-side checks are not the only enforcement points; service, DLL, and driver code also query `SCertInfo` or inspect feature option bits.
- Online certificate retrieval is centralized around `get_cert.php` and uses serial numbers, update keys, hardware IDs, and a hash key depending on request type.
- `EVAL_MAX` is explicitly marked as UI-only; the authoritative evaluation limit is server-side.

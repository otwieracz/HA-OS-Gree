# Security Review: HA-OS-Gree Custom Integration

**Date:** 2026-05-11
**Reviewer:** Senior security engineer (automated review)
**Scope:** Entire fork snapshot at this commit. Non-test sources under `custom_components/gree/` reviewed: `__init__.py`, `bridge.py`, `climate.py`, `config_flow.py`, `const.py`, `entity.py`, `switch.py`, `manifest.json`. Test files (`test_*.py`, `conftest.py`, snapshots) excluded.

**Result: No high-confidence (>=8) security vulnerabilities found.**

## Why

This integration is a thin Home Assistant wrapper around the pinned upstream `greeclimate==1.4.1` library:

- **No code-execution sinks** in the changeset: no `eval`/`exec`/`pickle`/`yaml.load`/`subprocess`/`os.system`, no `open()` on user-controlled paths, no SQL, no HTML/template rendering. Deserialization is delegated to HA core / upstream lib.
- **Sole user input** is the config-flow `ip` field, validated via `IPv4Address(ip.strip())` in `config_flow.py:34` and `__init__.py:37`; non-IPv4 raises `ValueError` before reaching any sink. The string never reaches the filesystem, shell, or a request URL.
- **Entity command paths** in `climate.py` / `switch.py` allowlist every enum-like argument (`HVAC_MODES_REVERSE`, `PRESET_MODES`, `FAN_MODES_REVERSE`, `SWING_MODES`) and `round()` temperatures before delegating to the device.
- **No crypto** in this codebase. Gree's AES handshake lives entirely in upstream `greeclimate` (out of scope: third-party library risk is managed separately).
- **Manifest** pins a specific upstream version and declares `iot_class: local_polling` (LAN only); no remote endpoints, no embedded secrets.

## Considered and explicitly ruled out

- `bridge.py` `device_update` accepts an IP change for a matching MAC from a UDP discovery reply. A LAN-foothold attacker could spoof a discovery response to redirect traffic for a known MAC, but binding/encryption is re-negotiated by upstream on each IP, exploitation requires pre-existing LAN access, and this is inherent to the `local_polling` design. Confidence below threshold.
- `config_flow.py:59` broad `except Exception` surfaces a generic "unknown" UI error. Not a security issue.
- `common.py` is test scaffolding (excluded).

## Methodology notes

- Categories examined: input validation (SQLi, command injection, XXE, template injection, path traversal), authn/authz, crypto/secrets, deserialization & code execution, data exposure.
- Exclusions applied: DoS, on-disk secrets, rate limiting, lack of hardening, outdated deps, regex injection/ReDoS, log spoofing, documentation issues, path-only SSRF.
- Findings threshold: confidence >= 8/10 with a concrete exploit path.

# Credentials Verification

```yaml
RequestID: REQ-20260331-223247
VerifiedAt: 2026-03-31T22:33:00
```

## Target Repository: statsclaw/example-probit

| Check | Result |
|-------|--------|
| Auth method | gh CLI (keyring token) |
| Push access | **PASS** (admin: true, push: true) |
| Verification method | `gh api repos/statsclaw/example-probit --jq '.permissions'` |

## Workspace Repository: TianzhuQin/workspace

| Check | Result |
|-------|--------|
| Auth method | gh CLI (keyring token) |
| Push access | **PASS** (admin: true, push: true) |
| Verification method | `gh api repos/TianzhuQin/workspace --jq '.permissions'` |

## Overall: PASS

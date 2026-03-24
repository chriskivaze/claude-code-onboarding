# Flutter Security Hardening & Privacy Compliance

For general code security (OWASP Top 10, injection, auth, secrets) use the `security-reviewer` agent instead. This reference focuses on **Flutter-specific** security and **mobile privacy compliance**.

## Flutter Client-Side Security

### Secure Storage

- Use `flutter_secure_storage` for sensitive data — never `SharedPreferences` for secrets
- No hardcoded API keys, secrets, or credentials in Dart code
- Obfuscate release builds (`--obfuscate --split-debug-info`)

### Network Security

- Certificate pinning for API connections
- No HTTP traffic in production (enforce HTTPS)
- API keys served via backend proxy, not embedded in app

### Build Security

- Obfuscate release builds: `flutter build apk --obfuscate --split-debug-info=build/symbols`
- ProGuard/R8 enabled for Android
- Strip debug symbols in release

### Deep Link Security

- Validate deep link parameters before processing
- Don't expose sensitive routes via deep links
- Use App Links (Android) / Universal Links (iOS) over custom schemes

## Privacy Compliance

### GDPR Requirements

- **Lawful Basis** — document the legal basis for each data type
- **Data Minimization** — collect only what's necessary
- **Consent Management** — granular consent with easy withdrawal
- **Data Subject Rights** — implement export, deletion, rectification, portability
- **Data Retention** — automated cleanup policies; never store data longer than needed

### CCPA Requirements

- **Consumer Rights** — right to know, delete, and opt out of data sales
- **Do Not Sell** — respect opt-out preferences
- **Privacy Preferences** — per-user opt-out tracking with audit history

## Pre-Release Security Checklist

- [ ] Zero hardcoded secrets in codebase
- [ ] `flutter_secure_storage` used for all sensitive data
- [ ] Release builds obfuscated
- [ ] Certificate pinning configured
- [ ] GDPR consent flow implemented
- [ ] Data export/deletion endpoints functional
- [ ] Privacy policy URL accessible from app
- [ ] Dependencies scanned for known vulnerabilities

## Incident Response (Mobile-Specific)

1. **Contain** — force app update or feature flag to disable compromised flow
2. **Investigate** — collect crash logs, analyze scope of data exposure
3. **Notify** — user notification + regulatory notification within required timeframes (72h GDPR)
4. **Remediate** — push hotfix, revoke compromised tokens, rotate keys
5. **Review** — post-incident report, update security policies

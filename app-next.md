# Desktop App — Next Release Items (beta.13+)

## beta.13
- [ ] Mac build — Apple signing wired, needs App Store Connect notarization key from Philip
- [ ] Mac CI matrix in build.yml (Hermes to add)

## UI / UX
- [ ] (add items here)

## Backend / API
- [ ] (add items here)


## Onboarding / Import
- [ ] Bulk history importer — Pro Local feature, file-based import for history you can't re-open in browser
      Platforms: ChatGPT (conversations.json), Claude (markdown export), Copilot, Gemini, Perplexity, Grok
      Extension already backfills open threads; this covers years of archived history
      UI: file drop zone, detect format automatically, parse + ingest to local DB
      Surface in onboarding AND in Settings as a persistent option
      REQUIRED for beta.13 — Philip's decision 2026-06-13

## macOS
- [ ] App Store Connect API key — generate at appstoreconnect.apple.com → Keys → New API key (Developer role)
- [ ] Add macos-latest to CI matrix in build.yml (Hermes)
- [ ] Base64 cert + add GH Actions secrets: MAC_CERT_P12_BASE64, MAC_CERT_PASSPHRASE, ASC_API_KEY, ASC_API_KEY_ID, ASC_API_ISSUER_ID
- [ ] Safari extension — convert Chrome MV3 using safari-web-extension-converter, wrap in Mac app, submit to Mac App Store


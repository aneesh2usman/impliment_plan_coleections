# Production-Grade Implementation Plan: Generic Abbreviation Preservation & Transliteration Architecture

Provide an enterprise-ready, production-grade solution in `translation_management` to prevent external translation engines (such as Google Translate) from guessing or expanding acronyms/abbreviations into full phrases. Supports explicit company-scoped glossaries, phonetic transliteration (e.g., `KAI` -> `كاي`), and zero-database-HTML plain-text token shielding (`__RAWADH_PROT_n__`).

## Production-Grade Architecture & Resolution Chain

```
Input Text (Plain Text)
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 1: Explicit Glossary Lookup                        │
│ (Custom company/system TranslationMemory overrides)     │
└──────────────────────────┬──────────────────────────────┘
                           │ (If no explicit match)
                           ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 2: Acronym & Abbreviation Detection Engine          │
│ (Regex \b[A-Z0-9]{2,6}\b with stop-word exclusions)    │
└──────────────────────────┬──────────────────────────────┘
                           │ (Detected acronym tokens)
                           ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 3: Token Placeholder Shielding                     │
│ Replace tokens: "Project KAI" -> "Project __RAWADH_0__" │
└──────────────────────────┬──────────────────────────────┘
                           │ (Send shielded plain text)
                           ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 4: External Translation Provider (Google Translate)│
│ Translates surrounding text in format='text' mode       │
└──────────────────────────┬──────────────────────────────┘
                           │ (Received translated text)
                           ▼
┌─────────────────────────────────────────────────────────┐
│ Tier 5: Post-Translation Unshielding & Transliteration  │
│ - Mode 1: Preserve original (Latin)                      │
│ - Mode 2: Phonetic Transliteration (e.g., KAI -> كاي)    │
│ - Mode 3: Apply explicit glossary mapping               │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
Resulting Clean Plain Text (Persisted in DB / Returned to API)
```

---

## Technical Specifications

### 1. Robust Acronym Protection Engine (`AcronymProtectionEngine`)
- **Location:** [acronym_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/acronym_service.py)
- **Features:**
  - Thread-safe, LRU-cached regex tokenization (`\b[A-Z0-9]{2,6}\b`).
  - Exclusion list for standard uppercase English words (e.g., `IN`, `AT`, `IS`, `ON`, `IT`, `OR`, `NO`, `AND`, `THE`, `FOR`, `WITH`).
  - Standardized token format: `__RAWADH_PROT_0__`, `__RAWADH_PROT_1__` (untranslated by NMT models).
  - High-precision Arabic phonetic transliterator mapping (English consonants/vowels -> Arabic characters: `KAI` -> `كاي`, `NASA` -> `ناسا`, `HVAC` -> `اتش_في_اي_سي` / `اتش_فاك`).
  - Tolerant fuzzy-unshielding recovery (handles potential whitespace additions around placeholders by external translation providers).

### 2. Integration Points
- **Hot-Path Request Lookup:** `RequestTranslationLookupService` in [translation_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/translation_service.py).
- **Offline Bulk Refresh:** `BatchTranslationService` in [batch_translation_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/batch_translation_service.py).
- **Background Queue Workers:** `async_translate_missing_keys_task` in [tasks.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/tasks.py).
- **Central Registry:** `TRANSLATION_ACRONYM_CONFIG` & `ACRONYM_STOP_WORDS` in [registry.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/registry.py).

---

## Proposed Changes

### Core Implementation Files

#### [NEW] [acronym_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/acronym_service.py)
- Implement `AcronymProtectionEngine`:
  - `shield_text(text, target_lang)` -> `(shielded_text, token_map)`
  - `unshield_text(shielded_text, token_map, target_lang, mode='transliterate')` -> `unshielded_text`
  - `transliterate_acronym_to_arabic(acronym)` -> `arabic_transliterated_str`
  - `get_explicit_glossary_override(acronym, company_id, target_lang)` -> `translated_str or None`

#### [MODIFY] [registry.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/registry.py)
- Add global acronym configuration, stop-word exclusions, and phonetic transliteration mapping rules.

#### [MODIFY] [translation_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/translation_service.py)
- Update lookup service to shield acronyms on missing keys before requesting translations.

#### [MODIFY] [batch_translation_service.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/services/batch_translation_service.py)
- Update bulk processing to apply acronym shielding and unshielding across batch operations.

#### [MODIFY] [tasks.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/tasks.py)
- Update background Celery task to apply acronym protection during async translation runs.

#### [MODIFY] [tests.py](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/tests.py)
- Add comprehensive production unit tests verifying:
  - Standalone acronym transliteration (`KAI` -> `كاي`).
  - Sentence token shielding (`Project KAI camera` -> `مشروع كاي كاميرا` / `مشروع KAI كاميرا`).
  - Special character safety (`&`, `<`, `>`, URLs, line breaks).
  - Explicit glossary overrides taking priority.

---

### Mandatory Documentation & Practice Materials

#### [NEW] [acronym_and_glossary_handling.md](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/translation_management/docs/acronym_and_glossary_handling.md)
- Production architecture documentation covering resolution chain, transliteration rules, and plain-text safety guarantees.

#### [NEW] [abbreviation_translation_practice.ipynb](file:///home/aneesh/Documents/python/docker/rawadhv2/rawadhai_backend/practice/abbreviation_translation_practice.ipynb)
- Interactive Jupyter practice notebook demonstrating production acronym shielding, Arabic transliteration (`KAI` -> `كاي`), API request examples, reusable test cases, edge cases, and verification steps.

## Verification Plan

### Automated Tests
- Run full test suite inside Docker:
  ```bash
  docker compose exec rawadh_backend python manage.py test translation_management
  ```

### Manual Verification
- Execute and validate test scenarios in `practice/abbreviation_translation_practice.ipynb`.

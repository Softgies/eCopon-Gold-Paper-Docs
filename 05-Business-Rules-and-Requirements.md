# 05 — Business Rules and Requirements

> Requirements are reproduced with original terminology. Conflicting statements are listed explicitly
> and **not** silently reconciled. Financial values are **not** corrected or chosen between. Values
> that could not be found in any inspected source are marked **UNVERIFIED** rather than asserted.

## 1. Evidence boundary (read this first)

- All **Gold Paper** business rules found in source come **only** from the standalone
  `E:\eCopon-Code\Gold-Paper-POC` repository, and every one is labelled there as a **POC test
  assumption / not an approved production business rule**.
- The eCopon production docs (`ecopon-2025/docs/*`, `ecopon-new-dashboard-latest/**`) contain **no**
  Gold Paper, "Golden Paper", "SMPoints", or halala requirement.
- The task brief references "500 SMPoints = 10 SAR paper", "0.020 halala per point", and "conflicting
  limits on papers and competition chances". The phrases **"10 SAR per paper"**, **"0.020 halala"**,
  **"SMPoints"**, and any **max papers / max chances** limit were **not located** in any inspected
  repository or document. They are recorded below as **UNVERIFIED — stated in the requirements brief,
  not found in inspected source** so they are not lost, but they must not be treated as sourced facts.

## 2. Requirements table

| # | Requirement | Verbatim quote (where sourced) | Source (file:line) | Status | Conflict / ambiguity | Decision owner / clarification |
| --- | --- | --- | --- | --- | --- | --- |
| R1 | POC is not production | "This repository is not part of the eCopon production applications." | `Gold-Paper-POC/README.md:3` | POC scaffold | — | — |
| R2 | Points-per-paper rate = 500 (test) | "`MockPointsPerPaper` defaults to 500. That rate is a POC test setting, not an approved business rule." | `Gold-Paper-POC/docs/conversion.md:5` | **Not approved** | vs. withdrawal rate R12; vs. UNVERIFIED R15 | Business owner must approve a rate |
| R3 | Rate flag persisted unapproved | "Every stored conversion sets `RateIsApprovedBusinessRule` to 0." | `Gold-Paper-POC/docs/conversion.md:5`; DDL `sql/schema/01-create-tables.sql:61` | **Not approved** | — | — |
| R4 | Mock rate is technical, not business | "`MockPointsPerPaper` ... is not an approved business conversion rate." | `Gold-Paper-POC/README.md:21` | **Not approved** | — | — |
| R5 | Conversion math (whole papers only) | "Whole papers are requested / rate. The debit is whole papers × rate. A remainder stays in the balance." | `Gold-Paper-POC/docs/conversion.md:7` | Proposed behavior | — | — |
| R6 | No partial paper | "A remainder smaller than one paper does not create a partial paper." | `Gold-Paper-POC/docs/acceptance-tests.md:7`; DDL `01-create-tables.sql:67` | Proposed | — | — |
| R7 | Rejection precedence | "`InsufficientBalance` wins over `NoWholePaper`. Both rejections insert one conversion row and one `ConversionRejected` event..." | `Gold-Paper-POC/docs/conversion.md:7` | Proposed | — | — |
| R8 | Successful conversion issues papers + event atomically | "A completed conversion updates the balance and inserts one Available paper per issued paper and one ConversionCompleted event in the same transaction." | `Gold-Paper-POC/docs/conversion.md:7` | Proposed | — | — |
| R9 | Paper source is conversion-only | `CK_PocPaper_Source CHECK (Source = 'Conversion')` | `Gold-Paper-POC/sql/schema/01-create-tables.sql:96` | POC schema | No "purchased" paper source exists | Clarify purchased vs. earned (see R16) |
| R10 | Idempotency / concurrency | "Repeating the same idempotency key returns the first result and does not debit again." / "Two concurrent conversions ... cannot make the balance negative." | `Gold-Paper-POC/docs/acceptance-tests.md:9-10` | Proposed | — | — |
| R11 | Earning / debiting / issuing not authorized | "Not included, and not authorized yet: ... Reading or debiting points. Issuing papers." | `Gold-Paper-POC/README.md:13-18` | **Not authorized** | — | Pending authorization |
| R12 | Loyalty withdrawal rate (production, different feature) | "100 points = 1 SAR ... 1000 / 100 = 10 SAR" (comment) | `E:\Download-21-09-2026\eWallet\core_usp_Loyalty_GetTotalPointsByUserId_V2.sql:134-137` (workspace candidate) | Existing (withdrawal) | **Different feature**; not a paper rate | — |
| R13 | Withdrawal rate is admin-configured | field `PointsPerCurrencyUnit` on withdrawal configuration | `ecopon-2025` `core_usp_InsertLoyaltyWithdrawalConfiguration_V2.sql:49-70`; `docs/API.md:31-32` | Existing | — | — |
| R14 | Cashback not available (loyalty campaigns) | admin message "Cashback is not available."; `FixedCashbackAmount = NULL` | `LoyaltyCampaignAdminController.cs:326`; `LoyaltyCampaignAdminRepository.cs:194` | Existing (disabled) | vs. proposed paper cashback R11 | — |
| R15 | **Paper = 10 SAR** | *(not found in any inspected source)* | — | **UNVERIFIED** (brief only) | vs. R2 (500 pts/paper) and R12 (100 pts/SAR) | Business owner must supply source |
| R16 | **Each point = 0.020 halala** | *(not found in any inspected source)* | — | **UNVERIFIED** (brief only) | vs. R12 (100 pts = 1 SAR → 1 pt = 1 halala) and draft R18 (1 pt = 0.025 SAR) | Finance owner must reconcile |
| R17 | **Max papers / max additional competition chances** | *(not found in any inspected source)* | — | **UNVERIFIED** (brief only) | Conflicting limits cannot be reconciled without a source | Product owner must supply limits |
| R18 | Draft loyalty-engine values (NOT Gold Paper) | "1 Point = 0.025 SAR" / "1 point per 10 SAR spent" / "flat 500 points for signing up" | `E:\Notepad++\new 14.txt:113-122` | Draft notes | Three different point values across sources | Needs single source of truth |

## 3. Explicit conflicts to reconcile (not resolved here)

1. **Point value disagreement.** R2 (500 points = 1 paper, unapproved) vs. R12 (100 points = 1 SAR,
   withdrawal) vs. R16 (0.020 halala/point, unverified) vs. R18 (0.025 SAR/point, draft). These
   cannot be combined without an authoritative decision. In particular, "500 points = 10 SAR paper"
   is **not** stated by any single inspected source; it is an inference across two different features.
2. **Paper monetary value (R15) is unverified.** Nothing in source assigns a SAR value to a paper.
3. **Max papers / max chances (R17) are unverified** and the brief itself calls them conflicting.
4. **Purchased vs. earned papers (R9/R16).** The POC models only conversion-sourced papers; purchase
   and gift are explicitly out of scope and unauthorized (R11).
5. **Cashback.** Disabled for loyalty campaigns today (R14); proposed for papers but unauthorized
   (R11).

## 4. What is approved vs. proposed (summary)

- **Approved / existing in production:** loyalty point earning via campaign SPs, eWallet ledger,
  voucher redemption, cash withdrawal at an admin-configured `PointsPerCurrencyUnit`, competition
  participation/extra-chances. (See documents 03 and 04.)
- **Proposed only (POC):** points→paper conversion, paper issuance, whole-paper math, idempotent
  conversion.
- **Not authorized at all:** paper purchase, sale/transfer, gifting, cashback, expiry, "papers for
  extra competition chances".
- **Unverified (brief only, no source):** 10 SAR/paper, 0.020 halala/point, SMPoints label, max
  papers, max additional chances.

> No value in this document may be treated as a final business rule. Every Gold Paper number remains
> subject to explicit business approval.

# 07 — Deployment, Operations, and Testing

> Describes how the **existing** eCopon loyalty platform is built, configured, operated, and tested.
> Secrets are referenced by key name only; no credential values are reproduced. There is **no Gold
> Paper deployment, configuration, or test** in eCopon today.

## 1. Solutions and deployable units

| Unit | Project | Type | Notes |
| --- | --- | --- | --- |
| Admin + services | `ecopon-2025/Ecopon.sln` | ASP.NET host(s) + class libs | Admin loyalty/campaign/voucher APIs, EF `DatabaseContext` |
| Consumer mobile backend | `ECoponDashboard` / `Mobile.Api` | ASP.NET Web API | Consumer EWallet/Redeem/Competition endpoints; path base `/mobile2026` |
| Voucher distribution | `Ecopon.VoucherConsoleApp` | Console/batch | `coc_DistributePrivateVouchers` |
| Shared data | `Ecopon.EF.DB` (`DatabaseContext.cs`) | EF Core | `coc_Consumers` and related entities |

> Exact hosting (IIS/App Service) and CI/CD pipelines were **not found** in the inspected source;
> recorded as **UNVERIFIED** rather than asserted.

## 2. Configuration keys (names only — values omitted)

Verified configuration surfaces from source:

- **Connection strings:** standard `appsettings.json` `ConnectionStrings` section (values omitted).
- **JWT / auth:** token validation parameters consumed by `CurrentUserService`; claim names
  `UserID`, `ConsumerID`, `CorporateId`.
- **Route base:** `/mobile2026` + versioned `api/v1.0/mobile/...` for `Mobile.Api`.
- **Admin-configured business values (data, not app config):**
  - `coc_LoyaltyWithdrawalConfiguration.PointsPerCurrencyUnit` (withdrawal rate).
  - `coc_LoyaltyCampaigns` targets/points (earning rules).
  - Voucher catalog rows.

> No Gold Paper configuration key exists. The **POC** (separate repo) introduced `Poc:Mode` and an
> optional `Poc:ConnectionString`; these are **not** part of eCopon and must not be deployed into it.

## 3. Database deployment

- Existing loyalty objects are deployed via the SQL scripts under the loyalty/`Phase2` script folders
  (e.g. `core_usp_Loyalty_AwardCampaignPoints_V2.sql`, `LoyaltyCampaign_*.sql`,
  `udt_LoyaltyVoucherCode`). See [`02-Database-Design.md`](02-Database-Design.md) for which objects
  have CREATE scripts vs. SP-usage-only columns.
- **Most consumer eWallet SP bodies are not in the inspected repo** — their deployment lives outside
  the checked-out source. This is a documentation gap, not a defect (see document 08).

## 4. Operations

- **Idempotency & concurrency** for earning are enforced inside
  `core_usp_Loyalty_AwardCampaignPoints_V2` (unique index `UQ_LoyaltyAwards_Campaign_User_Event`,
  `UPDLOCK/HOLDLOCK`, `THROW 51002/51010`). Operationally, duplicate submissions are de-duped by
  `@IdempotencyKey`.
- **Status history** is captured in `coc_LoyaltyTransactionStatusHistory` and
  `coc_UserLoyaltyWalletLedger`, giving an audit trail for point movements.
- **Error signaling:** SP `THROW` codes (51002, 51005 VOUCHER_EXPIRED, 51010) surface as handled
  failures in the repositories.
- **Monitoring/alerting/runbooks** were **not found** in source — **UNVERIFIED**.

## 5. Testing

### 5.1 Existing eCopon tests
- No loyalty/eWallet automated test project was located in the inspected eCopon source — recorded as
  **UNVERIFIED** (absence of evidence, not evidence of absence).

### 5.2 POC tests (separate repo — for reference only)
- `Gold-Paper-POC/tests/GoldPaperPoc.Tests` contains the only automated coverage of the proposed
  conversion behavior: whole-paper math, `InsufficientBalance` > `NoWholePaper` precedence, idempotent
  retry (no double debit), sticky rejection, mode validation, SQL-guard retention. **97 tests passed**
  in the POC.
- **These are POC results and must not be presented as production verification of eCopon.** They
  demonstrate the proposed rules in isolation only.

## 6. Deployment readiness for Gold Paper

Gold Paper is **not deployable** in eCopon today: there is no feature code, no schema, no config, and
no test coverage inside eCopon. A deployment plan is outlined in
[`08-Implementation-Gaps-and-Technical-Plan.md`](08-Implementation-Gaps-and-Technical-Plan.md) and is
a proposal only.

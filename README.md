# Gold Paper — Technical Overview (eCopon System)

> **Scope.** This documentation set describes how a **Gold Paper** feature would fit into the
> **existing eCopon platform**, based on a read-only inspection of the actual eCopon source
> repositories. It is **not** documentation of the standalone `Gold-Paper-POC` scaffold. The POC is
> referenced only as *proposed / experimentally verified* behavior and is always labelled as such.
>
> **No source code, SQL scripts, database objects, or permissions were modified** to produce this
> documentation. No production database was contacted.

## 1. Headline finding

**Gold Paper is not implemented in the eCopon application.** A repository-wide search of both
inspected eCopon solutions found **no** occurrence of `Gold Paper`, `GoldPaper`, `Golden Paper`, or
`SMPoints`, and **no** points-to-paper conversion, paper issuance, paper purchase/sale/gift, or paper
redemption/expiry logic.

The only use of the word "Paper" in the eCopon source is the unrelated **Paper Draw** competition
feature (`ScreenEnum.AddPaperDrawWinners`, `core_coc_GetPaperDrawNames`,
`core_coc_GetPaperDrawWinners`). That is a competition-winners mechanism, not a loyalty paper.

What **does** exist is a substantial, reusable **loyalty / eWallet / voucher / withdrawal /
competition** platform that a future Gold Paper feature could build on. This documentation describes
that existing platform in detail and separates it clearly from the proposed Gold Paper design.

## 2. Purpose and scope of Gold Paper (as proposed)

The Gold Paper concept (sourced only from the `Gold-Paper-POC` repository and the task brief) is a
**points-to-paper conversion**: a consumer converts loyalty points into whole "papers" at a fixed
rate, with a remainder staying in the balance. Every proposed rule — including the **500 points per
paper** rate — is explicitly marked in the POC as *"a POC test assumption, not an approved production
business rule"* and the POC persists `RateIsApprovedBusinessRule = 0`.

See [`05-Business-Rules-and-Requirements.md`](05-Business-Rules-and-Requirements.md) for the full
requirements table and the unresolved conflicts (e.g. 500 points/paper vs. the loyalty-withdrawal
rate of 100 points = 1 SAR; the "10 SAR per paper" and "0.020 halala per point" values could **not**
be located in any inspected source and are recorded as **unverified**).

## 3. Position within the eCopon platform

Two eCopon solutions were inspected:

| Solution | Role | Path |
| --- | --- | --- |
| **ecopon-2025** (`Ecopon.sln`) | Admin web app + API services + business/repository + EF model. "Main" eCopon repo (this `docs/` lives here). | `E:\eCopon-Code\Smart-Match\Smart-Match\ecopon-2025` |
| **ECoponDashboard backend** | Consumer/mobile API (`Mobile.Api`) + Clean-Architecture application/domain/infrastructure. | `E:\ecopon-new-dashboard-latest\Smart-Match\ECoponDashboard\backend` |

A Gold Paper feature would most naturally sit **next to the existing loyalty eWallet**: the consumer
balance it would debit is the loyalty-points balance exposed by `Mobile.Api` (`EWallet/total-points`
→ `core_usp_Loyalty_GetTotalPointsByUserId_V2`) and awarded by the loyalty campaign engine
(`core_usp_Loyalty_AwardCampaignPoints_V2`). See
[`01-System-Architecture.md`](01-System-Architecture.md).

## 4. Current implementation status (summary)

Classification legend: **(1)** Implemented & verified in eCopon source · **(2)** Partially
implemented · **(3)** Existing reusable functionality, not Gold Paper-specific · **(4)** Specified/
proposed, not implemented · **(5)** Unverified due to missing source/evidence.

| Capability | Status | Evidence anchor |
| --- | --- | --- |
| Consumer loyalty points balance read | **(3)** | `Mobile.Api` `EWalletController.total-points` → `core_usp_Loyalty_GetTotalPointsByUserId_V2` |
| Loyalty point earning (campaign: registration/referral/participation) | **(2/3)** SP + repo exist; **not wired to any HTTP route** | `LoyaltyCampaignAdminRepository.CallAward` → `core_usp_Loyalty_Award*Points_V2` |
| Loyalty wallet transaction ledger & history | **(3)** | `core_usp_Loyalty_GetWalletTransactions_V2`; `coc_Wallet_LoyaltyTransactions` |
| Points → eWallet voucher redemption | **(3)** | `Mobile.Api` `RedeemController` → `core_usp_Loyalty_RedeemEWalletVoucher_V2` |
| Points → cash withdrawal (points×`PointsPerCurrencyUnit`) | **(3)** | `Mobile.Api` `EWallet/withdrawal/*` → `core_usp_Loyalty_CreatePointWithdrawal_V2` |
| Competition participation / extra chances / draws | **(3)** | `ConsumerProfile.my-extra-chances` → `core_coc_GetMyCompetitionExtraChances_2026` |
| Consumer identity & eligibility | **(3)** | `coc_Consumers` (`CocConsumer`), JWT claims (`ConsumerID`, `UserID`) |
| **Gold Paper: points → paper conversion** | **(4)** proposed (POC only) | `Gold-Paper-POC` `ConversionCalculator`; not in eCopon |
| **Gold Paper: paper issuance / status / expiry** | **(4)** proposed (POC only) | `Gold-Paper-POC` `PocPaper`; not in eCopon |
| **Gold Paper: purchase / sale / gift / cashback of papers** | **(4)** not implemented / not authorized | POC README "Not included, and not authorized yet" |
| Physical DDL of eWallet tables (`coc_UserLoyaltySummary`, `coc_Wallet_LoyaltyTransactions`, …) | **(5)** no `CREATE TABLE` in repo | Known only via SP column usage / EF entities |

## 5. Relevant application and service components

- **Consumer loyalty (mobile):** `Mobile.Api` `EWalletController`, `RedeemController`,
  `LoyaltyRepository` (13 `core_usp_Loyalty_*_V2` stored procedures).
- **Loyalty admin (web):** `Ecopon.Web.Application` `LoyaltyCampaignAdminController`,
  `LoyaltyWalletTransactionsController`, `LoyaltyVouchersController`,
  `LoyaltyWithdrawalController`; service `MobileSettingsService`/`LoyaltyCampaignAdminService`;
  repositories `MobileSettingsRepository`, `LoyaltyCampaignAdminRepository`.
- **Loyalty data model (SQL + EF):** `Scripts/Mobile/*.sql`, `Scripts/Mobile/Phase2/*.sql`,
  `Ecopon.EF.DB/DatabaseContext.cs`.

## 6. Supporting documents

| Document | Contents |
| --- | --- |
| [`01-System-Architecture.md`](01-System-Architecture.md) | Solutions, projects, service boundaries, auth, Mermaid architecture diagram. |
| [`02-Database-Design.md`](02-Database-Design.md) | Existing loyalty/consumer schema, verified vs. inferred vs. POC. |
| [`03-API-and-Service-Implementation.md`](03-API-and-Service-Implementation.md) | Verified endpoints, DTOs, handlers, repositories, SPs. |
| [`04-End-to-End-Business-Flows.md`](04-End-to-End-Business-Flows.md) | Verified flows with sequence diagrams; proposed flows separated. |
| [`05-Business-Rules-and-Requirements.md`](05-Business-Rules-and-Requirements.md) | Requirements table and unresolved conflicts. |
| [`06-Integration-and-Dependencies.md`](06-Integration-and-Dependencies.md) | Dependencies on loyalty, identity, voucher, competition, referral, wallet. |
| [`07-Deployment-Operations-and-Testing.md`](07-Deployment-Operations-and-Testing.md) | Projects, config keys (secrets omitted), tests, operations. |
| [`08-Implementation-Gaps-and-Technical-Plan.md`](08-Implementation-Gaps-and-Technical-Plan.md) | Evidence-based gap analysis and proposed sequence. |

## 7. Key dependencies and unresolved questions

- **Which balance is "SMPoints"?** `coc_Consumers.BalancePoint` is a computed
  `LoyaltyPoint - RedeemPoint` (legacy), while the eWallet uses
  `coc_UserLoyaltySummary.CurrentPointsBalance` (Phase 2). These are **separate** balances; no source
  proves they are interchangeable, and neither is labelled "SMPoints". See
  [`02-Database-Design.md`](02-Database-Design.md).
- **Paper monetary value is unverified.** No inspected source defines "10 SAR per paper" or "0.020
  halala per point". The only point↔currency rate in source is the loyalty-withdrawal
  `PointsPerCurrencyUnit` (admin-configured; example comment "100 points = 1 SAR").
- **Earning is not exposed over HTTP.** The award stored procedures exist and have C# call sites in
  the admin repository, but no controller route invokes them.
- **eWallet physical DDL is absent from the repo** — table shapes are known only from SP column usage
  and EF entities.

## 8. Source traceability

Every significant claim in these documents cites a repository, relative file path, and a class /
method / endpoint / stored-procedure / table name. Where an object is referenced in code but its
database definition is not present in the inspected repositories, the document states that explicitly
rather than inventing columns or constraints.

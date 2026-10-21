# 03 — API and Service Implementation (verified)

> Only endpoints that exist in source are listed as implemented. Proposed Gold Paper endpoints are in
> a clearly separated section at the end. Stored-procedure **bodies** for the mobile loyalty SPs are
> **not present** in the inspected repositories (only their names and call sites); this is noted where
> relevant.

Base URL for consumer endpoints: `/mobile2026/api/v1.0/mobile/{Controller}/{action}`.

## 1. Consumer loyalty / eWallet — `Mobile.Api`

Controller: `Mobile.Api/Controllers/V1/EWalletController.cs`. Repository:
`ECoponDashboard.Infrastructure/Implementations/Repositories/Mobile/LoyaltyRepository.cs` (all access
via Dapper `CommandType.StoredProcedure`). Auth: no `[Authorize]` on the controller; each handler
returns **401** when the JWT user id is `<= 0`.

| Method | Route (relative) | Request | Response | Handler | SP (via `LoyaltyRepository`) |
| --- | --- | --- | --- | --- | --- |
| GET | `total-points` (L16) | — | `TotalPointsByUserIdResult` | `GetTotalPointsByUserIdHandler` | `core_usp_Loyalty_GetTotalPointsByUserId_V2` (L45) |
| GET | `withdrawal` (L21) | — | `WithdrawalScreenResult` | `GetUserWithdrawalHandler` | `core_usp_Loyalty_GetWithdrawalScreen_V2` (L81) |
| POST | `withdrawal/validate` (L26) | `ValidateWithdrawalCommand` | `ValidateWithdrawalResult` | `ValidateWithdrawalHandler` | `core_usp_Loyalty_ValidateWithdrawal_V2` (L130) |
| POST | `withdrawal/confirm` (L30) | `CreatePointWithdrawalCommand` | `UserPointWithdrawalResult` | `UserPointWithdrawalHandler` | `core_usp_Loyalty_CreatePointWithdrawal_V2` (L171) |
| GET | `withdrawal/history` (L34) | `GetUserWithdrawalHistoryQuery` | `Pagination<UserWithdrawalHistoryResult>` | `GetUserWithdrawalHistoryHandler` | `core_usp_Loyalty_GetPointWithdrawalHistory_V2` (L219) |
| GET | `withdrawal/status/{conversionId}` (L42) | `GetUserWithdrawalHistoryDetailQuery` | `UserWithdrawalDetailsResult` | `GetUserWithdrawalHistoryDetailHandler` | `core_usp_Loyalty_GetPointWithdrawalDetails_V2` (L267) |
| GET | `transactions` (L49) | `GetLoyaltyTransactionsQuery` | `Pagination<LoyaltyTransactionHistoryResult>` | `GetLoyaltyTransactionsQueryHandler` | `core_usp_Loyalty_GetWalletTransactions_V2` (L517) |
| GET | `transaction/details` (L62) | `GetLoyaltyTransactionDetailsQuery` | `LoyaltyTransactionDetailsResult` | `GetLoyaltyTransactionDetailsQueryHandler` | `core_usp_Loyalty_GetWalletTransactionDetails_V2` (L569) |
| GET | `vouchers` (L69) | `GetRedeemVouchersQuery` | `Pagination<RedeemVoucherResult>` | `GetRedeemVouchersQueryHandler` | `core_usp_Loyalty_GetEWalletVouchers_V2` (L310) |
| GET | `voucher/details/{voucherID}` (L77) | `GetRedeemVoucherDetailsQuery` | `RedeemVoucherDetailsResult` | `GetRedeemVoucherDetailsHandler` | `core_usp_Loyalty_GetEWalletVoucherDetails_V2` (L373) |
| GET | `voucher/validate/{voucherID}` (L85) | `GetValidateVoucherRedemptionQuery` | `ValidateVoucherRedemptionResult` | `GetValidateVoucherRedemptionQueryHandler` | `core_usp_Loyalty_ValidateEWalletVoucherRedemption_V2` (L414) |
| POST | `redeem/voucher` (L93) | `RedeemVoucherCommand` | `RedeemVoucherTransactionResult` | `RedeemVoucherCommandHandler` | `core_usp_Loyalty_RedeemEWalletVoucher_V2` (L459) |
| GET | `redemption/status` (L101) | `GetVoucherRedemptionDetailsQuery` | `VoucherRedemptionTransactionDetailsResult` | `GetVoucherRedemptionTransactionDetailHandler` | `core_usp_Loyalty_GetVoucherRedemptionStatus_V2` (L614) |

`RedeemController.cs` exposes alias routes (`eWallet/vouchers*`, `eWallet/vouchers/transaction`) that
map to the same voucher commands/handlers/SPs.

- **Transaction type constants:** `ECoponDashboard.Application/Helper/LoyaltyTransactionTypes.cs`
  L4–7 (`EARNED_POINTS`, `Redemption`, `Withdrawal`).
- **Validators:** none found under `LoyaltyFeatures` (withdrawal body relies on model binding).
- **Error handling:** handler-level 401 for missing user; SP errors surfaced through the `Result<T>`
  envelope; voucher expiry raised as SP error `VOUCHER_EXPIRED` (SQL 51005, `LoyaltyRepository.cs`
  L665–666).

> **Balance read (Gold Paper relevance).** `total-points` is the consumer-facing points balance a
> Gold Paper conversion would preview/debit. The SP body (`core_usp_Loyalty_GetTotalPointsByUserId_V2`)
> is not in these repositories, so its exact source table (`coc_UserLoyaltySummary` vs. a legacy
> column) could **not** be verified here.

## 2. Loyalty admin — `ecopon-2025`

### 2.1 Loyalty campaign admin (earning-rule configuration + award SP wrappers)
- Web: `Ecopon.Web.Application/Controllers/LoyaltyCampaignAdminController.cs` (`Configure` L36–64,
  `Save` L104–119). API: `Ecopon.API.Service/.../LoyaltyCampaignAdminController.cs` L23–93
  (`api/LoyaltyCampaignAdmin/*`, `[AllowAnonymous]` L12–13). Service:
  `LoyaltyCampaignAdminService.cs`. Repository: `LoyaltyCampaignAdminRepository.cs`.
- **Award stored-procedure call sites** (`LoyaltyCampaignAdminRepository.CallAward` L479–498):

| SP | Call site | Notes |
| --- | --- | --- |
| `dbo.core_usp_Loyalty_AwardRegistrationPoints_V2` | `LoyaltyCampaignAdminRepository.cs` L451 | `@UserID`, `@CorporateID`, `@EventDate`, `@OverallResult` |
| `dbo.core_usp_Loyalty_AwardReferralPoints_V2` | L461 | `@ReferrerUserID`, `@InviteeUserID`, `@CorporateID`, `@EventDate` |
| `dbo.core_usp_Loyalty_AwardECouponParticipationPoints_V2` | L472 | `@UserID`, `@ECouponParticipationID` |
| `dbo.core_usp_Loyalty_AwardCampaignPoints_V2` | **no C# call site** | invoked by the award SPs |

> **Important status note.** These award SPs are reachable from the admin **repository/service** layer
> but **no controller route invokes `AwardRegistration`/`AwardReferral`/`AwardECouponParticipation`**.
> Point earning is therefore **implemented at the data layer but not exposed over HTTP** in the
> inspected code (classification **(2) Partially implemented**).
- Cashback is disabled in admin: `FixedCashbackAmount = NULL` (`LoyaltyCampaignAdminRepository.cs`
  L194); UI message `LoyaltyCampaignAdminController.cs` L326.

### 2.2 Loyalty wallet transactions (admin read-only)
- Web `LoyaltyWalletTransactionsController` (`Configure` L39–151, `Details` L155–252). API
  `api/LoyaltyWalletTransactions/GetTransactions|GetTransaction`. Repository
  `MobileSettingsRepository.GetLoyaltyWalletTransactions` L893–973 →
  `USP_Mobile_GetLoyaltyWalletTransactionsAdmin_V2`; detail L976–1018 →
  `core_usp_GetLoyaltyWalletTransactionAdmin_V2`. SP bodies present under `Scripts/Mobile`.

### 2.3 Loyalty voucher admin (catalog)
- API `api/LoyaltyVouchers/GetVouchers|GetVoucher|InsertVoucher|UpdateVoucher|SetActive`
  (`LoyaltyVouchersController.cs` L26–67). Repository `MobileSettingsRepository` L732–891 →
  `USP_Mobile_GetLoyaltyVouchers_V2`, `core_usp_GetLoyaltyVoucher_V2`,
  `core_usp_InsertLoyaltyVoucher_V2`, `core_usp_UpdateLoyaltyVoucher_V2`,
  `core_usp_SetLoyaltyVoucherActive_V2` (uses TVP `dbo.udt_LoyaltyVoucherCode`).

### 2.4 Loyalty withdrawal admin (rules + wallet types)
- API `api/LoyaltyWithdrawal/*` and `api/LoyaltyWithdrawalWalletTypes/*`. Repository
  `MobileSettingsRepository` L266–687 → `USP_Mobile_GetLoyaltyWithdrawalConfigurations_V2`,
  `core_usp_GetLoyaltyWithdrawalConfiguration_V2`, insert/update/enable SPs; wallet-type SPs.
  Field `PointsPerCurrencyUnit` is the admin-set points→currency rate.

## 3. Consumer identity & profile (reused for eligibility)

- `Mobile.Api/Controllers/V1/AccountController.cs`: `login` → `core_USP_Web_IsValidUser`
  (`AccountRepository.ValidateUser` L22–25); `Register` → EF insert `CocUser` + `CocConsumer` with a
  loyalty seed from `CocatLoyaltyPointSetup` id 101 (`CreateConsumerCommandHandler.cs` L44–79).
- `Mobile.Api/Controllers/V1/ConsumerProfileController.cs`: profile reads/writes,
  `my-extra-chances` (L117) → `core_coc_GetMyCompetitionExtraChances_2026`
  (`ConsumerProfileRepository.cs` L413).

## 4. What is NOT present (and must not be invented)

- **No** `GetTotalPoints` (no-suffix), **no** `USP_Mobile_SaveECouponPurchasesValidateInvoice` in
  `ecopon-2025` (the validate-invoice SP is referenced only from `Mobile.Api`:
  `ECoponDashboard.Domain/Constants/StoredProcedures.cs` L19).
- **No** Gold Paper endpoint of any kind in either repository.

## 5. Proposed Gold Paper endpoints (design only — NOT implemented)

The following are **recommendations**, mirroring the POC contract, provided only because they are
supported by the proposed requirements. They do **not** exist in eCopon.

| Proposed method/route | Purpose | Reused existing dependency |
| --- | --- | --- |
| `POST /…/mobile/GoldPaper/convert` | Convert points → whole papers (idempotent) | debits loyalty balance (eWallet) |
| `GET /…/mobile/GoldPaper/summary` | Balance, rate, papers, successful conversions | reads loyalty balance |
| `GET /…/mobile/GoldPaper/history` | Conversion / paper / event history | new paper ledger |

A proposed endpoint must not be presented as production behavior. See
[`08-Implementation-Gaps-and-Technical-Plan.md`](08-Implementation-Gaps-and-Technical-Plan.md).

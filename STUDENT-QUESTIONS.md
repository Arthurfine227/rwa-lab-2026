# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:

A useful accounting relation is reported backing >= outstanding obligations, with all terms expressed in six-decimal USDC units. For this design, reported backing is vault.reserveBalance() + custodian.realHoldings(). Outstanding obligations include vault.totalClaimValue(), which values all unburned shares at the current NAV, and queue.totalClaimable(), because settled shares have already been burned but their cash may still await withdrawal. If looking at the whole system, cash held by the queue must also be included on the backing side. A more precise liability calculation replaces the current NAV value of escrowed shares with queue.pendingAssets(), since those tickets lock an earlier NAV. Equality is only an ideal accounting snapshot; surplus or rounding can make an inequality more appropriate.

This relation checks consistency between recorded numbers. It cannot establish that securities exist, that the SPV owns them, that they are unencumbered, or that they can be sold promptly. Ex2 demonstrates that a custodian can record a purchase without receiving cash. Independent custody statements, reconciliations, and enforceable ownership documents are therefore necessary evidence beyond the chain.

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

The legal structure determines what an on-chain claim actually represents. An ERC-20 balance records an address and quantity, but it does not identify who legally owns the Treasury bills, which creditors can reach them, or how a holder enforces redemption after the issuer fails. An SPV can separate fund assets from the issuer's operating liabilities. The technical system must be designed around that separation: subscriptions, share issuance, custody records, and redemption instructions should refer to the same legal fund and reconcile with its ownership register.

Bankruptcy remoteness does not remove every custodian risk. If securities are properly segregated and held for the SPV, the intended outcome is recovery or transfer of those securities rather than treating them as the custodian's ordinary assets. Missing assets, commingling, fraud, or disputed records can still delay or reduce recovery. A token holder's entitlement depends on the fund documents and applicable law; the Solidity contract alone does not guarantee a direct Treasury claim or deposit insurance. I would require clear beneficial ownership terms, a replacement custodian process, and a recovery procedure linking approved holders to legal claims.

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:

A single reporter can fail through compromised credentials, deliberate fraud, valuation mistakes, delayed updates, or selective publication. A stolen key could submit an inflated NAV; separate signing keys, hardware protection, and threshold approval reduce that risk. Intentional misreporting needs independent valuation and custodian reconciliation, backed by contractual liability and meaningful penalties. Decimal or pricing errors call for range checks, unit tests, and comparison against an independent reference. Operational outages require freshness limits, backup reporters, and a published recovery procedure. Selective timing can advantage insiders, so updates should follow a disclosed schedule and use consistent valuation cutoffs.

Access control is necessary but does not prove the truth of a number. In this exact lab, MockPriceFeed.setPrice() is publicly callable, so anyone can bypass the vault's REPORTER_ROLE by writing directly to the feed. That additional path must be restricted before treating the role as a complete security boundary. Moreover, the queue invariants prove escrow and cash accounting rather than NAV accuracy: they can remain green under a false price. Production controls therefore need independent evidence as well as correct authorization.

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:

A stale NAV creates an information advantage even when its last published value was honest. Suppose the feed still reports 1.00 while the true share value has risen to 1.10. An informed investor can subscribe at the old price, receive too many shares, and participate in the higher valuation after the feed updates. The extra entitlement dilutes existing holders. If the true value instead falls to 0.90, an informed holder can enqueue redemption while the published NAV remains 1.00. The queue locks that excessive payout, shifting the loss onto remaining holders or the issuer.

The attacker needs neither a stolen reporter key nor a fabricated observation; transaction timing exploits a missing update. I would require a reliable update timestamp, reject new subscriptions and redemption requests after a maximum age, and use a common valuation cutoff for orders. Existing settled claims should remain payable when cash is available. This mock returns the current block timestamp from latestRoundData(), even without a price update, so simply checking its updatedAt would not detect freezing. The feed must store the actual publication time before a freshness rule can work.

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:

The queue converts a share position into a fixed cash claim when enqueue is called. If NAV rises afterward, the redeemer receives the earlier locked amount and gives up the subsequent appreciation. Holders who remain invested retain exposure to the higher NAV and may benefit economically from assets left after the fixed redemption liability is paid. If NAV falls, the allocation reverses: the redeemer keeps the earlier higher entitlement while remaining holders or the issuer bear the funding shortfall. This comparison assumes actual asset values follow the reported NAV; changing the oracle alone creates no cash.

Locking can be reasonable when a fund publishes a valuation cutoff and clearly states that exit ends investment exposure at that point. However, it invites timing advantages if the price is stale or a large order arrives just before a known update. I would use a common forward-priced valuation window and disclose fees or dilution adjustments. Applying settlement-time NAV is another option, but it gives waiting investors uncertain payouts and makes settlement timing consequential. In either design, vault.reserveBalance() need not cover queue.pendingAssets(): securities can hold value while the liquid USDC buffer is insufficient.

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:

The Reserve Primary Fund valued its shares at 97 cents in September 2008 and delayed, then suspended, redemption payments during severe market stress. USDC's March 2023 crisis involved uncertainty about reserves at Silicon Valley Bank and banking availability over a weekend. Circle resumed processing redemptions on Monday, and by March 15 reported substantially clearing its backlog. These cases distinguish impairment and orderly asset liquidation from temporary access to cash and payment rails. Sources: [SEC](https://www.sec.gov/divisions/investment/guidance/reservefundmmffaq.htm) and [Circle](https://www.circle.com/blog/march-15-update-on-usdc-operations).

For a tokenized fund, closing redemption means restricting entry into, or settlement of, the conversion process; it should not erase tickets or change their priority secretly. I would preserve FIFO records, publish reasons and reopening conditions, and allow already funded claims to be collected. The lab's token pause blocks minting, transfers into escrow, and burns during settlement. However, claim() transfers USDC and can still operate: it does not call the paused token. A single token switch therefore does not exactly control every redemption stage. Separate subscription, enqueue, and settlement controls would better handle a liquidity squeeze while preserving cash withdrawals already owed.

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:

For a regulated fund, admission should follow the product's legal eligibility requirements and a documented risk policy. Verifying both transfer endpoints prevents an approved holder from moving shares to an unverified or prohibited recipient. That protection also imposes a practical cost: ordinary users must finish identity checks before subscribing, receiving transfers, or entering processes that change token balances. The interface should explain eligibility and verification status before accepting an order, rather than relying on an unexplained revert.

I would separate policy approval from routine execution. A compliance team could maintain approved addresses under documented procedures, while material policy changes require governance approval and legal review. Address removal should have recorded reasons, periodic review, and an appeal or recovery route where lawful. Time-limited emergency restrictions can handle urgent cases without allowing permanent unexplained freezes. Wallet changes also need a secure re-verification process so losing a key does not automatically destroy the economic claim. Administrative actions should use distinct roles, monitored events, and multiple approvers for sensitive changes. The ability to move the eligibility line should follow accountable authority rather than whoever happens to possess one key.

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:

An issuer may need freeze and seizure powers to meet sanctions obligations, enforce court orders, correct fraud, or recover assets after a verified wallet compromise. Those powers can make a regulated security usable in its legal environment. Conversely, they undermine unconditional bearer ownership: a compromised or abusive administrator can censor holders, destroy balances, or favor particular investors. Investors should understand those risks before buying, and the terms should specify whether a burn cancels a claim, compensates a holder, or implements an authorized transfer.

Ex4.3 reveals an operational conflict: removing Alice from the whitelist freezes transfers but also prevents burning her shares. Temporarily restoring approval to permit seizure opens an avoidable transfer window. I would implement a narrowly scoped forced-action route with explicit authorization and audited accounting, while retaining normal endpoint checks for ordinary transfers. Compliance staff should hold the routine whitelist role; a separate multisignature legal enforcement authority should approve seizure. Minting, pausing, and enforcement should use separate keys. Event logs, legal case references, thresholds, review deadlines, and compensation records would make exceptional powers traceable without pretending they are trustless.

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook` |
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay` |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation` |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and say which plank it breaks:

> Your answer:

My highest-priority scenario is an attacker directly calling MockPriceFeed.setPrice() to inflate NAV, then enqueueing existing shares at that artificial price before settlement. This breaks plank (b), attestation: the vault reporter role does not protect the feed itself. The queue locks the inflated payout, so subsequent price correction does not repair the ticket. If sufficient reserves are available, settlement can fund an excessive claim while both queue invariants still hold. Access to approved shares limits the attack, and insufficient reserves can make settlement revert, but neither makes the price trustworthy. I would restrict feed writes to an authorized updater, require independent valuation checks, and reject orders at stale or implausible prices.

# 06 · Challenges, Trade-offs & Lessons

## Challenges

### 1. Game UI updates break the model overnight
**Problem:** a seasonal update changed the eFootball result screen (new fonts and layout), and auto-resolution for that game dropped from ~88% to ~40% in a day.
**Solution:** monitoring of auto-resolution rate **per game** with an alert when it drops more than 15 points. A "UI drift" playbook: collect ~150 new screenshots from the review queue → annotate → fine-tune from the previous weights (≈1 hour on GPU) → regression test → deploy. Turnaround went down to **under 48 hours**. Nothing broke in the meantime, because low-confidence cases simply went to admins.

### 2. Phone photos of TVs
**Problem:** console players photograph their TV, with angles, glare and moiré. Accuracy on these photos was ~79%.
**Solution:** perspective rectification + CLAHE + selective denoising, and training augmentations that simulate these photos. Accuracy rose to ~92%.

### 3. OCR on stylized game fonts
**Problem:** Tesseract misread stylized score digits (`8` ↔ `3`, `1` ↔ `7`).
**Solution:** digits detected as YOLO classes (no OCR). OCR is kept only for player names, and there it matches against **2 known names** with fuzzy matching instead of reading free text.

### 4. Gas UX for non-crypto users
**Problem:** most users had never held POL, and asking them to buy gas tokens would have killed the feature.
**Solution:** relayer-sponsored minting + EIP-712 gasless redemption, with gas priced in USDC and deducted transparently in the quote.

### 5. Nonce management under concurrency
**Problem:** parallel withdrawals from one relayer wallet caused `nonce too low` / replacement errors during testing.
**Solution:** a serialized transaction queue + local nonce manager + same-nonce fee-bump replacement for stuck transactions.

### 6. Keeping a real-money reward economy balanced
**Problem:** coins redeemable as crypto attract farming and multi-accounting.
**Solution:** caps, diminishing returns, ownership proofs, payouts delayed until the referred user is verified, and z-score anomaly detection. Every rule was configurable, so the economy could be tuned without deploys.

## Trade-offs I made consciously

| Decision | Alternative | Why |
|---|---|---|
| One YOLO model per game family | One model per game / one giant model | Shared classes (digits, banners) generalize; a per-game YAML handles the differences |
| ONNX Runtime on CPU | GPU inference | ~300 ms latency was fine for this flow, and CPU hosting is much cheaper |
| Voucher NFT backed by USDC | Direct USDC transfers | Auditability, user custody and future collectible rewards, for a small extra gas cost |
| Polygon PoS | Ethereum L1 / other L2s | Very low fees, native USDC, broad wallet support, mature tooling |
| Rules as JSON data | Hard-coded reward logic | The product team could iterate on the economy without engineers |

## What I'd do next

- **Video proof for high-stakes matches:** short clips verified with the same models frame by frame, harder to fake than one screenshot.
- **Direct API verification** where official match APIs exist (e.g. Riot Match-V5 for LoL), so screenshots become a fallback for those games.
- **Account abstraction (ERC-4337)** with a paymaster, for an even simpler wallet experience (social login wallets).
- **A third-party smart-contract audit** before raising per-voucher limits.
- **An ML-based fraud model** to replace the rule- and z-score-based abuse heuristics once enough labelled cases exist.

## Lessons

1. **Design for "I don't know".** A model that hands off when unsure is worth more than one that is always confident.
2. **Data beats architecture.** The biggest gains came from preprocessing and hard-example retraining, not from bigger models.
3. **In Web3, the off-chain part is most of the work.** Ledgers, retries, reconciliation and UX took more effort than the contract itself.
4. **Idempotency is a feature.** Every money-moving path got a unique key and a database constraint behind it.

# Drinkripples customizations — woocommerce-gateway-stripe

**Base:** upstream `woocommerce/woocommerce-gateway-stripe` tag **10.9.0** (latest stable release)
**Branch:** `ripples/10.9.0` · **Scope:** 2 files, ~55 added lines, 2 commits
**Last rebased:** 2026-08-19

All customizations are **opt-in via the `wc_stripe_upe_params` PHP filter**. With no filter active, the plugin behaves identically to stock 10.9.0. The multi-account logic itself (choosing which Stripe account/keys to serve) lives outside this repo, in the Ripples plugin/theme; this fork only makes the checkout JS honor it. Every customized block in the source is marked with a `// Drinkripples customization:` comment — grep for that to find them all.

---

## Change 1 — Payment Element options via server data
`client/classic/upe/payment-processing.js` (commit `5668a243a`)

| Filter key | Effect | Upstream default |
|---|---|---|
| `fonts` | Array of extra [Elements font rules](https://docs.stripe.com/js/elements_object/create#stripe_elements-options-fonts) appended to the fonts option | Only fonts auto-detected from the page |
| `paymentElementWallets` | Merged into the Payment Element `wallets` option, e.g. `{ "applePay": "auto", "googlePay": "auto" }` shows Apple/Google Pay **inside** the Payment Element | `never` for both (wallets render as separate express-checkout buttons) |
| `layout` | Full override of the Payment Element [`layout`](https://docs.stripe.com/payments/payment-element#layout) option, applied after upstream's OC/tabs logic | `tabs`, or accordion when Optimized Checkout is on |

> Note: the old fork **hardcoded** wallets to `'auto'`. It is now filter-driven to keep the diff minimal and behavior-neutral — your plugin must set `paymentElementWallets` to restore it. Beware of wallets appearing twice if express checkout buttons are also enabled.

## Change 2 — API re-creation + element re-mount on "save payment method" toggle
`client/classic/upe/deferred-intent.js` (commit `e1a497c55`)

- `const api` → `let api`.
- New handler, gated on filter key `remountOnSaveToggle` (boolean): when `#wc-stripe-new-payment-method` changes, it empties the UPE containers, resets component state via `initializeUPEComponents()`, rebuilds `WCStripeAPI` from **fresh** `getStripeServerData()` (which may now carry a different account / publishable key), and re-mounts.
- This is the client-side half of multi-Stripe-account support (e.g. one-off payments vs. saved/subscription payments on different accounts).

> Note: the old fork gated this on `isOCEnabled` (a flag upstream has since renamed/reworked) and left a broken half-merged duplicate of the handler after the May 2026 upstream merge. The port uses its own flag and coexists with upstream's new `shouldShowOptimizedCheckout` handler (which updates `setupFutureUsage` in place). If both are active, the re-mount supersedes the in-place update.

## PHP integration (in your Ripples plugin)

```php
add_filter( 'wc_stripe_upe_params', function ( $params ) {
    $params['fonts'] = array(
        array( 'cssSrc' => 'https://example.com/fonts.css' ),
    );
    $params['paymentElementWallets'] = array(
        'applePay'  => 'auto',
        'googlePay' => 'auto',
    );
    $params['layout'] = array( 'type' => 'accordion' ); // optional
    $params['remountOnSaveToggle'] = true; // multi-account remount
    return $params;
} );
```

The filter is applied in `includes/payment-methods/class-wc-stripe-upe-payment-gateway.php` (`javascript_params()`); swap the account keys (publishable/secret) in the same plugin via the gateway settings filters you already use.

## What was in the fork but is NOT a customization

The old fork's diff also contained rebuilt `assets/js/*.min.js`, a regenerated `languages/*.pot`, and a `package-lock.json` bump — all build artifacts, dropped from the port. The May 2026 merge commit contributed nothing except the broken duplicate handler noted above.

## Rebasing onto the next upstream release

1. `git fetch upstream && git checkout -b ripples/X.Y.Z X.Y.Z`
2. `git cherry-pick 5668a243a e1a497c55` (or `git am patches/*.patch`)
3. If conflicts: the anchors are the `options = { … fonts: getFontRulesFromPage() }` block, the `wallets:` block, the layout `if/else`, and the `shouldShowOptimizedCheckout` checkbox handler in `deferred-intent.js`. Re-place the marked blocks near them.
4. Verify: `npm ci --ignore-scripts --engine-strict=false`, then
   `npx wp-scripts test-unit-js --config tests/js/jest.config.js client/classic/upe/__tests__/payment-processing.test.js` (70 tests) and `npm run build:webpack`.
5. Ship: `npm run build` produces the release zip (needs composer + full toolchain).

## Deliverables in this folder

- `patches/0001-*.patch`, `patches/0002-*.patch` — the two commits as mailbox patches (`git am`-able)
- `ripples-10.9.0.bundle` — full git bundle containing tag `10.9.0` and branch `ripples/10.9.0`. To push it to your fork:
  ```bash
  git clone https://github.com/idokd/woocommerce-gateway-stripe.git
  cd woocommerce-gateway-stripe
  git bundle unbundle /path/to/ripples-10.9.0.bundle   # or: git fetch /path/to/ripples-10.9.0.bundle ripples/10.9.0:ripples/10.9.0
  git push origin ripples/10.9.0
  ```

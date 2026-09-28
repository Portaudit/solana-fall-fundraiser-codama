# Codama × Fundraiser — Notes

## Toolchain versions

- anchor-cli: 1.1.2
- node: v22.22.3
- codama CLI (`npx codama --version`): 1.6.3
- @codama/renderers-js: 2.5.0

## Why `fundraiser` and `vault` need an explicit account, but `contributorAccount`, `contributorAta`, and `tokenProgram` don't

`fundraiser`'s seeds are `[b"fundraiser", fundraiser.maker]` (see `programs/fundraiser/src/instructions/contribute.rs`). `maker` is a field read out of the `Fundraiser` account's *data*, not an account passed into the `Contribute` instruction at all — it only appears as an account in `initialize.rs`. Codama's client-side resolver can only derive a PDA from addresses it already has in hand (other passed-in accounts, constants, or program IDs); it can't fetch and read account data to find a seed. So there's no way for it to guess which `fundraiser` PDA we mean, and we have to pass it explicitly.

`vault` is an associated token account with `associated_token::mint = fundraiser.mint_to_raise, associated_token::authority = fundraiser`. Once we've supplied `fundraiser`'s address ourselves (see above) and `mintToRaise` is already a required input, both pieces `vault`'s ATA derivation needs are known, so Codama resolves it automatically — it doesn't need to be listed separately in our `getContributeInstructionAsync` call. It only "needs an explicit account" in the sense that it's downstream of `fundraiser`, which does.

`contributorAccount` is seeded by `[b"contributor", fundraiser, contributor]` — both `fundraiser` (now explicit) and `contributor` (the signer we always pass) are already known, so it resolves on its own. `contributorAta` is just the ATA of `mintToRaise` + `contributor`, both already known. `tokenProgram` isn't a PDA at all — it's the constant SPL Token program address, which Codama bakes in from the IDL. None of these three depend on anything outside the accounts we already had to supply.

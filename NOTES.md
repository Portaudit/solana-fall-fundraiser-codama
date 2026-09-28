# Codama × Fundraiser — Notes

## Toolchain versions

- anchor-cli: 1.1.2
- node: v22.22.3
- codama CLI (`npx codama --version`): 1.6.3
- @codama/renderers-js: 2.5.0

## Why `fundraiser` is required in `ContributeAsyncInput` but optional in `InitializeAsyncInput`

In `initialize.rs`, `fundraiser`'s seeds are `[b"fundraiser", maker.key()]`, and `maker` is itself an account in that same `Initialize` struct (`pub maker: Signer<'info>`). Codama's resolver only needs inputs it already has in hand, and `maker` is right there, so it can compute the PDA itself — `fundraiser` comes out optional in `InitializeAsyncInput`.

In `contribute.rs`, `fundraiser`'s seeds are the same `[b"fundraiser", fundraiser.maker]` — but `Contribute` never takes `maker` as an account at all. `maker` only exists as a *field inside the `Fundraiser` account's own data*. A resolver can't read account data to find a seed (that would mean fetching the account before it knows the account's address), so it has no way to guess which `fundraiser` PDA is meant. We have to pass it in explicitly, and it comes out required in `ContributeAsyncInput`.

This isn't just a Codama limitation, it's a program-design choice: `refund.rs` declares the same `fundraiser` account but *does* take `maker` explicitly and seeds off `maker.key()`, so `fundraiser` is optional again in `RefundAsyncInput`. `contribute.rs` chose to omit `maker` and save one account in every contribute transaction — the cost lands on every client, which now has to already know (or look up) the fundraiser's address to call it.

`vault`, `contributorAccount`, `contributorAta`, and `tokenProgram` are the four accounts that stay optional. `vault` is the ATA of `mint_to_raise` + authority `fundraiser` — once `fundraiser` and `mintToRaise` are known (both required inputs), Codama derives it. `contributorAccount` is seeded `[b"contributor", fundraiser, contributor]` — again both pieces are already known. `contributorAta` is the ATA of `mintToRaise` + `contributor`. `tokenProgram` isn't a PDA at all, it's the constant SPL Token program address baked into the IDL. `contributor` and `fundraiser` are the two required accounts; the other four are optional because every seed they need is either a required input, a constant, or another PDA already resolved from those.

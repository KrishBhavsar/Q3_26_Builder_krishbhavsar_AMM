# AMM Video

A Solana automated market maker (AMM) written in Rust with the Anchor framework.
The program manages a two-token liquidity pool, mints liquidity-provider (LP)
tokens, supports swaps using a constant-product curve, and routes swap fees to
separate treasury accounts.

## What is implemented

The required AMM functionality is implemented through four instructions:

1. `initialize` creates a pool configuration, LP mint, token vaults, and fee
   treasury accounts.
2. `deposit` adds token X and token Y liquidity and mints LP tokens.
3. `withdraw` burns LP tokens and returns the user's proportional share of both
   pool tokens.
4. `swap` exchanges one pool token for the other with slippage protection and
   sends the swap fee to the appropriate treasury.

The integration tests run in [LiteSVM](https://github.com/LiteSVM/litesvm), so
they do not require a running local validator.

## Program information

- Program name: `amm_video`
- Program ID: `6KoUjko5kqLHaF31gdWGBihf8Pw8dUNte2hBpBEJveVe`
- Framework: Anchor `1.0.1`
- Language: Rust 2021
- Token program: SPL Token
- Pool token precision: 6 decimals
- Curve library: `constant-product-curve`

## Repository structure

```text
.
├── Anchor.toml
├── programs/
│   └── amm-video/
│       ├── src/
│       │   ├── constants.rs
│       │   ├── error.rs
│       │   ├── instructions.rs
│       │   ├── lib.rs
│       │   ├── state.rs
│       │   └── instructions/
│       │       ├── deposit.rs
│       │       ├── initialize.rs
│       │       ├── swap.rs
│       │       └── withdraw.rs
│       └── tests/
│           ├── ix_handlers/
│           └── tests.rs
├── migrations/
├── runbooks/
└── Cargo.toml
```

### Main source files

- `src/lib.rs` exposes the Anchor program and its public instruction entry
  points.
- `src/state.rs` defines the persistent `Config` account.
- `src/instructions/initialize.rs` creates the pool accounts.
- `src/instructions/deposit.rs` handles liquidity deposits and LP minting.
- `src/instructions/withdraw.rs` handles LP burning and liquidity withdrawal.
- `src/instructions/swap.rs` handles swaps and fee transfers.
- `src/error.rs` contains AMM-specific errors and curve-error conversions.
- `tests/tests.rs` contains the LiteSVM integration tests.

## AMM concepts used by this project

### Liquidity pool

The pool holds two SPL tokens:

- Token X
- Token Y

The pool uses the constant-product relationship:

```text
x * y = k
```

where `x` and `y` are the current reserves. A swap changes the reserves while
approximately preserving `k`, after accounting for the swap fee and integer
rounding.

### LP tokens

When a user deposits liquidity, the program mints LP tokens to represent the
user's proportional ownership of the pool.

When a user withdraws liquidity, the program:

1. Burns the requested LP tokens.
2. Calculates the user's proportional share of token X and token Y.
3. Transfers both tokens from the pool vaults to the user.

### Basis-point fees

The pool fee is stored as basis points:

```text
100 basis points = 1%
10,000 basis points = 100%
```

For example, the test pool uses a fee of `30`, which equals `0.30%`:

```text
10,000,000 input tokens * 30 / 10,000 = 30,000 fee tokens
```

## Account architecture

Each pool is identified by a configuration PDA derived from the numeric seed:

```text
["config", seed.to_le_bytes()]
```

The pool's LP mint is derived from the config address:

```text
["lp", config_address]
```

The two normal liquidity vaults are associated token accounts owned by the
config PDA:

```text
vault_x = ATA(config, mint_x)
vault_y = ATA(config, mint_y)
```

The fee treasuries are separate token-account PDAs:

```text
treasury_x = PDA(["treasury_x", config_address])
treasury_y = PDA(["treasury_y", config_address])
```

The treasury accounts are not associated token accounts because a single owner
and mint pair can only have one associated token account. The vault already
uses the config PDA as the owner for each mint, so the treasury accounts use
different PDA addresses.

### `Config` account

The persistent configuration stores:

| Field | Meaning |
|---|---|
| `seed` | Identifies the pool configuration. |
| `authority` | Optional authority stored for future administration. |
| `mint_x` | Token-X mint address. |
| `mint_y` | Token-Y mint address. |
| `fee` | Swap fee in basis points. |
| `locked` | Prevents deposits and withdrawals when true. |
| `config_bump` | PDA bump for the config account. |
| `lp_bump` | PDA bump for the LP mint. |

## Instruction reference

### `initialize`

```rust
initialize(seed, fee, authority)
```

Creates:

- The `Config` PDA.
- The LP token mint.
- The token-X vault.
- The token-Y vault.
- The token-X treasury.
- The token-Y treasury.

The initializer pays for all newly created accounts. The fee must be no more
than `10_000` basis points.

### `deposit`

```rust
deposit(amount, max_x, max_y)
```

- `amount`: LP tokens the user wants to receive.
- `max_x`: maximum token X the user permits the program to take.
- `max_y`: maximum token Y the user permits the program to take.

For the first deposit, `max_x` and `max_y` establish the initial pool ratio.
For later deposits, the constant-product curve calculates the required token
amounts. The instruction fails if either required amount exceeds the user's
maximum.

The program then transfers both tokens into the vaults and mints LP tokens
using the config PDA as the mint authority.

### `withdraw`

```rust
withdraw(amount, min_x, min_y)
```

- `amount`: LP tokens to burn.
- `min_x`: minimum token X the user accepts.
- `min_y`: minimum token Y the user accepts.

The instruction calculates the proportional withdrawal amounts and rejects the
transaction if either amount is below the user's minimum. It then burns the
user's LP tokens and transfers token X and token Y from the vaults.

### `swap`

```rust
swap(is_x, amount_in, min_amount_out)
```

- `is_x = true`: swap token X for token Y.
- `is_x = false`: swap token Y for token X.
- `amount_in`: input token amount.
- `min_amount_out`: minimum output amount accepted by the user.

The curve calculates three values:

```text
deposit = full input amount
fee     = protocol fee
withdraw = output amount
```

The program performs two input transfers:

```text
pool deposit = deposit - fee
treasury     = fee
```

Then it transfers the output token from the opposite pool vault to the user.

## Fee flow

Suppose a user swaps token X for token Y:

```text
User X account ── pool deposit ──> vault_x
User X account ── fee ───────────> treasury_x
vault_y ─────── output Y ────────> user Y account
```

For a token-Y input, the same flow uses `vault_y` and `treasury_y` instead.

The current project creates and fills treasury accounts, but it does not yet
include a `collect_fees` instruction. Therefore, fees remain in the treasury
accounts until a future authorized withdrawal instruction is added.

## Testing

The tests use LiteSVM to run the compiled program in-process. The test setup:

1. Creates a LiteSVM instance.
2. Loads `target/deploy/amm_video.so`.
3. Creates token-X and token-Y mints with six decimals.
4. Derives the pool, LP mint, vault, and treasury addresses.
5. Builds raw Anchor instructions.
6. Sends transactions to LiteSVM.
7. Checks transaction success and the collected fee balance.

The current tests cover:

- Pool initialization.
- Initial liquidity deposit.
- LP withdrawal.
- Token swap.
- A `30 bps` swap fee being deposited into `treasury_x`.

## Local setup

Install the required tools:

- Rust and Cargo.
- Solana CLI.
- Anchor CLI `1.0.1`.
- Node.js and Yarn, if you need the JavaScript tooling.

Check the toolchain:

```bash
rustc --version
cargo --version
solana --version
anchor --version
```

Install Rust dependencies:

```bash
cargo fetch
```

## Build and test

Build the deployable program:

```bash
NO_DNA=1 anchor build --ignore-keys
```

The repository's configured program ID and local keypair may differ. The
`--ignore-keys` flag skips that keypair consistency check and builds using the
`declare_id!` value in `src/lib.rs`.

Run the Rust tests:

```bash
NO_DNA=1 cargo test -p amm-video --tests
```

Because the integration test loads `target/deploy/amm_video.so`, run the build
before the tests whenever the program source changes.

Format the Rust code:

```bash
cargo fmt --all
```

## Running with Anchor localnet

The provider is configured for localnet in `Anchor.toml`:

```toml
[provider]
cluster = "localnet"
```

To use a local validator manually:

```bash
solana-test-validator
```

In another terminal, build and deploy:

```bash
NO_DNA=1 anchor build --ignore-keys
NO_DNA=1 anchor deploy --provider.cluster localnet --provider.wallet ~/.config/solana/id.json
```

The automated project test command is also available through Anchor:

```bash
NO_DNA=1 anchor test --skip-local-validator
```

For the current LiteSVM tests, the direct Cargo command is the clearest option
because it uses the test harness already defined in `programs/amm-video/tests`.

# Asteria

[Asteria](https://asteria.txpipe.io/) is an on-chain competitive game on Cardano built by [TxPipe](https://txpipe.io/) to showcase what the eUTxO model can do. Players write automated bots that pilot ships across a 2D coordinate grid, competing to reach the center `(0, 0)` and mine the central **Asteria** prize.

This tx3 covers the full player-facing flow: spawning a ship, moving across the grid, gathering fuel and prize tokens from pellets, mining the central Asteria, and quitting the game.

## Overview

State lives in UTxOs at three Plutus validators:

- **Spacetime** — owns every ship UTxO. Enforces movement rules: a ship's coordinates can change by `(delta_x, delta_y)` only if the player burns Fuel equal to the Manhattan distance `|delta_x| + |delta_y|`, and only if the elapsed time in the transaction's validity window covers that distance at the configured speed limit.
- **Pellet** — owns pellet UTxOs scattered across the grid. Each pellet holds some Fuel and (optionally) sponsor prize tokens. A ship at the same coordinates as a pellet can take Fuel and/or claim its prize tokens.
- **Asteria** — owns the single central UTxO holding the prize pool and a ship counter. New ships are registered against this counter; the winning mine transaction pays out from this UTxO's ADA balance.

Two minting policies back the game tokens:

- **Shipyard** (`SpacetimePolicy`) mints/burns the **Ship** NFT (lives in the ship UTxO at Spacetime) and the **Pilot** token (kept in the player's wallet to prove control of the ship).
- **Fuel** (`PelletPolicy`) mints/burns the **Fuel** native token consumed by movement.

A player joins by minting a Ship + Pilot pair against the Asteria counter, ships up at a chosen `(pos_x, pos_y)` at least the minimum distance from origin, and then plays via `move_ship` / `gather_fuel` / `gather_token` until they either `mine_asteria` at `(0, 0)` or `quit_game`.

## Transactions

| Transaction | Description |
|---|---|
| `create_ship` | Mint a new Ship + Pilot, increment the Asteria ship counter, and spawn the ship at `(p_pos_x, p_pos_y)` with an initial fuel allocation |
| `move_ship` | Update a ship's position by `(p_delta_x, p_delta_y)` and burn `required_fuel` Fuel, within a validity window that satisfies the Spacetime speed limit |
| `gather_fuel` | At a pellet's coordinates, transfer Fuel from the pellet UTxO to the ship UTxO |
| `gather_token` | At a pellet's coordinates, transfer Fuel to the ship and a sponsor prize token to the player's wallet |
| `mine_asteria` | At `(0, 0)`, withdraw `mine_amount` ADA from the Asteria UTxO to the player, then burn the ship and its remaining fuel |
| `quit_game` | Burn the ship and its remaining fuel, recovering the locked min-UTxO ADA to the player |

## Important considerations

- **Pilot token gates every ship action.** Every transaction except `create_ship` takes the Pilot token as a fungible input from the player's wallet and returns it via `pilot_change` — losing the Pilot token means losing control of the ship.
- **Movement is validity-window bound.** `move_ship` requires both `since_slot` and `until_slot`. The Spacetime validator checks that the elapsed slots cover the Manhattan distance at the configured speed; an over-tight window will reject an otherwise legal move.
- **`last_move_timestamp` must be set by the caller.** `create_ship` and `move_ship` write `last_move_latest_time` into the new ship datum; the caller computes the timestamp (typically the validity-window end) before invoking.
- **Fuel cost equals Manhattan distance.** For `move_ship`, `required_fuel` must equal `|p_delta_x| + |p_delta_y|` — the parameter is passed explicitly because tx3 does not currently have an absolute-value operator.
- **Pellet references are passed by `UtxoRef`.** `gather_fuel` and `gather_token` take `pellet_ref: UtxoRef` rather than resolving the pellet by address, because there are many pellet UTxOs on the grid and the caller chooses one based on the ship's current coordinates.
- **Sponsor token policy is per-pellet.** `gather_token` takes `token_policy_hash`, `token_name`, and `token_amount` as parameters because different pellets hold different sponsor tokens (IAGON, SUNDAE, HOSKY, etc.); the caller reads them from the chosen pellet's value before invoking.
- **Mine payout is bounded by protocol policy.** `mine_amount` is the ADA the player withdraws from the Asteria UTxO; the Asteria validator enforces an upper bound (a fraction of the pool) on-chain.

## Caller preparation

Several values must be prepared off-chain before invoking transactions.

### `create_ship`

| Parameter | Source |
|---|---|
| `ship_name`, `pilot_name: Bytes` | Token names for the new Ship and Pilot; conventionally derived from the current `AsteriaDatum.ship_counter`. |
| `p_pos_x`, `p_pos_y: Int` | Spawn coordinates chosen by the player. Must satisfy the protocol's minimum-distance-from-origin rule. |
| `initial_fuel: Int` | Initial Fuel allocation, capped by the protocol's max-fuel parameter. |
| `ship_mint_lovelace_fee: Int` | Mint fee paid into the Asteria UTxO; read from protocol parameters. |
| `last_move_timestamp: Int` | Timestamp written into the new ship datum (typically the validity-window end). |

### `move_ship`

| Parameter | Source |
|---|---|
| `p_delta_x`, `p_delta_y: Int` | Movement delta chosen by the player. |
| `required_fuel: Int` | Equals `|p_delta_x| + |p_delta_y|`, computed off-chain. |
| `since_slot`, `until_slot: Int` | Validity window — its size must cover the Manhattan distance at the Spacetime speed limit. |
| `last_move_timestamp: Int` | Timestamp written into the updated ship datum (typically `until_slot`). |
| `ship_name`, `pilot_name: Bytes` | Names of the ship being moved and its pilot token. |

### `gather_fuel` / `gather_token`

| Parameter | Source |
|---|---|
| `pellet_ref: UtxoRef` | The chosen pellet UTxO at the ship's current coordinates. |
| `p_amount` / `fuel_amount: Int` | Fuel to transfer from the pellet, capped by the pellet's Fuel balance and the ship's remaining capacity. |
| `token_policy_hash`, `token_name`, `token_amount: Bytes`/`Int` (gather_token only) | Sponsor prize token held by the chosen pellet; read from its value. |
| `since_slot: Int` | Validity-window lower bound. |

### `mine_asteria` / `quit_game`

| Parameter | Source |
|---|---|
| `ship_fuel: Int` | Current Fuel balance on the ship UTxO (must be burned exactly). |
| `mine_amount: Int` (mine_asteria only) | ADA to withdraw from the Asteria UTxO; bounded by the on-chain payout cap. |
| `since_slot: Int` | Validity-window lower bound. |

## References

- **Homepage / app:** [asteria.txpipe.io](https://asteria.txpipe.io/)
- **Source:** [txpipe/asteria](https://github.com/txpipe/asteria)
- **Starter kit:** [txpipe/asteria-starter-kit](https://github.com/txpipe/asteria-starter-kit)

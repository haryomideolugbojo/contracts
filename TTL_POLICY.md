# Storage TTL Bump Policy

## Overview

This document defines the repository-wide storage TTL (Time-To-Live) bump policy for the decentralized healthcare system. It ensures consistent data retention across all smart contracts and prevents silent data expiry.

## Problem Statement

Previously, TTL extension was inconsistent across contracts:

- Some contracts (patient-registry, pacs-integration) aggressively bumped keys
- Most contracts (35+) relied on default Soroban TTL or didn't explicitly manage it
- This risked silent data expiry for critical healthcare records

## Solution: Retention Classes

We define three retention classes based on data criticality:

### 1. Critical Retention Class

**Used for:** Patient records, medical history, prescriptions, clinical trials, allergy records

- **Bump Amount:** 535,680 ledgers (~31 days at 5s/ledger)
- **Threshold:** 518,400 ledgers (~30 days)
- **Minimum TTL:** 535,680 ledgers
- **Policy:** Bump on every write and read operation

**Contracts using Critical:**

- patient-registry
- pacs-integration
- allergy-management
- health-records (recommended)
- prescription-management (recommended)
- clinical-trial (recommended)

### 2. Operational Retention Class

**Used for:** Temporary records, session data, intermediate states, audit logs

- **Bump Amount:** 120,960 ledgers (~7 days at 5s/ledger)
- **Threshold:** 60,480 ledgers (~3.5 days)
- **Minimum TTL:** 120,960 ledgers
- **Policy:** Bump on write operations; optional on reads

**Recommended for:**

- telemedicine (session data)
- medical-claims (temporary states)
- referral (intermediate states)

### 3. Ephemeral Retention Class

**Used for:** Counters, temporary caches, transient state

- **Bump Amount:** 17,280 ledgers (~1 day at 5s/ledger)
- **Threshold:** 8,640 ledgers (~12 hours)
- **Minimum TTL:** 17,280 ledgers
- **Policy:** Bump on write operations only

**Recommended for:**

- Instance storage counters
- Temporary caches
- Session tokens

## Implementation

### Centralized Configuration

All TTL constants are defined in `contracts/ttl-config/src/lib.rs`:

```rust
pub mod critical {
    pub const LEDGER_BUMP_AMOUNT: u32 = 535_680;
    pub const LEDGER_THRESHOLD: u32 = 518_400;
}

pub mod operational {
    pub const LEDGER_BUMP_AMOUNT: u32 = 120_960;
    pub const LEDGER_THRESHOLD: u32 = 60_480;
}

pub mod ephemeral {
    pub const LEDGER_BUMP_AMOUNT: u32 = 17_280;
    pub const LEDGER_THRESHOLD: u32 = 8_640;
}
```

### Helper Functions

The `ttl-config` crate provides helper functions for easy TTL management:

```rust
// Extend TTL for a key
extend_critical_ttl(env, &key);
extend_operational_ttl(env, &key);
extend_ephemeral_ttl(env, &key);

// Conditionally extend if key exists
extend_critical_ttl_if_exists(env, &key);
extend_operational_ttl_if_exists(env, &key);
extend_ephemeral_ttl_if_exists(env, &key);
```

### Usage Pattern

**On Write Operations:**

```rust
pub fn save_record(env: &Env, record: &Record) {
    let key = DataKey::Record(record.id);
    env.storage().persistent().set(&key, record);
    extend_critical_ttl(env, &key);  // Always bump on write
}
```

**On Read Operations (Critical Data):**

```rust
pub fn get_record(env: &Env, record_id: u64) -> Result<Record, Error> {
    let key = DataKey::Record(record_id);
    let result = env.storage().persistent().get(&key).ok_or(Error::NotFound);

    if result.is_ok() {
        extend_critical_ttl_if_exists(env, &key);  // Bump on successful read
    }

    result
}
```

## Migration Guide

### For Existing Contracts

1. **Add dependency** to `Cargo.toml`:

   ```toml
   [dependencies]
   ttl-config = { path = "../ttl-config" }
   ```

2. **Replace local constants** with imports:

   ```rust
   use ttl_config::critical::{LEDGER_BUMP_AMOUNT, LEDGER_THRESHOLD};
   ```

3. **Add TTL bumping** to storage functions:
   - Write operations: Always bump
   - Read operations: Bump if critical data

4. **Test** that TTL is extended:
   - Verify snapshots include TTL extension calls
   - Add tests for TTL bump behavior

### For New Contracts

1. Add `ttl-config` dependency
2. Import appropriate retention class
3. Implement TTL bumping in storage layer
4. Document retention class choice in contract README

## Testing

### TTL Bump Verification

Each contract should include tests verifying TTL bumping:

```rust
#[test]
fn test_record_ttl_bumped_on_write() {
    let env = Env::default();
    let contract = setup(&env);

    // Write a record
    contract.save_record(&record);

    // Verify TTL was extended (check snapshots)
    // TTL should be >= LEDGER_BUMP_AMOUNT
}

#[test]
fn test_record_ttl_bumped_on_read() {
    let env = Env::default();
    let contract = setup(&env);

    // Write and read a record
    contract.save_record(&record);
    let retrieved = contract.get_record(record.id);

    // Verify TTL was extended on read
}
```

### Snapshot Testing

Test snapshots capture TTL extension calls. Example from allergy-management:

```json
{
  "ledger": {...},
  "storage": {
    "persistent": [
      {
        "key": "Allergy(1)",
        "value": {...},
        "ttl_extended": true
      }
    ]
  }
}
```

## Monitoring & Maintenance

### Key Metrics

- **TTL Expiry Rate:** Monitor contracts for unexpected data loss
- **Bump Frequency:** Verify bumps occur at expected intervals
- **Storage Growth:** Track persistent storage size per contract

### Alerts

Set up alerts for:

- Records approaching TTL expiry without bumps
- Contracts with no TTL bump activity
- Unexpected storage deletions

## Manual TTL Extension

When the automated `Extend Contract TTLs` workflow fails, or you need to extend
TTLs outside the schedule, run `scripts/extend-ttls.sh` directly.

### 1. Prerequisites

- The [Stellar CLI](https://developers.stellar.org/docs/tools/cli/stellar-cli) (`stellar`) installed and on `PATH`.
- A `deployments/<network>.json` manifest containing the contract IDs to extend (e.g. `deployments/mainnet.json`).
- A Stellar CLI identity/secret with enough XLM to pay the extension fees for every contract in the manifest.

### 2. Identity / secret to use

- **Mainnet:** use the identity backing the `MAINNET_DEPLOYER_IDENTITY` GitHub Actions secret — the same
  identity the scheduled job authenticates with. Get the secret value from whoever holds repo/org secret
  access (repo admin or the deployment owner), then import it locally before running the script:

  ```bash
  stellar keys add mainnet-deployer --secret-key   # paste the MAINNET_DEPLOYER_IDENTITY secret value
  ```

  Never commit this secret or paste it into a shell history file that gets synced anywhere. Prefer an
  interactive prompt (as above) over passing `--secret-key` inline.

- **Testnet:** use the identity backing `TESTNET_DEPLOYER_IDENTITY`, imported the same way under a
  different local key name (e.g. `testnet-deployer`).

### 3. Run the script

```bash
# Dry run first — logs what would be extended without submitting any transactions.
./scripts/extend-ttls.sh \
  --network mainnet \
  --identity mainnet-deployer \
  --ledgers-to-extend 535680 \
  --dry-run

# Then the real run.
./scripts/extend-ttls.sh \
  --network mainnet \
  --identity mainnet-deployer \
  --ledgers-to-extend 535680
```

Key flags (see `./scripts/extend-ttls.sh --help` for the full list):

- `--network <name>` — `mainnet` or `testnet`.
- `--identity <name>` — the local CLI identity name from step 2 (not the raw secret).
- `--ledgers-to-extend <count>` — defaults to `535680` (~1 year at 5s/ledger); match the value the
  cron job normally uses unless you have a specific reason to diverge.
- `--dry-run` — logs the contracts that would be extended without calling `stellar contract extend`.

The script reads contract IDs out of `deployments/<network>.json` and calls `stellar contract extend`
for each one, printing an `Extended: N / Failed: N / Total: N` summary at the end and exiting non-zero
if any contract failed.

### 4. Verify success

- **Script output:** confirm the summary line reports `Failed: 0` and that every contract ID logged an
  `Extending TTL for: <id>` line without a following `WARNING: Failed to extend TTL for: <id>`.
- **`stellar contract extend` output:** the command itself reports the resulting expiration ledger for
  each entry it touches — capture the script's stdout (it's not redirected) and confirm the reported
  ledger is `~LEDGERS_TO_EXTEND` ledgers ahead of the ledger the transaction landed in.
- **Current TTL / expiry lookup (per contract, without extending anything):** query the contract
  instance's `liveUntilLedgerSeq` via [Stellar Laboratory](https://laboratory.stellar.org)'s "Ledger
  Entries" / contract-data explorer for the target network, using the contract's `C...` address — or
  call the network's Soroban RPC `getLedgerEntries` method directly with the contract instance's
  ledger key, and compare the returned `liveUntilLedgerSeq` against the current sequence from
  `getLatestLedger`:

  ```bash
  RPC_URL="https://mainnet.sorobanrpc.com"   # use the network's RPC endpoint

  curl -s -X POST "$RPC_URL" -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' | jq '.result.sequence'
  ```

  The contract is healthy if `liveUntilLedgerSeq` is comfortably above the current ledger sequence
  (well beyond `CRITICAL_THRESHOLD`, ~86,400 ledgers / ~1 day, from `scripts/extend-ttls.sh`). See the
  [Stellar CLI TTL cookbook](https://developers.stellar.org/docs/tools/cli/cookbook/extend-contract-instance)
  for how to encode a contract instance's ledger key for `getLedgerEntries`.

### 5. If extension still fails

- Re-run with `--dry-run` to confirm the manifest and contract IDs are being parsed correctly.
- Check the identity's XLM balance — insufficient balance is the most common cause of `stellar contract
  extend` failures.
- Confirm `deployments/<network>.json` is up to date and every listed contract ID is still deployed.
- If the failure persists, escalate in the deployment/automation-bug issue rather than retrying
  indefinitely — repeated manual extension without addressing the root cause just delays discovering why
  the cron job itself is broken.

## Compliance Checklist

- [ ] All critical healthcare data uses Critical retention class
- [ ] TTL bumping implemented on write paths
- [ ] TTL bumping implemented on read paths (critical data)
- [ ] Tests verify TTL extension behavior
- [ ] Documentation updated with retention class choice
- [ ] Snapshots include TTL extension verification
- [ ] No hardcoded TTL constants (use ttl-config)

## FAQ

**Q: Why bump on read operations?**
A: Critical healthcare data must never expire unexpectedly. Bumping on reads ensures active records stay fresh even if writes are infrequent.

**Q: Can I use different retention classes for different keys?**
A: Yes. Use Critical for patient records, Operational for temporary data, Ephemeral for counters.

**Q: What if a record isn't accessed for 31 days?**
A: It will expire. This is intentional for Operational/Ephemeral data. For Critical data, implement a background job to bump keys periodically.

**Q: How do I choose a retention class?**
A: Ask: "If this data expires, would it harm patient care?" If yes → Critical. If maybe → Operational. If no → Ephemeral.

## References

- [Soroban Storage Documentation](https://soroban.stellar.org/docs/learn/storing-data)
- [TTL Configuration Module](contracts/ttl-config/src/lib.rs)
- [Patient Registry Implementation](contracts/patient-registry/src/lib.rs)

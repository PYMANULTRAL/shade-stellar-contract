# Disaster Recovery and Business Continuity Plan

This runbook covers recovery from the loss of critical capabilities in the Shade Protocol: admin keys, the platform account key, RPC providers, indexer data, billing keeper infrastructure, and archived contract state. It complements the [Admin Operations and Incident Response Runbook](admin-runbook.md), which handles *active* incidents. This document handles *recovery of capability* once an incident has been contained.

Each scenario documents its impact, detection, recovery steps, and recovery time objective (RTO) and recovery point objective (RPO) where applicable. Scenarios that cannot be fully recovered from are listed in [Unrecoverable scenarios](#unrecoverable-scenarios) together with their preventive controls.

## Scope and definitions

| Term | Meaning |
|------|---------|
| RTO | Recovery Time Objective: maximum acceptable time from detection until the capability is restored. |
| RPO | Recovery Point Objective: maximum acceptable window of data or state that may need to be re-derived or replayed. |
| Admin key | The Stellar account authorized as `Admin` on the `shade` contract. |
| Multisig signer | An individual key participating in a multisig controlling the admin role or the platform account. |
| Platform account | The `PlatformAccount` contract (or its controlling key) that receives platform fees. |
| Indexer | Off-chain service that consumes contract events and materialises queryable state. |
| Billing keeper | Off-chain process that triggers recurring billing cycles and auto-withdrawals. |

## Scenario 1: Loss or compromise of the admin key or a multisig signer

**Impact.** The admin can pause the contract, change fees, manage tokens and merchants, assign roles, and upgrade WASM. Loss of the only admin key blocks all privileged operations. Compromise allows an attacker to do the same maliciously. A single lost multisig signer reduces redundancy; loss of enough signers to meet the threshold blocks administration entirely.

**Detection.**
- Admin signing attempts fail or a key is reported lost.
- Unexpected `upgrade`, `grant_role`, `set_fee`, or pause events appear in the event stream without a matching change ticket.
- Monitoring alerts on admin-address activity outside maintenance windows.
- A signer confirms device loss or theft.

**Recovery steps.**
1. Freeze: if compromise is suspected, pause the contract immediately using any remaining valid signer (see [Admin Runbook § Pause procedure](admin-runbook.md#pause-procedure)).
2. Inventory: list the admin address, all multisig signers, and the signing threshold. Identify which keys are lost, compromised, or still trusted.
3. Quorum check: confirm enough trusted signers remain to meet the threshold. If not, proceed to step 6.
4. Rotate: initiate a two-step admin transfer to a new key or new multisig (see [Admin Runbook § Role management](admin-runbook.md#admin-transfer) and [ADR 0004](../architecture/decisions/0004-two-step-admin-transfer.md)). Accept the transfer from the new address only after verifying it on a second channel.
5. Revoke: revoke roles held by lost or compromised addresses with `revoke_role`. Review pending fee proposals (`PendingFee`) and cancel any the attacker could have queued.
6. Quorum lost entirely: if the threshold can no longer be met, the contract cannot be administered on-chain. Recovery requires a pre-approved fallback (a cold-stored recovery key or a documented governance escalation) that was established in advance. Without it, the scenario is unrecoverable (see Unrecoverable scenarios).
7. Post-incident: audit all admin events since the earliest suspected compromise, re-verify fee and token configuration, and record the incident.

**RTO.** Pause within 30 minutes of detection. Restored administration within 24 hours when a quorum remains; not applicable when quorum is lost.

**RPO.** Not applicable to stored data; configuration changes made by an attacker must be reverted by reviewing events back to the suspected compromise time.

## Scenario 2: Loss of the platform account key

**Impact.** The platform account receives platform fees. Loss of its key means fee revenue can no longer be withdrawn and the operator cannot perform platform-level actions gated on that key. User payments and merchant balances are not directly affected.

**Detection.**
- Withdrawal attempts from the platform account fail authorization.
- Fee accrual is visible on-chain but cannot be swept.
- The key holder reports loss.

**Recovery steps.**
1. Confirm the loss and check whether a backup of the key (encrypted seed, hardware wallet recovery phrase) exists in the designated secure store.
2. If a backup exists, restore the key to a new signing environment and verify the derived address matches the platform account address before any transaction.
3. If no backup exists and the platform account is a plain key account, fee balances are not recoverable through that key. Document the lost balance amount using fee events.
4. If the platform account is controlled through the contract admin (for example, the admin can reassign the fee recipient), use the admin flow to point fee accrual to a new platform account and update documentation and integrator configuration.
5. Verify the new recipient by sending a small test fee and confirming receipt.

**RTO.** Fee sweeps resumed within 48 hours where a backup or admin reassignment exists.

**RPO.** Fee events since the last successful sweep define the amount at risk; they remain auditable from the event stream even if not withdrawable.

## Scenario 3: RPC provider outage

**Impact.** Clients, the indexer, and keepers cannot read state or submit transactions. Contract state and funds are unaffected. Users see failed reads and submissions.

**Detection.**
- Health checks against the configured RPC endpoint fail or exceed latency thresholds.
- Indexer and keeper logs show repeated connection or timeout errors.
- Provider status page reports an incident.

**Recovery steps.**
1. Switch the RPC endpoint in service configuration to the secondary provider. Maintain at least one alternate provider configured in advance for every environment.
2. Restart the indexer and billing keeper against the new endpoint and confirm they resume from their last processed ledger.
3. Verify transaction submission with a read-only call and a no-op test transaction on testnet or mainnet as appropriate.
4. If no secondary provider is available, operate read-only using a self-hosted Stellar RPC node if one is maintained; otherwise, pause non-essential submissions and communicate status to integrators.
5. Once the primary recovers, switch back during a low-traffic window and confirm stable operation for one hour.

**RTO.** Failover to secondary provider within 30 minutes.

**RPO.** None for on-chain state. Off-chain services resume from their last checkpoint; no ledger events are lost because the chain retains them.

## Scenario 4: Indexer database loss and full rebuild

**Impact.** Queryable history and analytics (invoices, payments, merchant activity, token analytics) become unavailable. On-chain state and funds are unaffected because the indexer is derived data.

**Detection.**
- Database connection failures or corruption errors.
- Queries return empty or inconsistent results compared with on-chain reads.
- Checkpoint table is missing or regressed.

**Recovery steps.**
1. Stop the indexer to prevent partial writes against a damaged database.
2. Preserve the damaged database snapshot for forensic review if the cause is unknown.
3. Provision a new empty database with the schema version matching the deployed indexer release.
4. Restore the most recent logical backup if available and it is within acceptable staleness; otherwise proceed to a full rebuild.
5. Full rebuild: start the indexer from the contract deployment ledger (or from the earliest ledger recorded in the deployment file) and replay all contract events in order. Events are the source of truth; no manual data entry is involved.
6. Verify rebuilt state by comparing invoice counts, merchant counts, and total payment volumes against values read directly from the contracts at a recent ledger.
7. Switch API traffic to the rebuilt database only after verification passes.

**RTO.** Restore from backup within 4 hours. Full event replay from deployment: estimated at under 24 hours for current volume, to be re-measured during each scheduled test and recorded in the test log.

**RPO.** Zero for on-chain-derived data when replayed from events; the indexer state at the time of loss is fully reconstructible. Data served from a logical backup may lag by the backup interval (target: 1 hour).

## Scenario 5: Billing keeper outage and catching up on missed cycles

**Impact.** Recurring billing cycles and scheduled auto-withdrawals are not triggered while the keeper is down. Merchants and subscribers experience delayed billing. Funds already held remain safe.

**Detection.**
- Keeper heartbeat or last-run timestamp exceeds the expected cycle interval.
- Cycles that should have executed show no on-chain execution events.
- Alerting on keeper process exit or missed-cycle counter.

**Recovery steps.**
1. Restart the keeper from its service manager. Confirm it authenticates and connects to the RPC endpoint.
2. Determine the set of missed cycles by comparing each subscription's next-due timestamp against the current ledger time.
3. Run the catch-up procedure in chronological order per subscription, executing each missed cycle once. The contract enforces interval boundaries, so duplicate execution of an already-processed cycle must fail and be treated as success.
4. Process auto-withdrawals only after billing cycles for the same merchant are reconciled, to avoid withdrawing before charges are settled.
5. Rate-limit catch-up submissions to avoid saturating the RPC provider and to respect Soroban resource limits; batch where supported.
6. Notify affected merchants of any delayed cycles and the actual execution times.
7. Record the number of missed and recovered cycles in the incident record.

**RTO.** Keeper running within 1 hour of detection. Full catch-up completed within 24 hours for outages up to 72 hours.

**RPO.** Cycles missed during the outage are recovered provided the subscription has not been cancelled and the payer retains sufficient balance. Cycles that cannot be funded are recorded as failed and are not retried automatically.

## Scenario 6: Archived contract instance or critical entries

**Impact.** Soroban archives ledger entries whose rent has expired. If a contract instance or a critical persistent entry (for example, balances, invoices, or configuration) is archived, calls touching it fail until it is restored. Funds are at risk of being unreachable until restoration.

**Detection.**
- Transactions fail with archived-entry errors.
- Monitoring of entry TTLs shows entries approaching expiry without a scheduled extension.
- Read calls return missing-entry errors for entries that are known to exist.

**Recovery steps.**
1. Identify the archived keys from the error response or from the TTL monitor.
2. Restore the entries using the restore-footprint operation with an account that has sufficient XLM to pay rent. Restoration is permissionless for the entry itself, so any funded operator account may perform it.
3. For the contract instance and code, extend TTL proactively where it is close to expiry.
4. Verify restoration with reads of each restored key and a test invocation of the affected function.
5. Review the TTL extension schedule and the monitor thresholds so the entry does not lapse again.

**RTO.** Restoration within 1 hour once detected and a funded operator account is available.

**RPO.** None if restoration occurs before permanent deletion. Entries that are archived and never restored are, on Soroban, retained in archive state and can still be restored; loss of the ability to restore at all would require reconstruction from events where the data is event-derivable.

## Unrecoverable scenarios

The following scenarios cannot be fully recovered from with the controls described above. Each lists the preventive controls that reduce the probability or impact.

| Scenario | Why it is not recoverable | Preventive controls |
|----------|---------------------------|---------------------|
| Admin quorum permanently lost with no fallback key | The contract's admin functions can only be exercised by the admin; no on-chain path restores control. | Maintain a threshold that tolerates the loss of at least one signer; store a sealed recovery key in geographically separate custody; rehearse signer rotation annually. |
| Platform account key lost with no backup and no admin reassignment | Funds controlled by a lost key cannot be moved. | Require encrypted backup of the platform key at creation; verify backup restores on a test device; prefer a contract-controlled platform account that the admin can repoint. |
| Permanent deletion of contract state after TTL lapse beyond restoration | If entries are not event-derivable and their archive is lost, the data cannot be reconstructed. | Monitor TTLs with alerts well before expiry; extend TTL on a schedule; ensure all state-changing events are retained by independent indexers. |
| Loss of on-chain funds through a protocol-level bug or malicious token | Funds moved by a contract vulnerability are not reversible by the protocol. | Pre-audit checklist and reentrancy guards (see [Threat model](../security/threat-model.md)); whitelist only audited tokens; keep fee and merchant balances on bounded exposure. |
| Full loss of the event history from the network | Indexer rebuild depends on the chain retaining events; if the network itself fails irrecoverably, the rebuild source is gone. | Maintain independent archives of the event stream from at least two indexers in separate regions; keep deployment records and ledger hashes. |

## Backup and preventive controls

- **Keys.** Admin and multisig keys are generated on hardware devices where possible, with recovery material held in sealed, geographically separate locations. Signer list and threshold are recorded in the admin runbook.
- **Providers.** At least two RPC providers are configured per environment with automated health checks.
- **Indexer.** Nightly logical backups are taken with a retention of 30 days; the event replay procedure is documented and tested.
- **Keeper.** Keeper runs under a supervised service with restart-on-failure and missed-cycle alerting.
- **Storage.** TTL extension runs on a schedule with alerting at a defined threshold before expiry.

## Testing schedule

| Activity | Frequency | Owner | Evidence recorded |
|----------|-----------|-------|-------------------|
| Tabletop review of all six scenarios | Quarterly | Protocol operations lead | Meeting notes and action items |
| Admin signer rotation rehearsal on testnet | Semi-annually | Admin signers | Transaction hashes and timing |
| RPC failover drill | Quarterly | Infrastructure on-call | Failover duration and any gaps |
| Indexer restore from backup | Quarterly | Indexer maintainer | Restore time and verification diff |
| Indexer full rebuild from events on testnet | Annually | Indexer maintainer | Replay duration and verification result |
| Keeper catch-up simulation | Semi-annually | Keeper maintainer | Missed-cycle count and recovery time |
| Archived entry restoration drill on testnet | Annually | Contracts engineer | Restore transaction and read-back verification |
| Platform key backup restore test | Annually | Platform key holder | Confirmation of derived address match |

Results of each test are recorded in the operations log and any gap against the RTO or RPO triggers a remediation issue. The plan itself is reviewed after every test and after every real incident.

← [Back to operations](README.md)

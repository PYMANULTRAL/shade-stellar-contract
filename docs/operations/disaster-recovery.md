# Disaster Recovery and Business Continuity Plan

This runbook covers recovery from the loss of critical capabilities in the Shade Protocol: admin keys, the platform account key, RPC providers, indexer data, billing keeper infrastructure, and archived contract state. It complements the [Admin Operations and Incident Response Runbook](admin-runbook.md), which handles *active* incidents. This document handles *recovery of capability* once an incident has been contained.

Each scenario documents its impact, detection, recovery steps, and recovery time objective (RTO) and recovery point objective (RPO) where applicable. Scenarios that cannot be fully recovered from are listed in [Unrecoverable scenarios](#unrecoverable-scenarios) together with their preventive controls. A schedule for exercising this plan is defined in [Testing schedule](#testing-schedule).

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

**RPO.** None if restoration occurs before permanent deletion. Entries that are archived and never restored remain recoverable only while the archive state is retained by the network; the plan treats loss of that retention as unrecoverable (see below).

## Unrecoverable scenarios

The following cases cannot be fully recovered from by the procedures above. Each lists the preventive controls that reduce the probability or impact.

| Scenario | Why it is not recoverable | Preventive controls |
|----------|---------------------------|---------------------|
| Admin quorum lost (Scenario 1, step 6) with no pre-approved fallback | No on-chain actor can authorize privileged changes, and contract upgrades are impossible. | Keep a cold-stored recovery key or governance escalation path documented and tested; set the multisig threshold so that losing one signer does not break quorum; keep signers on independent hardware in separate locations. |
| Platform account key lost with no backup and no admin reassignment path (Scenario 2, step 3) | Funds controlled solely by a lost key cannot be moved. | Maintain an encrypted backup of the platform key in a designated secure store; verify the backup restores the expected address at each scheduled test; prefer a fee recipient that the admin can reassign. |
| Admin key compromise that has already executed irreversible actions (for example, a malicious upgrade or drained balances) | Chain state changes cannot be rolled back. | Two-step admin transfer and timelocked sensitive actions; pause capability held by a separate role; monitoring and alerting on admin-address activity. |
| Archived entries never restored and archive retention lost | Soroban state that is permanently evicted cannot be rebuilt from the ledger alone if archive data is no longer available from any provider. | Proactive TTL extension on all critical entries; TTL monitor with alert thresholds well before expiry; restoration tested on schedule; at least one archive-capable RPC source configured. |
| Subscription cycles whose payer balance is insufficient during catch-up (Scenario 5, RPO) | The missed payment cannot be collected after the fact. | Balance-based alerts to payers and merchants before each cycle; clear cycle status reporting; merchant notification of failed cycles. |
| Indexer data lost with no event replay possible because the chain history source is unavailable | Derived data cannot be rebuilt without the event source. | Keep at least two independent RPC or history providers; retain periodic logical backups of the indexer; archive contract events to durable storage. |

## Testing schedule

The plan is exercised on the following schedule. Each exercise is recorded in a test log with the date, scenario, measured RTO and RPO, deviations from this document, and follow-up actions.

| Frequency | Exercise |
|-----------|----------|
| Quarterly | Tabletop review of Scenarios 1, 2, and the unrecoverable list; confirm signer roster, threshold, and recovery contacts are current. |
| Quarterly | Failover drill for RPC provider (Scenario 3) against the secondary provider on testnet. |
| Semi-annually | Indexer restore from logical backup and full event replay on a staging database (Scenario 4); re-measure replay duration. |
| Semi-annually | Billing keeper outage simulation and catch-up on testnet (Scenario 5). |
| Semi-annually | Restore an archived entry on testnet and verify the TTL monitor alerts (Scenario 6). |
| Annually | Full admin key rotation drill on testnet, including a simulated lost-signer scenario (Scenario 1). |
| Annually | Platform key backup restore verification (Scenario 2). |

After any real incident or any change to the admin set, signers, RPC providers, or indexer schema, the relevant exercise is re-run within 30 days.
# Operations

Runbooks for protocol operators, written from the [runbook template](../contributing/templates/runbook-template.md).

- [Deployment Runbook](deployment.md) — deploying `shade` and uploading the `account` WASM to local, testnet, and mainnet.
- [Contract Upgrade Runbook](upgrade-runbook.md) — pre-flight, execution, verification, and rollback for upgrading a deployed `shade` contract.
- [Admin Operations and Incident Response Runbook](admin-runbook.md) — routine admin operations (fees, tokens, merchants, roles, platform account) and incident response.
- [Disaster Recovery and Business Continuity Plan](disaster-recovery.md) — recovery from loss of admin or platform keys, RPC provider outages, indexer data loss, billing keeper outages, and archived contract state, including recovery objectives, unrecoverable scenarios, and the testing schedule.
- [Releases Runbook](releases.md) — versioning, tagging, building reproducible WASMs, recording hashes, writing release notes, publishing a GitHub Release, and the criteria for a breaking change.
- Rotating the admin key (two-step transfer) — see [Admin Runbook § Role management](admin-runbook.md#admin-transfer).
- Pausing and unpausing the Shade contract — see [Admin Runbook § Pause procedure](admin-runbook.md#pause-procedure) and [Admin Runbook § Unpause criteria](admin-runbook.md#unpause-criteria).
- Proposing and voting on a contract upgrade — see [Upgrade Runbook § Execution](upgrade-runbook.md#execution).
- Recovering from a lost or compromised admin key or multisig signer — see [Disaster Recovery § Admin key and multisig signer](disaster-recovery.md#admin-key-and-multisig-signer).

← [Back to documentation home](../README.md)

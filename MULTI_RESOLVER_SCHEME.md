# Multi-resolver scheme

TrustLink accepts one `resolver` address per escrow. A multi-resolver service
can sit behind that address and enforce an M-of-N vote, primary/backup
fallback, or both. See [docs/resolvers.md](docs/resolvers.md) for the scheme
selection, deadline gating, and deadlock risks.

The resolver service must preserve TrustLink's dispute deadline, appeal window,
and `PendingFinalization` flow. The escrow contract does not infer a backup
address from the resolver field.

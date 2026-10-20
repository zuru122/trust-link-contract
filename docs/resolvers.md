# Resolver schemes

TrustLink stores one resolver address per escrow. The resolver is consulted
only for disputes; normal delivery confirmation and time-based release do not
require resolver approval. Choose the resolver account with the same care as
the escrow parties because a compromised resolver can decide a disputed
escrow's payout.

## Single resolver

The simplest scheme is one trusted address. `resolve_dispute` authenticates
that address and selects either `Release` (pay the payees) or `Refund` (pay the
buyer). After the resolver decision, the escrow enters `PendingFinalization`
while the appeal window is open; `finalize_dispute` completes the payout after
the window, unless a permitted appeal reopens the dispute.

## Multi resolver: M-of-N vote

An M-of-N resolver is normally a separate contract or account abstraction that
collects votes from N members and exposes one resolver address to TrustLink.
TrustLink sees only that address, so the vote threshold and membership rules
must be enforced by the resolver layer before it signs the on-chain decision.
Document and monitor membership changes, quorum, replacement, and signer
authentication outside the escrow contract.

## Fallback resolver and deadline gating

A fallback design can use a primary resolver and a backup resolver. The
fallback layer should expose the primary while it is healthy and allow the
backup only after a configured deadline. The on-chain record contains the
resolver address and dispute deadline; it does not automatically switch
between two arbitrary addresses. A fallback contract must therefore enforce
its own `can_resolve_now`/deadline policy before invoking the escrow.

Fallbacks must not bypass the escrow's dispute deadline, appeal window, or
`PendingFinalization` checks. A resolver decision is not final until the appeal
window has elapsed and `finalize_dispute` is called.

## Deadlock risk

Resolver schemes can deadlock when the primary is unavailable, the backup is
not authorized, fewer than M members remain, or a vote never reaches quorum.
Mitigate this with an explicit deadline, a tested backup path, bounded signer
rotation, and monitoring for disputes approaching their deadline. Without a
live authorized path, funds can remain locked in `Disputed` or
`PendingFinalization` even though the escrow contract itself is functioning.

See [the multi-resolver scheme](../MULTI_RESOLVER_SCHEME.md),
[the architecture overview](../ARCHITECTURE.md), and the
[state-machine specification](state-machine.md).

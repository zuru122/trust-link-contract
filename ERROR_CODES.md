# Contract error codes

`ContractError` is the stable on-chain error ABI. Numeric values must not be
renumbered. The authoritative definitions are in
[`contracts/escrow/src/errors.rs`](contracts/escrow/src/errors.rs).

| Code | Name |
|---:|---|
| 1 | InvalidAmount |
| 2 | InsufficientBalance |
| 3 | EscrowNotFound |
| 4 | InvalidState |
| 5 | NotAuthorized |
| 6 | AlreadyInitialized |
| 7 | FeeExceedsMax |
| 8 | EscrowHasNoBuyer |
| 9 | ShippingWindowNotElapsed |
| 10 | InvalidEvidenceHash |
| 11 | DisputeNotFound |
| 12 | ArithmeticError |
| 13 | DeliveryBeforeDisputeWindow |
| 14 | ContractPaused |
| 15 | ArithmeticOverflow |
| 16 | InvalidStateTransition |
| 17 | InputTooLong |
| 18 | InvalidAddress |
| 19 | SameAddress |
| 20 | AmountExceedsMaximum |
| 21 | InvalidTrackingId |
| 22 | DeliveryNotRecorded |
| 23 | ConflictingRoles |
| 24 | DisputeWindowClosed |
| 25 | TokenNotAllowed |
| 26 | NotPendingFinalization |
| 27 | AppealWindowActive |
| 28 | PlatformFeeExceedsMax |
| 29 | AmountBelowMinimum |

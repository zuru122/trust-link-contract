# Integration guide

The contract's notes-aware create entrypoint takes nine arguments after the
contract environment. `payees` is a Soroban `Vec<Payee>`; use one payee for a
single seller or several entries whose `bps` values total 10,000.

```ts
import { EscrowClient, createFreighterTransport } from "trustlink-escrow-bindings";

const transport = await createFreighterTransport({
  contractId: "C...YOUR_CONTRACT_ADDRESS",
  networkPassphrase: "Test SDF Network ; September 2015",
  rpcUrl: "https://soroban-testnet.stellar.org",
});
const client = new EscrowClient(transport);

const escrowId = await client.create_escrow(
  [{ address: "G...SELLER", bps: 10_000 }], // seller_or_payees
  "G...BUYER",                              // buyer, or null
  "G...RESOLVER",
  "C...TOKEN",
  1_000_000n,
  100,                                      // fee_bps
  0,                                        // resolver_fee_bps
  86_400n,                                  // shipping_window
  null,                                     // notes; use "" for an empty note
);
```

`buyer` is optional at creation. Passing `null` leaves it unset until funding.
Use the bundled `createFreighterTransport` and `EscrowClient` shown above so
the example matches the published bindings. The generated client must expose
the notes-aware method as `create_escrow_with_notes`; the legacy
`create_escrow` entrypoint accepts the same first eight arguments and stores no
notes.

After funding, the normal completion guards are:

- `confirm_delivery`: buyer authorization, `Shipped` state, and the dispute
  deadline has closed.
- `auto_release`: no active dispute, dispute deadline closed, and either the
  shipping window or the recorded-delivery release window has elapsed.

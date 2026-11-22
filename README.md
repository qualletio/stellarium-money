# stellarium-money

The specification of `sh.stellarium.money`, the official Stellarium resources for moving money from contracts.

> **Status: design.** Nothing here is built yet. `Account` needs the operator's money staging, built-in `Account` support in stellarium-ts, and `payable` contracts in RequestScript 1.8.0. The full design is in [docs/money.md](https://github.com/qualletio/stellarium-ts/blob/main/docs/money.md) in stellarium-ts.

## Where it runs

There's no money node to deploy. stellarium-ts serves `sh.stellarium.money` on every node as a built-in resource:

- **Writes** (`earmark`, `pay`, `tryPay`, `refund`) are staged on the operator by the node that hosts the calling contract. The operator accepts them only from that contract version's registered host.
- **Reads** (`balance`, `fundingOf`, `canReceive`) are answered by the operator and signed, so the operator can check them when it replays the run.
- **Nodes never learn `sh.stellarium.money` from peers,** so no peer can answer contracts with made-up balances.

This repository holds the resources' interface and the conformance tests that stellarium-ts runs against its implementation.

## Resources

All resources are found under the `sh.stellarium.money` path.

### Account

The balance of the contract that calls it. Every contract version has one, kept by the operator. Contracts receive money when callers attach it to a call to one of their `pub payable fn`s, naming who it may be paid to, and spend it with `Account`:

```
const job: com.example.Job@1
job.fund{amount: 20000000, payees: ["org_courier"]}(...)
```

- **Only contracts can call `Account`,** and only for their own balance. A request script's calls fail.
- **The functions are free.** The operator takes the platform fee from `pay`, rounded up.
- **Every amount belongs to its funder** (`payment.from`), with the payees the funder named, optionally under a key the contract chooses. Every outflow names whose funding it spends.
- **Results are `{ value, error }`,** like every resource call. Read them with `.value`.
- **A failed call fails the run.** If any `Account` call returns an error, the run fails when it ends, even if the contract ignored the error. `tryPay` returning `false` isn't an error.
- **Nothing moves until the whole run succeeds.** Changes are staged, and apply with the contract's state when the run that started everything succeeds. A failed run moves no money and saves no state.
- **Funding can change behind a contract's back.** The time limit (14 days by default, at most 30) and card disputes return money to its funders without running the contract. Read `fundingOf` before paying out, rather than trusting your own records.

#### balance

`balance(): int64`

Returns the contract's balance in micro-USD, including what the current run has received and spent so far. The balance is the sum of every funder's unspent funding.

#### earmark

`earmark(key: string)`

Labels the money attached to the current call with a key, such as a job id, so it can be spent separately from the same funder's other money. Call it at most once per call, and only when money was attached.

#### fundingOf

`fundingOf(org: string, key: string): int64`

Returns how much of the balance is one organization's unspent funding under a key, including the current run's changes. A `null` key means money that was never earmarked.

#### canReceive

`canReceive(org: string): boolean`

Whether an organization can be paid now: it has completed Stripe Connect onboarding, so payouts are enabled, and it isn't frozen or banned. Check it before accepting money for a payee.

#### pay

`pay(to: string, amount: int64, from: string, key: string): int64`

Pays an organization out of one funder's money (`from`) under a key. The amount, less the platform fee, is paid out to it through Stripe Connect. Returns that funding after the payment. Fails if:

- the amount isn't positive;
- that funding is too low;
- `to` isn't one of the payees the funder named;
- `to` can't receive payouts.

#### tryPay

`tryPay(to: string, amount: int64, from: string, key: string): boolean`

Like `pay`, except that when `to` can't receive payouts, it pays nothing and returns `false`, and the run carries on. Use it when a contract pays several organizations, so one that loses its payouts can't block the others. The other failures still fail the run.

#### refund

`refund(to: string, amount: int64, key: string): int64`

Returns an organization's unspent funding under a key, with no fee, one step back to whoever sent it: the organization's prepaid balance if a script sent the money, or the balance of the contract that sent it. No code runs on the way back, and payees don't apply. Fails if the amount isn't positive or is more than that funding. Returns that funding after the refund.

Keys are optional and default to `null`.

## Example

A buyer pays for a dataset, split between its author and its host. The buyer names both as payees: `split.split{amount: 1000000, payees: ["org_author", "org_data"]}()`.

```
contract com.example.Split {
    const account: sh.stellarium.money.Account

    pub payable fn split(): int64 {
        const author = payment.amount * 70 / 100
        account.pay(to: "org_author", amount: author, from: payment.from)
        account.pay(to: "org_data", amount: payment.amount - author, from: payment.from)

        return payment.amount
    }
}
```

- Both payments settle together, or neither does.
- Integer division truncates, so `org_data` gets the remainder.
- If either organization can't receive payouts, every call fails. To keep paying the author regardless, use `tryPay` for each payee and refund what couldn't be paid.

See [Payment on delivery](https://github.com/qualletio/stellarium-ts/blob/main/docs/money.md#payment-on-delivery) in the design for a contract that holds a buyer's money until the work is confirmed.

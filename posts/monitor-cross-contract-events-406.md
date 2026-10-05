---
layout: default
title: "Monitor: do events from a monitored contract match when another contract made the call? (issue #406)"
---

# Monitor: do events from a monitored contract match when another contract made the call?

Yes for events, no for functions. Verified in the OpenZeppelin Monitor 1.6.0 source, `src/services/filter/filters/evm/filter.rs`. Upstream question: [openzeppelin-monitor #406](https://github.com/OpenZeppelin/openzeppelin-monitor/issues/406).

## The situation

You monitor contract A. A user calls contract B, and B calls A internally. The transaction's `to` is B. Does your monitor on A fire?

## Events: yes

`filter_block` fetches all logs for the block with no address filter, groups them by transaction hash, and evaluates every transaction in the block against every monitor. `find_matching_events_for_transaction` walks the transaction's logs, keeps the ones whose `log.address` equals a monitored address, and adds that emitting address to the transaction's involved addresses. It never looks at `transaction.to`. A transaction counts as involving a monitored address if that address is the sender, the recipient, or the emitter of a matched log.

So if A emits an event during the internal call, a monitor on A with an event condition for that event matches, provided A's `contract_spec` (the ABI array in the monitor's `addresses` entry) contains the event so the log can be decoded.

## Functions: no

`find_matching_functions_for_transaction` decodes `transaction.input` against the ABI of the address in `transaction.to`. An internal call from B into A is not in the transaction input, so a function condition on A does not see it. A `fallback` call has no selector to decode either. Monitor 1.6.0 does no call tracing.

## What to do

For cross-contract calls, emit an event from the monitored contract and match the event. Use function conditions only for direct calls to the monitored address.

A matching monitor fragment:

```json
"addresses": [{"address": "0xA...", "contract_spec": [ ...ABI including the event... ]}],
"match_conditions": {
  "events": [{"signature": "Rebalanced(address,uint256)", "expression": "amount > 0"}]
}
```

Signatures use exact Solidity types with no parameter names or spaces.

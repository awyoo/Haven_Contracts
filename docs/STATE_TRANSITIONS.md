# Haven Registry State Transitions

This document summarizes the current Haven Registry contract behavior for
contributors who need to reason about device lifecycle changes before editing
the contract.

The raw IMEI is never stored on-chain. Every device transition is keyed by a
`BytesN<32>` SHA-256 hash of the IMEI.

## Device State

`DeviceState` is stored at `DataKey::Device(hashed_imei)` and currently tracks:

- `owner`: the registered device owner.
- `hashed_imei`: the SHA-256 IMEI hash used as the device identifier.
- `device_model`: the human-readable model string.
- `is_stolen`: whether the owner has reported the device stolen.
- `registered_at`: the ledger sequence when the device was registered.
- `recovery_contact`: contact details shown to a finder while stolen.
- `insurer`: the insurer address once an insurance claim has been filed.

`DataKey::Bounty(hashed_imei)` stores the currently promised recovery bounty.
`DataKey::DeviceCount` tracks the registration count.

## Transition Summary

| Transition | Function | Required caller | From | To | Storage effects |
|---|---|---|---|---|---|
| Register device | `register_device` | `owner` | Unregistered | Registered, not stolen | Creates `DeviceState`, increments `DeviceCount`, emits `DeviceRegistered` |
| Report stolen | `report_stolen` | Current device `owner` | Registered, not stolen | Registered, stolen | Sets `is_stolen = true`, stores `recovery_contact`, stores bounty amount |
| Confirm recovery | `confirm_recovery` | Current device `owner` | Registered, stolen | Registered, not stolen | Sets `is_stolen = false`, clears `recovery_contact`, removes bounty record |
| File insurance claim | `file_insurance_claim` | Current device `owner` | Registered, not previously claimed | Claimed by insurer | Sets `insurer = Some(insurer)` |

## Valid Paths

### Registration

`register_device(owner, hashed_imei, device_model)` creates the first
`DeviceState` for a hashed IMEI. The same hash cannot be registered twice.

After registration:

- `is_stolen` is `false`.
- `recovery_contact` is empty.
- `insurer` is `None`.
- `DeviceRegistered` is emitted with topics `("dev_reg", "register")`.

Covered by:

- `test_register_device`
- `test_register_device_emits_event`
- `test_register_device_duplicate`
- `test_get_device`

### Stolen Report

`report_stolen(owner, hashed_imei, bounty_amount, recovery_contact)` moves a
registered device into the stolen state.

Current behavior:

- Only the registered owner can call it.
- Calling it for an already stolen device panics with
  `"device already reported as stolen"`.
- The promised bounty amount is stored, but token escrow transfer is still a
  TODO in `killswitch.rs`.
- The `DeviceStolen` event is planned but not yet implemented.

Covered by:

- `test_report_stolen`
- `test_report_stolen_twice`

### Recovery

`confirm_recovery(owner, hashed_imei, finder)` moves a stolen device back to the
not-stolen state.

Current behavior:

- Only the registered owner can call it.
- Calling it for a device that is not stolen panics with
  `"device is not reported as stolen"`.
- The bounty record is removed.
- Token payout to `finder` is still a TODO in `recovery.rs`.
- The `DeviceRecovered` event is planned but not yet implemented.

Covered by:

- `test_confirm_recovery`
- `test_recover_not_stolen`

### Insurance Claim

`file_insurance_claim(owner, hashed_imei, insurer)` records the insurer as the
authority for the device after a claim.

Current behavior:

- Only the registered owner can initiate the claim.
- A device can only be claimed once.
- A second claim panics with `"insurance claim already filed"`.
- Claim metadata, insurer salvage actions, bounty handling, cooldowns, and the
  `InsuranceClaimed` event are planned but not yet implemented.

Covered by:

- `test_file_insurance_claim`
- `test_file_insurance_claim_twice`

## Invariants

- A hashed IMEI is unique. Once `DataKey::Device(hashed_imei)` exists,
  registering that hash again must fail.
- Owner authorization is required for owner-only flows:
  `register_device`, `report_stolen`, `confirm_recovery`, and
  `file_insurance_claim`.
- `report_stolen` cannot be called twice without a successful recovery in
  between.
- `confirm_recovery` only applies to devices currently marked stolen.
- `file_insurance_claim` is terminal for the current claim model because the
  contract has no unclaim or insurer transfer function yet.
- The only event emitted by the current implementation is `DeviceRegistered`.
  Planned event names are documented in `docs/EVENTS.md`.

## Related Files

- `contracts/haven_registry/src/device.rs`
- `contracts/haven_registry/src/killswitch.rs`
- `contracts/haven_registry/src/recovery.rs`
- `contracts/haven_registry/src/insurance.rs`
- `contracts/haven_registry/src/test.rs`
- `docs/EVENTS.md`

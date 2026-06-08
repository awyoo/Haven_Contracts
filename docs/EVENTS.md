# Haven Registry Events

This document describes the events emitted, or explicitly planned, by the
Haven Registry smart contract for off-chain indexers, analytics jobs, and
frontend notification services.

The raw IMEI is never included in an event. Indexers should use
`hashed_imei: BytesN<32>` as the stable join key and call
`get_device(hashed_imei)` when they need the full `DeviceState`.

## Event Catalog

| Event | Status | Function | Topics | Payload |
|-------|--------|----------|--------|---------|
| `DeviceRegistered` | Implemented | `register_device()` | `(dev_reg, register)` | `(hashed_imei: BytesN<32>, owner: Address, device_model: String)` |
| `DeviceStolen` | Planned | `report_stolen()` | `(stolen)` | `(hashed_imei: BytesN<32>, bounty_amount: i128)` |
| `DeviceRecovered` | Planned | `confirm_recovery()` | `(recovered)` | `(hashed_imei: BytesN<32>, finder: Address)` |
| `InsuranceClaimed` | Planned | `file_insurance_claim()` | `(insured)` | `(hashed_imei: BytesN<32>, insurer: Address)` |

## DeviceRegistered

Emitted when a new device is successfully registered on-chain.

**Source:** `contracts/haven_registry/src/device.rs::register_device`

**Topics:**
- `dev_reg` (`symbol_short`) - Device registry event namespace
- `register` (`symbol_short`) - Registration action

**Payload fields:**
- `hashed_imei: BytesN<32>` - SHA-256 hash of the device IMEI
- `owner: Address` - Stellar address of the device owner
- `device_model: String` - Human-readable device model, such as `iPhone 15 Pro`

**Example:**

```text
Topics: ("dev_reg", "register")
Payload: (
  hashed_imei: 0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20,
  owner: GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX,
  device_model: "iPhone 15 Pro"
)
```

Indexer use cases:
- build the canonical off-chain list of registered devices
- notify a user after registration is confirmed
- track registration volume and model distribution

## Planned Lifecycle Events

The following events are documented as the expected stable contract for future
event work. The module files already contain TODO markers for these emissions.

### DeviceStolen

Emitted when a device owner reports a registered device as stolen and records a
recovery bounty.

**Source:** `contracts/haven_registry/src/killswitch.rs::report_stolen`

**Topics:**
- `stolen` (`symbol_short`) - Stolen-device lifecycle action

**Payload fields:**
- `hashed_imei: BytesN<32>` - Device hash being marked stolen
- `bounty_amount: i128` - Promised escrow amount recorded for recovery

**Example:**

```text
Topics: ("stolen")
Payload: (
  hashed_imei: 0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20,
  bounty_amount: 1000000
)
```

### DeviceRecovered

Emitted when the owner confirms a stolen device has been recovered.

**Source:** `contracts/haven_registry/src/recovery.rs::confirm_recovery`

**Topics:**
- `recovered` (`symbol_short`) - Recovery lifecycle action

**Payload fields:**
- `hashed_imei: BytesN<32>` - Device hash being recovered
- `finder: Address` - Stellar address of the finder receiving the bounty

**Example:**

```text
Topics: ("recovered")
Payload: (
  hashed_imei: 0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20,
  finder: GCFINDERXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
)
```

### InsuranceClaimed

Emitted when an insurance claim is filed and device salvage rights move to the
insurer.

**Source:** `contracts/haven_registry/src/insurance.rs::file_insurance_claim`

**Topics:**
- `insured` (`symbol_short`) - Insurance claim lifecycle action

**Payload fields:**
- `hashed_imei: BytesN<32>` - Device hash being claimed
- `insurer: Address` - Stellar address of the insurer receiving salvage rights

**Example:**

```text
Topics: ("insured")
Payload: (
  hashed_imei: 0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20,
  insurer: GCINSURERXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
)
```

## Integration Notes

- Events are emitted in the same transaction as their state transition.
- `DeviceRegistered` is emitted after `DeviceState` is persisted and
  `DeviceCount` is incremented.
- Lifecycle event consumers should refresh `DeviceState` with
  `get_device(hashed_imei)` before showing final UI state.
- Indexers should not rely on raw IMEI values, recovery contact information, or
  other PII appearing in event topics or payloads.

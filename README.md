# Reduration

Reduration, a human-readable time duration format

## Example

- `28 days`
- `1 hours` (cannot be the singular form)
- `2 days 2 nanos`
- `9.58s`

## ABNF Grammar

```abnf
duration =  days   [ws hours]  [ws mins]   [ws secs]   [ws millis] [ws nanos]
duration =/ hours  [ws mins]   [ws secs]   [ws millis] [ws nanos]
duration =/ mins   [ws secs]   [ws millis] [ws nanos]
duration =/ secs   [ws millis] [ws nanos]
duration =/ millis [ws nanos]
duration =/ nanos

days   = dec                [ws] ("days"   | "d")
hours  = dec                [ws] ("hours"  | "h")
mins   = dec                [ws] ("mins"   | "m")
secs   = dec ["." 1*6DIGIT] [ws] ("secs"   | "s")
millis = dec ["." 1*3DIGIT] [ws] ("millis" | "ms")
nanos  = dec                [ws] ("nanos"  | "ns")

dec = DIGIT *(DIGIT / "_")
```

## Semantics

### Data structure

Deserializer must emit an error if a value cannot representable in the following datatype.

```rust
// Almost 256-bit
pub struct Reduration {
    days: Option<u32>, // 0 to 4_294_967_295
    hours: Option<u32>, // 0 to 4_294_967_295
    mins: Option<u32>, // 0 to 4_294_967_295
    secs: Option<u32>, // 0 to 4_294_967_295
    secs_fraction: Option<u32>, // 0 to 999999
    millis: Option<u32>, // 0 to 4_294_967_295
    millis_fraction: Option<u16>, // 0 to 999
    nanos: Option<u32>, // 0 to 4_294_967_295
}
```

### Evaluation

No leap second.

```rust
let timestamp_in_nano: BigUint = nanos + 1000 * (millis + 1000 * (secs + 60 * (mins + 60 * (hours + 24 * days))));
```

## Other formats

- ISO 8601 (RFC 3339)

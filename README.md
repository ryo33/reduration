# Reduration

Reduration, a human-readable (relatively short) time duration format

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
pub struct Reduration {
    days: Option<32>, // 0 to 4_294_967_295
    hours: Option<u32>, // 0 to 4_294_967_295
    mins: Option<u32>, // 0 to 4_294_967_295
    secs: Option<u32>, // 0 to 4_294_967_295
    secs_fraction: Option<u32>, // 0 to 999999999
    millis: Option<u32>, // 0 to 4_294_967_295
    millis_fraction: Option<u32>, // 0 to 999999
    micros: Option<u32>, // 0 to 4_294_967_295
    micros_fraction: Option<u16>, // 0 to 999
    nanos: Option<u32>, // 0 to 4_294_967_295
}
```

### Evaluation

No leap second.

```rust
let timestamp_in_nano: BigUint = nanos + 1000 * (millis + 1000 * (secs + 60 * (mins + 60 * (hours + 24 * days))));
```

### Rounding or truncation

Deserialization must not truncate or round the sub-seconds even if the system does not support microsec or nanosecs. Let users choose round or sail or floor within data conversion.

### Overflow

A deserializer must throw an error if overflow occurred while translate to target data.

## Memory efficient deserialization

As shown in semantics/data structure section, naive data structure for deserialization took over 256-bit. Not to take large memory, the following pseudo technique is recommended.

```rust
let mut nanos = 0;
let mut previous_token_kind = None;
while let Some(token) = parse_next_token(&mut input) {
    if let Some(prev) = previous_next_token {
        assert!(prev > token.kind);
    }
    match token {
        Token::Nanos(value) => {
            checked!(nanos += value);
        }
        Token::Micros { value, frac } => {
            checked!(nanos += value * 1000);
            assert!(frac <= 999);
            checked!(nanos += frac);
        }
        _ => unimplemented!(),
    }
    previous_token_kind = Some(token.kind);
}
```

## Other formats

- [tailhook/humantime](https://github.com/tailhook/humantime)
- ISO 8601 (and RFC 3339)

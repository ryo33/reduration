# Reduration

Reduration, a human-readable second-wise time duration format

## "calender-wise" vs "second-wise"

Let:

- "Calender-wise duration" ignores leap seconds when appending a duration to a specific date-time.
- "Second-wise duration" just handles passed seconds.

Reduration is designed and intended for the "second-wise" duration. Therefore:

- it does not have second-ambiguous units, "month" or "year". "week" is not second-ambiguous but not included to keep it a small format.
- it always interprets an hour as exactly 3600 seconds and a day as exactly 24 * 3600 seconds.

## Use-cases

- in a URL parameter: `?valid_for=1%20days` instead of `?valid_for=86400`
- serialization/deserialization: `"{\"valid_for\": \"2 hours\"}"`
- in chat: "The token is valid for `300s`, not `3000s`."
- 

## Example

- `28 days`
- `1 hours 1 nanos` (cannot be singular form)
- `9.58s` (shorthand style)
- `1h -1s` (same as `3599s`)
- `999_999_999 days` (999999999 is the max value for each field)
- `plus 1 days` and `minus 1 days` (plus/minus must be explicit for signed-reduration)

## ABNF Grammar

```abnf
; **case sensitive**

reduration = day-part
signed-reduration = ("plus" / "minus") day-part

day-part =  days [ws [sign] hour-part]
day-part =/ hour-part

hour-part =  hours [ws [sign] min-part]
hour-part =/ min-part

min-part =  mins [ws [sign] sec-part]
min-part =/ sec-part

sec-part =  secs-frac ; no milli-part
sec-part =/ secs [ws [sign] milli-part]
sec-part =/ milli-part

milli-part =  millis-frac ; no micro-part
milli-part =/ millis [ws [sign] micro-part]
milli-part =/ micro-part

micro-part =  micros-frac ; no nanos
micro-part =/ micros [ws [sign] nanos]
micro-part =/ nanos

days   = 1*9dec [ws] ("days"   / "d")
hours  = 1*9dec [ws] ("hours"  / "h")
mins   = 1*9dec [ws] ("mins"   / "m")
secs   = 1*9dec [ws] ("secs"   / "s")
millis = 1*9dec [ws] ("millis" / "ms")
micros = 1*9dec [ws] ("micros" / "us")
nanos  = 1*9dec [ws] ("nanos"  / "ns")

secs-frac   = 1*9dec "." 1*9dec [ws] ("secs"   / "s")
millis-frac = 1*9dec "." 1*6dec [ws] ("millis" / "ms")
micros-frac = 1*9dec "." 1*3dec [ws] ("micros" / "us")

dec = DIGIT / "_"
sign = ("-" / "+") [ws]
ws = " "
```

## Semantics

### Pseudo data structure

```rust
struct Reduration {
    days: Dec9,
    hours: Signed9,
    mins: Signed9,
    secs: Signed9,
    secs_frac: Dec9,
    millis: Signed9,
    millis_frac: Dec6,
    micros: Signed9
    micros_frac: Dec3,
    nanos: Signed9
}

struct Signed9(i32); // -999_999_999 to 999_999_999
struct Dec9(u32); // 0 to 999_999_999
struct Dec6(u32); // 0 to 999_999
struct Dec3(u16); // 0 to 999

enum Value<P, N> {
    Positive(P),
    Negative(N),
}

struct SignedReduration {
    minus: bool,
    tail: Reduration,
}
```

### Evaluation

No leap second.

```rust
let timestamp_in_nano: BigUint = nanos + 1000 * (millis + 1000 * (secs + 60 * (mins + 60 * (hours + 24 * days))));
```

### Precision (no rounding or truncation)

Lack of precision must be reported as an error. There are two cases (into and from);

1. a reduration object converted into other representation that lacks time precision to represent the same value;
2. a reduration object converted from a value of other representation that has picoseconds-level or more granular values.

Therefore, a user need to manually perform a round/ceil/floor before such a conversion.

### Integer overflow handling

Every integer overflows in conversion between reduration and other representations must be reported as errors. A user still can ignore the overflow errors.

### Signed grammar and unsigned grammar

Reduration format can be used in both use-case, signed duration or unsigned duration, but a system must not allow both grammars `reduration` and `signed-reduration`.

### Negative values are prohibited

Regardless of which grammar, the `day-part` must be 0 or positive nanoseconds, and negative values must be rejected as invalid reduration.

Valid strings:

- `1 hours`
- `-1 hours 61min`

Invalid strings:

- `-1 hours`
- `1 hours -61min`

### Leading zeros

Leading zeros are ignored.

## Implementation

### Serializer

A serializer must follow the grammar.

### Deserializer

A deserializer:

- must handle an input in UTF-8 encoding.
- can allow incorrect casing like `1 Hours` or `1H`.
- can allow incorrect spacing like `1   hours` (too many spaces) or `1mins2secs` (needs a space after `mins`).
- can allow leading/trailing spaces.
- can allow UTF-8 whitespaces.
- should not allow other incorrect grammars not listed in the above.

### Conversion to other representations

A converter should have an option that let users choose `seil`, `floor`, or `round` as fallback logic on the precision error.

### Memory-efficient deserialization

As shown in the semantics/data structure section, naive numerical data structure takes not-tiny memory. The following technique is recommended for memory efficiency.

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

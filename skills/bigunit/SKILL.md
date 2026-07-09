---
name: bigunit
description: "Guide for using the BigUnit TypeScript library for precise arithmetic with numbers of varying precision. Use this skill whenever code involves BigUnit, BigUnitFactory, precision-sensitive arithmetic, cryptocurrency token amounts, financial calculations with bigint, or when the user needs to work with numbers that have different decimal precisions (like converting between ETH 18 decimals and USDC 6 decimals). Also use when you see imports from 'bigunit', references to BigUnitish types, or asPrecision() calls."
---

# BigUnit Library Guide

BigUnit is a TypeScript library for precise arithmetic with numbers of varying precision, using `bigint` internally. Zero dependencies.

```
npm install bigunit
```

## Core Concepts

**Internal representation**: Values are stored as `bigint` scaled by `10^precision`. So `123.45` at precision 2 is stored as `12345n`.

**Precision propagation**: Arithmetic operations return the HIGHEST precision of the two operands. Use `.asPrecision(n)` to convert the result.

## Quick Reference

### Creating Values

```typescript
import { BigUnit, BigUnitFactory, RoundingMethod } from "bigunit";

// Flexible factory - accepts number, string, bigint, or BigUnit
BigUnit.from(123.456, 5)                    // from number
BigUnit.from("789.012483655091331", 18)     // from decimal string
BigUnit.from(1234567n, 12)                  // from bigint (= 0.000001234567)
BigUnit.from(existingBigUnit)               // from another BigUnit (keeps its precision)

// Specific constructors
BigUnit.fromNumber(123.456, 5)
BigUnit.fromDecimalString("0.00000012", 8)
BigUnit.fromBigInt(500000000000000000n, 18) // 0.5 with 18 decimals
BigUnit.fromValueString("12345", 2)         // internal bigint as string
BigUnit.fromObject({ value: "12345", precision: 2 })

// Factory for consistent precision (great for token types)
const BTC = new BigUnitFactory(8, "BTC");
const ETH = new BigUnitFactory(18, "ETH");
const USDC = new BigUnitFactory(6, "USDC");

const balance = BTC.from(0.005);            // precision 8, name "BTC"
```

### Arithmetic

All methods accept `BigUnitish` (number | string | bigint | BigUnit). Result precision = max(operand precisions).

```typescript
unit.add(other)       // addition
unit.sub(other)       // subtraction
unit.mul(other)       // multiplication
unit.div(other)       // division
unit.mod(other)       // modulo
unit.abs()            // absolute value
unit.neg()            // negate
unit.percent(20)      // 20% of unit
unit.fraction(1, 4)   // 1/4 of unit
unit.percentBackout(10)  // reverse a 10% markup
```

### Comparisons

```typescript
unit.eq(other)        // equal
unit.gt(other)        // greater than
unit.lt(other)        // less than
unit.gte(other)       // greater or equal
unit.lte(other)       // less or equal
unit.isZero()
unit.isPositive()
unit.isNegative()
BigUnit.max(a, b)
BigUnit.min(a, b)
```

### Output

```typescript
unit.toString()       // full precision decimal: "123.45600"
unit.format(2)        // formatted to N decimals: "123.46" (rounds)
unit.toNumber()       // JS number (may lose precision for large values)
unit.toBigInt()       // internal bigint representation
unit.toObject()       // { value: string, precision: number, name?: string }
unit.toDTO()          // { value, precision, decimalValue, name }
unit.toJSON()         // serialized string
```

### Precision Conversion

```typescript
unit.asPrecision(6)                           // truncates by default
unit.asPrecision(6, RoundingMethod.Nearest)   // round half up
unit.asPrecision(6, RoundingMethod.Floor)      // always down
unit.asPrecision(6, RoundingMethod.Ceil)       // always up
unit.asOtherPrecision(otherUnit)              // match another unit's precision
BigUnit.asHighestPrecision(a, b)              // returns [a', b'] at max precision
```

## Common Pitfalls

**1. bigint requires explicit precision**
```typescript
// WRONG - throws MissingPrecisionError
unit.add(1000n);

// RIGHT
unit.add(1000n, 8);
// or
unit.add(BigUnit.from(1000n, 8));
```

**2. Result precision may surprise you**
```typescript
const usdc = BigUnit.from(100, 6);   // precision 6
const rate = BigUnit.from(1.5, 18);  // precision 18
const result = usdc.mul(rate);       // precision 18, NOT 6!
// Convert back:
result.asPrecision(6);
```

**3. asPrecision truncates by default, doesn't round**
```typescript
BigUnit.from("1.999", 3).asPrecision(2)                          // "1.99" (truncated)
BigUnit.from("1.999", 3).asPrecision(2, RoundingMethod.Nearest)  // "2.00" (rounded)
```

**4. Use strings to avoid JavaScript float issues**
```typescript
// Risky - float arithmetic happens before BigUnit sees it
BigUnit.from(0.1 + 0.2, 18);

// Safe
BigUnit.from("0.3", 18);
```

**5. toNumber() can lose precision for very large values**
Use `.toString()` or `.format()` for display. Only use `.toNumber()` when you genuinely need a JS number and accept the precision limits.

## Typical Patterns

### Token conversion (crypto)
```typescript
const ETH = new BigUnitFactory(18, "ETH");
const USDC = new BigUnitFactory(6, "USDC");
const USD = new BigUnitFactory(4, "USD");

const ethBalance = ETH.from("1.5");
const ethPrice = USD.from(2000);
const usdValue = ethBalance.mul(ethPrice).asPrecision(USD.precision);
console.log(`$${usdValue.format(2)}`); // "$3000.00"
```

### Fee calculation
```typescript
const amount = BTC.from(1.5);
const fee = amount.percent(2);           // 2% fee
const total = amount.add(fee);
```

### Splitting / distributing
```typescript
const pool = USDC.from(1000);
const share = pool.fraction(1, 3);       // 1/3 of pool
```

### Removing a markup
```typescript
// Price includes 10% tax, get pre-tax amount
const priceWithTax = USD.from(110);
const preTax = priceWithTax.percentBackout(10); // ~100
```

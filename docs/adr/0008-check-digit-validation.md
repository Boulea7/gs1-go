# ADR 0008: Opt-in GTIN check-digit validation

## Status
Accepted

## Date
2026-10-10

## Context
Scanner workflows need tolerant parsing, while some callers need to reject
GTINs with mismatched check digits. The existing `ValidateAssociations`
option, CLI `-strict`, and WASM `strict` explicitly validate AI pairing and
exclusion rules. Expanding those options would tighten an established
contract for callers that only requested association checks.

The GS1 Syntax Dictionary's
[AI 01 and AI 02 entries](https://github.com/gs1/gs1-syntax-dictionary/blob/2026-01-27/gs1-syntax-dictionary.txt)
define 14-digit numeric GTINs with the `csum` check-digit constraint. The
package already implements the modulo-10 algorithm in `ValidateGTIN` and
exposes `ErrInvalidCheckDigit`.

## Decision
- Keep default parsing lenient and keep the existing association options
  association-only.
- Add independent `ParseOptions.ValidateCheckDigits`, CLI `-check-digits`,
  and WASM `validateCheckDigits` options. They validate AI 01 and AI 02,
  including supported bare GTIN and retail-carrier forms normalized to AI 01.
- Extend the documented `Barcode.Validate()` semantic umbrella to run
  association checks followed by these GTIN checks. Validate its current
  `Elements` without reparsing `Raw`.
- Preserve error priority: syntax, length, and data type first; associations
  next when enabled; check digits last. Keep only the first check-digit error
  locally during parsing, with no persistent offsets or new validation
  framework.
- Report normalized AI-start or carrier-payload-start positions for parsing
  errors. Post-parse validation provides AI context only.
- Preserve existing bare-GTIN/AI-prefix disambiguation. Eight-digit bare GTIN
  interpretation still requires `AssumeBareGTIN8`.
- For eight-digit UPC-E (`]E0`), compare the received check digit with the
  expanded expected digit at conversion time. Keep default expansion intact.
  For a 13-digit `]I2` payload, retain synthesis of the stripped check digit;
  there is no received digit to validate.

Alternatives rejected: expanding the old strict options would change their
documented meaning; a general validation framework or stored input offsets
would add complexity beyond the two GTIN AIs.

## Consequences
- Callers can select all four combinations of association and GTIN checks.
  Association-only callers can continue to use `ValidateAssociations()`;
  callers of `Barcode.Validate()` gain GTIN checking.
- Lenient UPC-E conversion loses the original digit. `Barcode.Validate()`
  can validate the normalized GTIN, but cannot recover an earlier mismatch
  from the scan. Callers that need that check must enable the parsing option.
- CLI errors keep their existing transport and exit codes; WASM errors keep
  the `{ error: string }` shape. Regulatory APIs are unaffected.
- There are no new dependencies, checksum algorithms, or metadata fields.
  The opt-off `ParseInto` hot path retains zero allocations. The optional
  Node test runs the actual WASM build without adding Node to `make all`.

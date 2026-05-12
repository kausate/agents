# Identifier formats per jurisdiction

Kausate normalizes companies to `kausateId = co_{jurisdiction}_{opaque}`. When *searching*, you can pass native registry identifiers via `companyNumber` or `advancedQuery`. The major formats:

| Jurisdiction | Native identifier | Format / example |
|---|---|---|
| `de` | Court-coded register number | `R2201_HRB 31248` (Cologne court HRB) — court code + register type + number |
| `gb` | Companies House CRN | 8 chars: `00445790` or `MC123456` |
| `fr` | SIREN / SIRET | 9-digit (SIREN) parent, 14-digit (SIRET) establishment |
| `nl` | KVK number | 8-digit |
| `at` | Firmenbuch FNR | digits + lowercase letter (`123456a`) |
| `it` | REA | regional code + number (`MI-1234567`); also Codice Fiscale (16) and P.IVA (11) |
| `ch` | UID | `CHE` + 9 digits |
| `be` | Enterprise number | 10 digits, optional dots: `1234.567.890` |
| `pl` | KRS / NIP / REGON | 10-digit (KRS) / 10-digit (NIP) / 9 or 14-digit (REGON) |
| `es` | NIF / CIF | letter + 7 digits + check digit |
| `se` | Organisationsnummer | 10 or 12 digits |
| `no` | Organisasjonsnummer | 9 digits |
| `fi` | Y-tunnus | 7 digits + `-` + check digit |

Jurisdiction codes are **always lowercase** (`de`, `fr`, `gb`, `nl`, …) — never `DE` or `GB`.

For everything else and for full coverage, call `GET /v2/platform/jurisdictions` to discover supported jurisdictions and their capabilities at runtime.

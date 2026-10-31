# `roundToCurrencyPrecision`: validar el `amount` (issue #16977 / PR #16979)

Parche listo para aplicar sobre el `develop` actual de `medusajs/medusa`.

## Qué incluye el parche

`packages/modules/payment/src/services/payment-module.ts` → `roundToCurrencyPrecision()` valida antes de redondear:

| Entrada | Resultado |
|---|---|
| `null` / `undefined` | `INVALID_DATA` → `Amount to round is required.` |
| `NaN`, `±Infinity`, `"NaN"`, `"Infinity"` | `INVALID_DATA` → `Amount to round must be a finite number.` |
| negativo (número o string) | `INVALID_DATA` → `Amount to round cannot be negative.` |
| `0` | pasa (sigue devolviendo `0`) |

Nota técnica: `bignumber.js` **no** lanza con `NaN`/`Infinity`, los representa y devuelve
`isFinite() === false`, por eso la comprobación es explícita (`converted.isFinite()`).
El cambio va **encima** del arreglo de locale que ya está en develop
(`resolvedOptions().maximumFractionDigits`, PR #17452), así que no lo pisa.

Tests: 9 casos nuevos dentro del spec que ya existe
(`src/services/__tests__/round-to-currency-precision.spec.ts`), no se crea un segundo archivo.
Changeset: `.changeset/great-pandas-invite.md` (`@medusajs/payment`, patch).

Verificado en el sandbox: `yarn test` del paquete `@medusajs/payment` → **30/30 tests OK**.
El parche aplica con `git apply --check` limpio sobre `medusajs/medusa@develop` (7e797dfd59).

## Cómo aplicarlo y reabrir el PR

```bash
# 1. Trae el develop más reciente de upstream
git fetch https://github.com/medusajs/medusa.git develop

# 2. Rehaz la rama del PR sobre ese develop
git checkout -B fix/validate-rounding-amount FETCH_HEAD
git apply contrib/rounding-amount/validate-rounding-amount.patch
git add -A packages/modules/payment .changeset/great-pandas-invite.md
git commit -m "fix(payment): validate the amount in roundToCurrencyPrecision"

# 3. Verifica
(cd packages/modules/payment && yarn test)

# 4. Push (tu cuenta sí puede empujar commits que tocan .github/workflows)
git push --force-with-lease origin fix/validate-rounding-amount
```

El `--force-with-lease` es necesario porque la rama actual está basada en un develop viejo.
Después: comenta el issue #16977 (texto abajo), espera a que te asignen y edita la descripción
del PR #16979 añadiendo `Fixes #16977`; el bot lo reabre automáticamente.

> Un token de GitHub App sin permiso `workflows` no puede empujar esta rama, porque el
> `develop` de octubre modifica `.github/workflows/*`. Con tu usuario no hay problema.


## Texto para comentar en el issue #16977

```markdown
I'd like to work on this one — I already have a patch ready that validates the amount at the
top of `roundToCurrencyPrecision` (`packages/modules/payment/src/services/payment-module.ts`):

- `null`/`undefined` → `INVALID_DATA` (`Amount to round is required.`)
- `NaN`/`Infinity` (number or string) → `INVALID_DATA` (`Amount to round must be a finite number.`)
  — `bignumber.js` represents those instead of throwing, so the check is explicit
- negative → `INVALID_DATA` (`Amount to round cannot be negative.`)
- `0` still passes through unchanged

It sits on top of the locale-independent precision lookup that is already in develop, and the
cases are covered in the existing `round-to-currency-precision.spec.ts` (9 new tests; the whole
`@medusajs/payment` unit suite passes).

I had opened #16979 for this, but it was closed automatically because the issue wasn't assigned
to me yet. Could a maintainer assign it? I'll reopen the PR as soon as it is assigned. Thanks!
```

## Descripción para el PR #16979

```markdown
## Summary

**What** — `roundToCurrencyPrecision` now validates the amount it receives: missing, non-finite
and negative values throw an `INVALID_DATA` error, zero is still allowed.

**Why** — The helper feeds the capture and payment-collection guards, so a `null`, `NaN`,
`Infinity` or negative amount used to be rounded silently and only surfaced later as a confusing
`You cannot capture more than the authorized amount substracted by what is already captured.`
(issue #16977).

**How** — Validation runs before the precision lookup in
`packages/modules/payment/src/services/payment-module.ts`:

- `amount == null` → `Amount to round is required.`
- `!converted.isFinite()` → `Amount to round must be a finite number.` (`bignumber.js` keeps
  `NaN`/`Infinity` as non-finite values instead of throwing, so it has to be checked explicitly)
- `MathBN.lt(converted, 0)` → `Amount to round cannot be negative.`

The locale-independent `resolvedOptions().maximumFractionDigits` lookup already on develop is
untouched.

**Testing** — 9 new cases in
`packages/modules/payment/src/services/__tests__/round-to-currency-precision.spec.ts`
(missing, `NaN`, `±Infinity`, `"NaN"`, `"Infinity"`, negative number and string, zero).
`yarn test` in `packages/modules/payment`: 30/30 passing.

## Checklist

- [x] I have added a changeset for this PR
- [x] The changes are covered by relevant tests
- [x] I have verified the code works as intended locally
- [x] I have linked the related issue(s) if applicable

Fixes #16977
```


## Issue pendiente: el test de draft orders sigue en `it.skip`

`integration-tests/http/__tests__/order/store/order.spec.ts:162` todavía tiene
`it.skip("should throw an error when fetching draft order", ...)`. El PR #16984 lo reactivaba,
pero se cerró por no tener issue enlazado: hay que abrir el issue primero.

Título sugerido: `[Bug]: store draft order isolation test is still skipped`

```markdown
### What's the bug?

`integration-tests/http/__tests__/order/store/order.spec.ts` still skips the test that asserts a
draft order can't be fetched through the public store route:

```ts
// TODO: This should have thrown an error, but doesn't seem to.
it.skip("should throw an error when fetching draft order", async () => {
```

So `GET /store/orders/:id` with a draft order id is effectively untested.

### Why the test fails today

1. The caught error isn't destructured (`const response = await api.get(...).catch((e) => e)`),
   so `response.status` / `response.data.message` can't be asserted — compare with the
   `const { response } = ...` pattern used a few lines above in the same file.
2. The `beforeAll` fixture completes the draft order via `/admin/orders/:id/complete` right after
   creating it, so by the time the assertion runs the order is no longer a draft.

### Expected behavior

The test is re-enabled with the fixture fixed and the response destructured, so it fails if the
store route ever exposes a draft order.

### Additional context

I had #16984 re-enabling it; it was closed automatically because no issue was linked.
Happy to reopen it once this issue is assigned to me.
```

> Ojo: no pude ejecutar ese test de integración aquí (necesita Postgres y el server de Medusa),
> así que antes de reabrir #16984 hay que correrlo en local (`yarn test:integration` en la raíz,
> filtrado a `integration-tests-http`).


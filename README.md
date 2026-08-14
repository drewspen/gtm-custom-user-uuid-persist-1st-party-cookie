# gtm-custom-user-uuid-persist-1st-party-cookie

A **Google Tag Manager (GTM) container export** (JSON) — not a single tag template — that you import directly into a GTM workspace to generate a client-side, per-visitor identifier and persist it as a first-party cookie. It's a companion piece to this author's [gtm-persist-utm-url-query-strings-1st-party-cookie](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie) repository and reuses the same underlying "Cookie Creator" / "Get Root Domain" building blocks, adding a hash-based ID generator on top.

This README is based on a direct inspection of the container export's actual contents (2 tags, 2 triggers, 6 variables, 3 folders, 1 built-in variable, and 5 bundled custom templates), not just the repository's one-line description.

## What problem does this solve?

Many marketing/analytics setups need a stable, anonymous way to recognize the same browser across visits — independent of (and often complementary to) Google Analytics' own client ID — for example, to de-duplicate visitor counts, correlate events across sessions in a data warehouse, or pass a consistent visitor reference into a CRM or CDP. This container export generates such an identifier entirely client-side, using only sandboxed GTM building blocks (no external script or server call), and stores it as a first-party cookie so the same value is read back on every subsequent visit.

## ⚠️ Important consideration: this is a hash, not an "encrypted UUID"

- **It isn't a 'true' UUID.** A 'true' UUID (e.g., RFC 4122) is a 128-bit value formatted as `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. What this container actually generates is the output of the **cyrb53** hash function — a fast, non-cryptographic 53-bit (or, with 64-bit output enabled as configured here, a 16-character hex string) hash designed for things like cache keys and checksums, not for identity or security purposes. It has no relation to the UUID spec's format or collision guarantees.
- **It isn't encrypted.** Hashing is not encryption — there's no key, and (unlike encryption) the operation isn't intended to be reversible in the first place, but a *non-cryptographic* hash like cyrb53 also offers none of the collision-resistance or preimage-resistance guarantees of a cryptographic hash (like SHA-256). It should not be treated as suitable for security-sensitive identifiers.

In short: this container produces a **pseudo-random, hash-derived identifier**, seeded from a random number and the current timestamp, that behaves like a UUID for casual first-party de-duplication purposes. If you need a cryptographically strong or spec-compliant identifier, this isn't it as configured.

## How the ID is generated

The identifier is built from three chained custom variables, then hashed:

1. **`Random Number`** (GTM's built-in `RANDOM` variable type) — generates a random number on evaluation.
2. **`Session Timestamp`** (via the bundled **Timestamp** template, by *luratic*) — calls GTM's sandboxed `getTimestampMillis()` API to get the current time in milliseconds since epoch. Despite the name, this is simply "the current timestamp at the moment this variable is evaluated," not a timestamp pinned to session start — the "Session" in its name reflects its intended use here, not a technical guarantee.
3. **`Session Timestamp String`** (via the bundled **Number & String Operations** template, by *mbaersch*) — performs a `calc` → `add` operation (`{{Session Timestamp}} + 0`) with `resultTransformation: string`, which is really just a convenient way to coerce the numeric timestamp into a string.
4. **`UUID Hash Value`** (via the bundled **cyrb53 Hasher** template, by *mbaersch*) — hashes the string `{{Random Number}}.{{Session Timestamp String}}` (e.g., `"482910473.1749485229123"`), using `{{Random Number}}` again as the hash seed, with **64-bit output** enabled — producing a 16-character hex string as the final ID value.

This value is what actually gets written to the `_my_uuid` cookie.

## Custom templates bundled in this export

The export bundles **five** custom GTM templates under `customTemplate` in the JSON. Two are shared with this author's UTM-persistence container; three are new:

| Template | Author | Type | Purpose |
|---|---|---|---|
| **Cookie Creator** | `gtm-templates-anto-hed` | Tag | Sets a browser cookie via sandboxed JavaScript, avoiding the need for extra CSP `script-src`/hash allowances that a raw Custom HTML cookie-setting tag would require. |
| **Get Root Domain** | `mbaersch` | Variable | Extracts the registrable root domain from `{{Page Hostname}}`, used as the cookie's `Domain` value. |
| **Timestamp** | `luratic` | Variable | Returns the current time in milliseconds since epoch via GTM's `getTimestampMillis` sandbox API. |
| **Number & String Operations** | `mbaersch` | Variable | A general-purpose math/string utility variable (arithmetic, boolean logic, `Math` methods, string functions); used here just for its `add`-with-string-output mode to stringify a timestamp. |
| **cyrb53 Hasher** | `mbaersch` | Variable | Implements the [cyrb53 hash function](https://stackoverflow.com/a/52171480) using polyfills for sandbox APIs GTM doesn't expose natively (`charCodeAt`, 32-bit integer multiplication, `padStart`), since GTM's sandboxed JS environment lacks some standard JS built-ins. |

## Tags, triggers, and the write-once/renew pattern

The container defines exactly **two tags**, both using the `Cookie Creator` template, following the same write-once-then-renew pattern seen in this author's UTM-persistence container — but triggered on **`PAGEVIEW`** here (rather than `INIT`, as used in the UTM container):

### 1. `Create UUID`
Fires when:
- `{{UUID Hash Value}}` is present/non-empty (i.e., a fresh hash was successfully computed).
- `{{UUID Cookie Value}}` (the existing `_my_uuid` cookie) is **not** already set.

**Effect:** writes `_my_uuid` = the freshly computed `{{UUID Hash Value}}` — establishing the visitor's identifier for the first time. Because this only fires when no cookie exists yet, a given browser gets exactly one ID for the life of the cookie; the hash is never recomputed for a returning visitor.

### 2. `Rewrite UUID`
Fires when:
- `{{UUID Cookie Value}}` **is** already set.

**Effect:** re-writes `_my_uuid` using its own existing value (not a new hash) — so the ID itself never changes on repeat visits, but its expiration is refreshed each time, extending the effective lifetime of the identifier for as long as the visitor keeps returning within the window.

### Cookie configuration (both tags)

- **Cookie name:** `_my_uuid`
- **Domain:** `{{Root Domain}}` — shared across subdomains.
- **Path:** `/`
- **Expiration:** 24 months (longer than the 12-month expiration used for the UTM cookies in this author's sibling repository).
- **SameSite:** `None`, with the SameSite checkbox enabled.
- **Secure:** disabled (`checkbox1Secure: false`) on both tags — see [Known issue](#known-issue-samesitenone-without-secure) below.
- **Consent:** both tags require `functionality_storage` consent (`consentStatus: NEEDED`).
- **Firing option:** `ONCE_PER_EVENT`.

## Variables

| Variable | Type | Purpose |
|---|---|---|
| `Random Number` | Built-in (`RANDOM`) | Random seed component for the hash. |
| `Session Timestamp` | Custom (Timestamp template) | Current epoch milliseconds. |
| `Session Timestamp String` | Custom (Number & String Operations) | Stringified version of `Session Timestamp`. |
| `UUID Hash Value` | Custom (cyrb53 Hasher) | The freshly computed hash-based ID, used only by the `Create UUID` tag. |
| `UUID Cookie Value` | 1st-Party Cookie | Reads back the existing `_my_uuid` cookie, used by both tags' trigger conditions and by `Rewrite UUID`'s value. |
| `Root Domain` | Custom (Get Root Domain template) | Root domain computed from `{{Page Hostname}}`, used as the cookie's `Domain`. |

One GTM **built-in variable**, `Page Hostname`, is enabled and feeds into `Root Domain`.

## Folder organization

Tags and variables are grouped into three GTM folders: an **`Analytics`** folder (holding `Root Domain` and `Random Number` — shared/general-purpose variables), a **`UUID Cookie Creator`** folder (the two tags plus `UUID Cookie Value` and `UUID Hash Value`), and a **`Session Timestamp`** folder (the two timestamp-related variables).

## Prerequisites

Because this is a full container export, importing it will bring in everything needed except:

1. **A GTM web container** — ideally a scratch/sandbox workspace, since importing can create duplicate tags/triggers/variables if names collide with what's already in your container.
2. **A consent management setup** wired into GTM's consent mode capable of granting `functionality_storage` consent — both tags require it to fire.
3. If you also plan to use this author's [UTM-persistence container](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie), note that **both exports bundle their own copies of the `Cookie Creator` and `Get Root Domain` templates** — importing both into the same container may prompt you to resolve duplicate template conflicts; keep one copy of each and re-point tags/variables at it if needed.

## Getting started

### Import into Google Tag Manager

1. In GTM, go to **Admin** → **Import Container**.
2. Choose `gtm_custom_user_uuid_persist_1st_party_cookie.json` from this repository.
3. Select the target container and **choose a new workspace** (recommended) rather than overwriting an existing one, so you can review the merge before publishing.
4. Choose **Merge**, and review the import summary — the export's account/container IDs (`999999999`, `GTM-AAAAAAAA`) are placeholder/scrubbed values, so GTM will import everything by name into your own container.
5. Confirm the import. All five bundled custom templates will import alongside the tags, triggers, and variables that depend on them.

### Post-import checklist

- **Confirm your consent setup** grants `functionality_storage` appropriately, or neither tag will ever fire.
- **Review the `Secure` cookie setting** (see below) before publishing to a production HTTPS site.
- **Decide if cyrb53 is fit for your purpose.** If you need a real UUID (for interoperability with systems expecting UUID-formatted values) or a cryptographically robust identifier, you'll want to swap in a different generation method rather than relying on this hash as-is.
- **Test in Preview mode**:
  - On first visit (no `_my_uuid` cookie present), confirm `Create UUID` fires and sets `_my_uuid` to a 16-character hex string.
  - On a subsequent page view/visit, confirm `Rewrite UUID` fires instead, and that `_my_uuid`'s value is unchanged from the first visit (only its expiration should refresh).

### Using the persisted value

Once set, `_my_uuid` can be read back anywhere later in the visit — via the included `UUID Cookie Value` variable or your own equivalent — to attach a consistent per-visitor reference to analytics events, form submissions, or other data sent to downstream systems.

## Known issue: `SameSite=None` without `Secure`

As with this author's UTM-persistence container, both cookie tags here set `checkbox1SameSite: true` with `dropDownMenu1SameSite: "None"`, while `checkbox1Secure` is `false`. Modern browsers (Chrome and others following the same spec) **require the `Secure` attribute whenever `SameSite=None` is used** — a cookie set this way will typically be **rejected outright** rather than falling back to a different SameSite behavior. Unless your site has a specific reason to avoid `Secure`, you'll likely want to enable the Secure checkbox on both tags after import so the `_my_uuid` cookie actually gets set in current browsers.

## Notes

- The container is intentionally small in scope: 2 tags, 2 triggers, 6 variables, 3 folders, 1 built-in variable, and 5 custom templates.
- This is an unofficial, personal automation export and is not affiliated with or endorsed by Google, nor by the authors of the bundled `Cookie Creator`, `Get Root Domain`, `Timestamp`, `Number & String Operations`, or `cyrb53 Hasher` community templates. Always review a container export's tags, triggers, and variables — and test thoroughly in a sandbox workspace — before merging it into a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this container export in a commercial or redistributed context.

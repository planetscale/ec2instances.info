# Handoff: GCP scraper fixes — live-credential verification and upstream PRs

This document is for a Claude agent (or human) with **zero context** on the
prior session. It explains what was built, why, and exactly what remains:
running a real end-to-end GCP scrape with live credentials, validating the
output, and opening two upstream PRs.

## 1. Goal

Contribute **two separate upstream PRs** to
[vantage-sh/ec2instances.info](https://github.com/vantage-sh/ec2instances.info)
fixing GCP data bugs:

1. **Missing machine series** — many GCP families (C2, C4A, C4D, C4N, N4A,
   M4N, H4D, X4, M1, A4, A4X, G4) never appear on the site.
2. **Missing bundled Local SSD pricing** — `-lssd` shapes and Z3 are priced
   identically to their SSD-less twins, and `local_ssd`/`local_ssd_size` are
   never populated for them.

PRs must target upstream's **`develop`** branch (their README, line 38:
"Make sure your pull requests target `develop` since this is our staging").
This fork (**planetscale/ec2instances.info**) carries the work.

## 2. State of the work

| Branch | Commit | Contents |
|---|---|---|
| `gcp-family-classification` | `cc9299a` | Missing-series fix (api.go +45/-6, scrape.go +8, api_test.go +99) |
| `gcp-local-ssd-pricing` | `7d6996d` | Bundled Local SSD pricing fix (api.go +140/-20, scrape.go +83, local_ssd_test.go +217) |
| `gcp-fixes-handoff` (this branch) | merge of both | **For end-to-end testing only — do not PR this branch upstream** |

Both feature branches are cut from fork `main` (`f925996`, which matches
upstream). All three branches: `go build ./...`, `go test ./...`, and
`gofmt -l .` are clean (run from `scraper/`).

Unit tests use fixture SKU display names verified **verbatim** against
Google's live SKU catalog (https://cloud.google.com/skus/sku-groups) on
2026-07-10.

**Cross-branch dependency:** C4D/C4A `-lssd` shapes need BOTH fixes to price
fully correctly — fix 1 makes the families exist at all, fix 2 adds the
bundled SSD component. Each branch alone is still a strict improvement and
stands on its own; the merged result on this branch is what a full validation
run should exercise.

## 3. Root causes (with file:line, as on this merged branch)

1. **Family allowlist regex.** `machineTypeRegex`
   (`scraper/gcp/api.go:632`) is an explicit alternation of family tokens;
   any family not listed parses to an empty `machineFamily`, gets no
   core/RAM rates, so every shape in that family ends with zero price and is
   dropped in `processGCPData` (`scraper/gcp/scrape.go:157`; the zero-price
   skip is at `scrape.go:522`, and instances with no pricing at all are
   discarded at `scrape.go:640`, "Only include instances that have pricing
   data"). Fix: add the missing families to the regex.

2. **Legacy first-generation SKU names.** C2 and M1 SKUs predate the
   "`<FAMILY> Instance Core/Ram`" naming — they read
   "`Compute optimized Core running in ...`" and
   "`Memory-optimized Instance Ram running in ...`" — no family token, and
   no "instance" token for C2, so they failed both the family regex and the
   `isInstanceSKU` gate in `processGCPData`. Fix: `legacySKURegex`
   (`scraper/gcp/api.go:646`) admits them through the gate
   (`scrape.go:318-319`) and maps them to C2/M1 in
   `parseMachineTypeFromSKU` (`api.go:799-809`). M1's "Upgrade Premium"
   surcharge SKUs (how M2 is billed) are deliberately excluded there.

3. **Local SSD pricing never implemented.** Bundled-SSD shapes (`-lssd`
   suffix, all of Z3, A3, etc.) are billed core + RAM + bundled SSD
   GiB-months, but the scraper only ever priced core + RAM. Fix:
   `parseLocalSSDSKU` (`api.go:721`) buckets Local SSD usage SKUs
   (intercepted before the instance gate at `scrape.go:283-313`),
   `bundledLocalSSDCapacityGB` (`api.go:212`) reads capacity from the
   machineTypes API (`bundledLocalSsds.partitionCount`, with a
   description-string fallback matching "`N local ssd`"), and the SSD
   component is folded into the shape price at `scrape.go:516-520`.
   Z3 partitions are 3,000 GiB; every other series is 375 GB
   (`localSSDPartitionGB`, `api.go:200`).

## 4. What remains — YOUR task

Run a **real end-to-end scrape** with GCP credentials and validate the
output. Everything so far was validated against unit-test fixtures only; no
live API call has been made with these changes.

### 4a. Credentials / environment

Three env vars, all required (checked in `scraper/main.go:50-54`):

- `GCP_CLIENT_EMAIL` — service account email (`scraper/gcp/oauth.go:54`)
- `GCP_PRIVATE_KEY` — the service account's RSA private key, PEM
  (`oauth.go:55`; newlines matter — quote it)
- `GCP_PROJECT_ID` — any project the SA can see; used for the
  machineTypes and regions Compute API calls (`scraper/gcp/api.go:485,868`)

OAuth scopes requested (`oauth.go:70`):
`https://www.googleapis.com/auth/compute.readonly` and
`https://www.googleapis.com/auth/cloud-billing.readonly`. So the SA needs
roughly Compute Viewer + Billing Catalog read access on the project.

### 4b. Run command (verified against `scraper/main.go`)

The scraper binary takes **no CLI arguments**. It scrapes AWS + Azure + GCP
unless filtered by the `ALLOWED_SERVICES` env var (comma-separated,
`main.go:19-33`). For a GCP-only run:

```sh
cd scraper
ALLOWED_SERVICES=gcp \
GCP_PROJECT_ID=... \
GCP_CLIENT_EMAIL=... \
GCP_PRIVATE_KEY="$(cat key.pem)" \
go run .
```

(Do **not** run `go run . gcp` — positional args are ignored; without
`ALLOWED_SERVICES=gcp` it will demand AWS and Azure credentials too.)

Output lands at `www/gcp/instances.json` **relative to the cwd** (so
`scraper/www/gcp/instances.json` when run as above; `SaveInstances` creates
the directory). The repo-level `fetch_data.sh` is the Dockerized equivalent
and already forwards the three GCP env vars.

Progress log lines to watch: "GCP SKU filtering: parsed=... priced=...
skippedByTaxonomy=... cudSKUs=... localSSDSKUs=..." — `localSSDSKUs`
should be well above zero.

### 4c. Validate the emitted JSON

Expected **us-central1 on-demand (Linux) hourly** prices, hand-verified
against Google's published pricing on 2026-07-10:

| instance_type | expected hourly (USD) |
|---|---|
| `c4d-standard-8` | ≈ 0.377982 |
| `c4d-standard-8-lssd` | ≈ 0.460174 |
| `c4-standard-8-lssd` | ≈ 0.477532 |
| `c3-standard-8` | 0.403216 |
| `c3-standard-8-lssd` | ≈ 0.444312 |
| `c2-standard-8` | 0.417616 |
| `m1-megamem-96` | 10.65216 |
| `n4a-standard-8` | 0.308 |
| `z3-highmem-88-highlssd` − `z3-highmem-88-standardlssd` | delta ≈ 1.972603 |

Also verify:

- `local_ssd: true` and `local_ssd_size` populated for every `-lssd` shape
  and all Z3 shapes (field names in `scraper/gcp/gcp_instance.go:57-58`).
- Attachable-SSD families (n1/n2/n2d/c2) are **unchanged**: `local_ssd`
  stays `false`, `local_ssd_size` absent, prices identical to a pre-fix run
  (`bundledLocalSSDCapacityGB` returns 0 for them by design).
- Spot and CUD prices for `-lssd` shapes also include the SSD component
  (spot uses the spot SSD rate; see `localSSDRate` usage in `scrape.go`).

### 4d. Things to watch / known unknowns

- **`bundledLocalSsds.partitionCount` population**: unit fixtures assume the
  machineTypes API returns it for `-lssd`/Z3/A3 shapes. If the live API
  omits it, the description-string fallback ("`N local ssd`",
  `descriptionLocalSSDRegex`, `api.go:194`) should catch it — confirm one
  or the other actually fired (a shape with `local_ssd_size: 0` that should
  have SSD means both failed).
- **X4 / H4D / A4X may not appear** even after the fix — they may lack
  public SKUs or be invisible to your project/zones. Absence is not
  necessarily a bug; check whether the billing catalog returns SKUs for them.
- **M2 remains unpriced by design.** M2 is billed as M1 base rates plus
  "Upgrade Premium" surcharge SKUs; pricing it correctly needs base+premium
  summing. Documented follow-up, deliberately out of scope.
- **Accelerator families (A2/A3/A4/G2/G4) exclude GPU SKUs** — their listed
  price is core+RAM(+SSD) only. Pre-existing upstream limitation; do NOT
  fix in these PRs.

## 5. Known cosmetic issue

The comment block above `familyLocalSSDSKURegex` (`scraper/gcp/api.go`,
~lines 699-712) gives the generic SKU examples as
"`SSD backed Local Storage running in Paris`" / "`...attached to Spot
Preemptible VMs running in Paris`". Live catalog names also appear as
"`SSD backed Local Storage`" / "`SSD backed Local Storage in <City>`" /
"`SSD backed Local Storage attached to Spot Preemptible VMs [in <City>]`"
— i.e. the "running in <region>" tail is not always present. The regex is
prefix-anchored (`^ssd\s+backed\s+local\s+storage\b`) so it handles all
forms correctly; only the comment is stale. Fix the comment if you touch
that code, otherwise leave it.

## 6. PR mechanics

- Open **two separate PRs**, one per feature branch, from this fork to
  `vantage-sh:develop` (**NOT `main`**).
- Do **NOT** PR the merged `gcp-fixes-handoff` branch upstream — it exists
  only so validation exercises both fixes together.
- Run gofmt before pushing any amendments (`make format` needs Docker; plain
  `gofmt -w .` from `scraper/` is equivalent for Go).
- Both PR bodies MUST begin with the exact two-line attribution block shown
  in the drafts below (first two lines, unmodified).
- Both PR descriptions include the cross-dependency note.

## 7. Draft PR titles and bodies (ready to paste)

### PR 1 — branch `gcp-family-classification`

**Title:** `gcp: fix missing machine series (C2, C4A/C4D/C4N, N4A, M4N, H4D, X4, M1, A4/A4X, G4)`

**Body:**

```markdown
<!-- ccr-slack-attribution -->
_Requested by **Joe Miller** · [Slack thread](https://planetscale.slack.com/archives/C06K8HH0SM6/p1783639527924329)_

## Problem

Many GCP machine series never show up on the site. **Before:** searching
for `c4d-standard-8`, `c2-standard-8`, `n4a-standard-8`, or
`m1-megamem-96` returns nothing. **After:** all of these appear with
correct on-demand, spot, and CUD pricing.

Two root causes in the GCP scraper:

1. `machineTypeRegex` in `scraper/gcp/api.go` is an explicit allowlist of
   family tokens. Families not in the list (C2, C4A, C4D, C4N, N4A, M4N,
   H4D, X4, M1, A4, A4X, G4) parse to an empty family, collect no core/RAM
   rates, and end up zero-priced — and zero-priced instances are dropped in
   `processGCPData`.
2. C2 and M1 use legacy first-generation SKU names
   ("Compute optimized Core running in ...",
   "Memory-optimized Instance Ram running in ...") that carry no family
   token — and, for C2, no "instance" token — so they failed both the
   family parse and the instance-SKU gate.

## Fix

- Add the missing family tokens to `machineTypeRegex` (ordered so longer
  tokens match before their prefixes, e.g. `c4d` before `c4`).
- Add `legacySKURegex` to admit the legacy C2/M1 SKU names through the
  instance gate and map them to C2/M1 in `parseMachineTypeFromSKU`. M1's
  "Upgrade Premium" surcharge SKUs (how M2 is billed) are excluded so they
  can't pollute M1 baseline rates.

All fixture SKU display names in the new tests were verified verbatim
against Google's live SKU catalog (cloud.google.com/skus/sku-groups).

## Notes

- M2 remains unpriced: it bills as M1 base + "Upgrade Premium" surcharge
  SKUs and needs summing logic — documented follow-up, out of scope here.
- Accelerator families (A4/A4X/G4) price core+RAM only; GPU SKUs are a
  pre-existing limitation unchanged by this PR.
- Cross-dependency: newly-added C4D/C4A `-lssd` shapes additionally need
  the companion bundled-Local-SSD pricing PR to include their bundled SSD
  component; this PR alone still lists them with correct core+RAM pricing.

`go build ./...`, `go test ./...`, and gofmt are clean.
```

### PR 2 — branch `gcp-local-ssd-pricing`

**Title:** `gcp: price bundled Local SSD into -lssd/Z3 shapes and populate capacity fields`

**Body:**

```markdown
<!-- ccr-slack-attribution -->
_Requested by **Joe Miller** · [Slack thread](https://planetscale.slack.com/archives/C06K8HH0SM6/p1783639527924329)_

## Problem

Machine types with bundled Local SSD are billed core + RAM + bundled SSD
capacity, but the GCP scraper only ever priced core + RAM. **Before:**
`c3-standard-8-lssd` shows the exact same price as `c3-standard-8`, and
its `local_ssd`/`local_ssd_size` fields are false/absent — same for every
`-lssd` shape and all of Z3. **After:** the bundled SSD component is folded
into on-demand, spot, and CUD prices, and `local_ssd: true` /
`local_ssd_size` (GB) are populated.

## Fix

- Parse Local SSD usage SKUs ("<FAMILY> Instance Local SSD ...", generic
  "SSD backed Local Storage ...", and their Spot/Preemptible variants) into
  a per-family/region/spot rate bucket. Commitment, reservation-scheduling
  (DWS calendar/flex-start), and suspended-VM-state SSD SKUs are excluded
  from baseline pricing.
- Read bundled capacity from the machineTypes API
  (`bundledLocalSsds.partitionCount`, falling back to the "N local ssd"
  description string). Partitions are 375 GB, except Z3's 3,000 GiB
  Titanium SSD disks.
- Fold `capacity × per-GiB-hour rate` into each shape's price. Families
  where Local SSD is an optional attachment (N1/N2/N2D/C2/...) report zero
  bundled capacity and are entirely unaffected.

All fixture SKU display names in the new tests were verified verbatim
against Google's live SKU catalog (cloud.google.com/skus/sku-groups).

## Notes

- Cross-dependency: C4D/C4A `-lssd` shapes also need the companion
  missing-machine-series PR before they appear at all; this PR alone still
  fixes every already-listed bundled-SSD shape (C3, C3D, C4, Z3, ...).

`go build ./...`, `go test ./...`, and gofmt are clean.
```

## 8. Conflict resolution reference (already done on this branch)

Merging the two branches conflicts in two places; resolved here as:

- `scraper/gcp/scrape.go` — kept the Local SSD interception block (fix 2)
  above the instance gate, AND kept `legacySKURegex.MatchString` in the
  `isInstanceSKU` gate (fix 1). Both are needed.
- `scraper/gcp/api.go` (`parseMachineTypeFromSKU` tail) — kept fix 1's
  legacy C2/M1 fallback block, followed by fix 2's `region = skuRegion(sku)`
  (which subsumes the old inline geo-taxonomy block).

If upstream asks for a rebase after one PR merges, resolve the same way.

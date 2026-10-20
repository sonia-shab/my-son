---
name: h-dashboard-import-test-pitfalls
description: Use when writing h-dashboard import tests. Flaky seeds.
---

# h-dashboard import test pitfalls

## When to Use

- Writing or editing a test under `tests/Feature/PersonImport/`,
  `tests/Feature/HardwareImport/`, `tests/Feature/KargoziniImportLivewireTest.php`
  or `tests/Feature/ImportsLivewireTest.php`.
- A CI run goes red on an import test and the failure looks unrelated to the
  change under review (a count off by one, a random person appearing).
- Debugging a locally-failing import test that passes in CI (or vice versa).


Pitfalls found while fixing PR #1208 (branch `fix/import-bulk-writes-phpstan`)
on `asgarimehdi/h-dashboard`. The import suite is the flakiest part of this
repo; these are the concrete traps.

## Never count imported rows by an n_code PREFIX

`createUserWithUnit()` (tests/Support/Concerns/InteractsWithTestSetup.php)
seeds a `Person` through `Person::factory()`, which generates a **random
ten-digit `n_code`** (`fake()->unique()->numerify('##########')`) and a **random
Persian `f_name`**.

An assertion like:

```php
Person::where('n_code', 'like', '51%')->where('f_name', 'علی')->count()
```

counts that seeded person whenever the draw happens to land on a '51' prefix
AND the name 'علی' — about 1 run in 100 for the prefix alone, 1 in ~670 for
the name, so roughly 1 in 67,000 for both. The count then reports 251 and the
test fails for a reason unrelated to the import. This actually reddened a CI
run.

**Instead, count the codes the sheet actually used:**

```php
$sheetNCodes = array_map(fn (array $row): string => $row['n_code'], $rows);
Person::whereIn('n_code', $sheetNCodes)->count();
```

`array_slice($rows, 250)` selects the second half of the sheet when the first
half is the pre-existing rows.

## Diagnosing a suspected flake from the assertion count

A failing PHPUnit test aborts the remaining assertions of that test. So the
gap between the green assertion total and the failing run's total identifies
**where** the test died:

- bulk person import test: 11 assertions; 4 trailing cache-namespace checks
- green total 424, failing total 420 => died at the row-count assertion
  (4 assertions short), not somewhere later in the file.

Also compare the failing run's **seed** with the file mtimes: a run whose
output file predates the fix commit tested the *old* code, and the flake may
already be fixed. `stat -c '%y' <file>` vs `git log -1 --format=%ci <commit>`
settles it in one command.

## Reproducing a data-dependent flake

`--order-by=random --random-order-seed=N` only fixes the **order** of tests.
Faker data is regenerated per run, so an order seed cannot reproduce a
data-dependent flake. To prove a fix, force the colliding data explicitly in
a throwaway test (e.g. `forceFill(['n_code' => '5199999999', 'f_name' => 'علی'])`
on the seeded person) instead of looping the suite.

## Bulk import writes

`persons.n_code` is unique. A sheet repeating a national code must be deduped
by `n_code` before the bulk INSERT (last write wins) or the chunk aborts on
`persons_n_code_unique`. Hardware has the same hazard on its compare key
("ON CONFLICT DO UPDATE command cannot affect row a second time").

## Local-vs-CI environment drift

- The tracked `.env.testing` ships an **empty** `APP_KEY` by design (issue
  #953). `git checkout -- .env.testing` wipes the locally generated key and
  the whole suite fails with `MissingAppKeyException`. Re-run
  `php scripts/ensure-test-app-key.php` afterwards. That script dirties a
  **tracked** file — never stage it; `scripts/check-app-key-leak.php` guards
  against committing a real key.
- A hardlinked `vendor/` can drift from `composer.lock` (installed laravel
  v13.38.0 vs locked v13.35.0), making local PHPStan report 2
  `unmatched-ignore` errors that CI never sees. Confirm by running the same
  tool on the base branch: identical output there means the branch is clean.

# Infigo Metabase Fork

**Base Release:** `v0.63.15` (upstream commit `e3bbcf84e4fc53b1929341a6e2af0fd7f2bee4f0`)
**Branch:** `infigo_v0_63_15`
**Commit Reference:** `25aba9358f2eeeef83b5236d0b68667d8f206e58`
**Venture Ticket Number:** VENTURE-14575 (originally VENTURE-7985 on `infigo_v0_50_26`)

## Motivation and Reason for Fork

The official Metabase repository was forked to address an issue related to dashboard notification
emails. Specifically, when sending out emails to recipients of dashboard notifications, users from
one group could see other users from another group. This violated our intended privacy restrictions
for group-based visibility.

To fix this, we modified the logic that determines which users are visible as notification
recipients, ensuring that only users within the same group can see each other's information. This
is a security requirement for us.

### Why upstream has not fixed this

The behaviour is controlled by the `user-visibility` setting, declared in
`src/metabase/users/settings.clj`:

```clojure
(defsetting user-visibility
  ...
  :feature      :email-restrict-recipients
  :type         :keyword
  :default      :all)
```

The `:feature :email-restrict-recipients` key gates the setting behind a paid edition. We build the
**OSS** edition (`MB_EDITION=oss`), so the setting can never be changed and always resolves to its
default of `:all` — every authenticated user can enumerate every other user. Upstream considers
this correct for OSS; we do not. The fix therefore has to live in the fork, and is expected to be
re-applied on every version bump.

## Change Details

File: `src/metabase/users_rest/api.clj`, in `defendpoint :get "/recipients"`.

> On `infigo_v0_50_26` this same code lived in `src/metabase/api/user.clj`. Upstream restructured
> `src/metabase/api/*` into per-domain `*_rest` modules between 0.50 and 0.63, so the change was
> re-applied by hand rather than rebased.

### Original Logic

```clojure
(cond
  ;; if they're sandboxed OR if they're a superuser, ignore the setting and just give them nothing or everything,
  ;; respectively.
  (perms/sandboxed-user?)
  (just-me)

  api/*is-superuser?*
  (all)

  ;; otherwise give them what the setting says on the tin
  :else
  (case (users.settings/user-visibility)
    :none (just-me)
    :group (within-group)
    :all (all)))
```

### Updated Logic

```clojure
(cond
  ;; if they're sandboxed OR if they're a superuser, ignore the setting and just give them nothing or everything,
  ;; respectively.
  (perms/sandboxed-user?) (just-me)
  api/*is-superuser?* (all)

  ;; otherwise give them what the setting says on the tin
  :else (within-group)) ; all others → group-mates only
```

#### Summary of the Change

- Removed the `case` branch handling `:none` and `:all` under `:else`.
- Simplified the `:else` branch to always use `within-group`, so group-restricted visibility is the
  default for everyone who is neither sandboxed nor a superuser.

## Rebase History

| Fork | Base | Carried | Dropped |
|------|------|---------|---------|
| `infigo_v0_50_26` | `v0.50.26` | recipient visibility fix (`src/metabase/api/user.clj`) | — |
| `infigo_v0_63_15` | `v0.63.15` | recipient visibility fix (`src/metabase/users_rest/api.clj`) | none |

**Carried-over set for the 0.50.26 → 0.63.15 rebase: 1 change, 0 dropped.**

Nothing was dropped as fixed-upstream: the `user-visibility` logic at `v0.63.15` is byte-identical
in behaviour to `v0.50.26`, only relocated, and the setting is still edition-gated.

`v0.50.26` and `v0.63.15` are **diverged release branches**, not ancestor and descendant — their
merge base is `29a1713df7c` (2024-05-17), and `v0.50.26` carries 1019 commits that `v0.63.15` does
not. Do not merge one into the other. Branch from the upstream tag and re-apply.

## Branch Layout

Each upgrade produces a pair of branches, mirroring the layout established for 0.50.26:

| Branch | Contents |
|--------|----------|
| `v0.63.15` | pristine snapshot of the upstream tag, zero Infigo commits |
| `infigo_v0_63_15` | the above plus the Infigo change set |

`git diff v0.63.15..infigo_v0_63_15` therefore always prints exactly the carried-over set.

Commit: https://github.com/Infigo-Official/metabase/commit/25aba9358f2eeeef83b5236d0b68667d8f206e58

## How to Verify

1. Clone the fork and check out the branch:

   ```bash
   git clone https://github.com/Infigo-Official/metabase.git
   cd metabase
   git checkout infigo_v0_63_15
   ```

2. Confirm the change set is exactly one file:

   ```bash
   git diff --stat v0.63.15..infigo_v0_63_15
   # src/metabase/users_rest/api.clj | 13 +++----------
   ```

3. Inspect the `cond` in `src/metabase/users_rest/api.clj` and confirm it matches the
   "Updated Logic" section above.

4. Runtime check, against a running instance as a non-admin, non-sandboxed user:

   ```
   GET /api/user/recipients
   ```

   The response must contain only users who share at least one group with the caller.

See `LocalSetup.md` for building and running locally.

---

*All other drivers, versions, and unrelated code are not relevant to this change.*

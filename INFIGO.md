# Infigo Metabase Fork

**Commit Reference:** `a9810bb1b8c2372d686e98005cd707f661bc0881`
**Venture Ticket Number:** VENTURE-7985

## Motivation and Reason for Fork

The official Metabase repository was forked to address an issue related to dashboard notification emails.
Specifically, when sending out emails to recipients of dashboard notifications,
users from one group could see other users from another group.
This violated our intended privacy restrictions for group-based visibility.

To fix this, we modified the logic that determines which users receive notifications,
ensuring that only users within the same group can see each other’s information when appropriate with is a requirement for security reasons.

## Change Details

### Original Logic

In the original code, the conditional logic for user visibility was as follows:

```clojure
(cond
  ;; if they're sandboxed OR if they're a superuser, ignore the setting and just give them nothing or everything,
  ;; respectively.
  (premium-features/sandboxed-user?)
  (just-me)

  api/*is-superuser?*
  (all)

  ;; otherwise give them what the setting says on the tin
  :else
  (case (user-visibility)
    :none (just-me)
    :group (within-group)
    :all (all)))
```

### Updated Logic

We updated the conditional clause to change how `:group` behavior is applied. The revised logic is:

```clojure
(cond
  ;; if they're sandboxed OR if they're a superuser, ignore the setting and just give them nothing or everything,
  ;; respectively.
  (premium-features/sandboxed-user?) (just-me)
  api/*is-superuser?* (all)

  ;; otherwise give them what the setting says on the tin
  :else (within-group)) ; all others → group-mates only
```

#### Summary of the Change

- Removed the `case` branch for handling `:none` and `:all` explicitly under `:else`.
- Simplified the `:else` branch to always use `within-group`, ensuring group-restricted visibility by default (unless the user is sandboxed or a superuser).

## Branch Information

The changes are applied on the branch:

```
infigo_v0_50_26
```

You can view the specific commit here:
```
https://github.com/Infigo-Official/metabase/commit/a9810bb1b8c2372d686e98005cd707f661bc0881
```

## How to Verify

1. Clone the forked repository and checkout the `infigo_v0_50_26` branch:

   ```bash
   git clone https://github.com/Infigo-Official/metabase.git
   cd metabase
   git checkout infigo_v0_50_26
   ```

2. Locate and inspect the modified conditional in the appropriate file  `user.clj`

3. You should see the updated `cond` clause matching the “Updated Logic” section above.

---

*All other drivers, versions, and unrelated code are not relevant to this change.*

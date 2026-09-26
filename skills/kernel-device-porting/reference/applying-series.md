# Fetching and applying a patch series to a different base

## Fetch

Patchwork series mbox (`/series/<id>/mbox/`) for the patches; lore
`/all/<cover-msgid>/t.mbox.gz` for the whole thread including reviews.
Always check for a newer version first.

Read the review replies before applying (strip `>` quote lines to skim).
Note outstanding requests and bot findings (e.g. Sashiko AI review) —
they're things to verify on hardware, not settled facts.

## Split and apply in order

Split the mbox one patch per message (python `mailbox`, name by `N/M`).
`git apply` works outside a git repo, so a tarball tree is fine.

- `git apply --check` per patch against an untouched tree gives false
  failures: later patches depend on earlier ones. Apply sequentially
  for real; `git apply` is atomic per patch, so stop at the first
  failure and inspect.

## When a patch fails: find the base, don't hand-edit

1. Print the failing hunk and the same region of the real file.
2. If the difference is style/cleanup, the series is based on newer
   code. Clues: the cover letter's stated base, commit ids cited in
   commit messages ("Since commit 777fd2dab446 ..."), and the file's
   upstream history:
   `gh api "repos/torvalds/linux/commits?path=<file>&per_page=8"`
3. Fetch the prerequisite commits (`github.com/torvalds/linux/commit/<sha>.patch`)
   and apply only the hunks you need:
   `git apply -C1 --include='drivers/<subsys>/<driver>/*' pre.patch`
4. Then apply the rest of the series unchanged.

Record the exact recipe (which prerequisites, which patches needed
reduced context) next to the saved patches, so it's reproducible and
reviewable.

## Reduced context (`-C1`)

Fine when only *surrounding* lines differ (e.g. a downstream fork has
extra nodes nearby) and the inserted block is unchanged. Afterwards,
print the result and check placement and that referenced labels
(`&vreg_l20b`, `&rpmhcc`) exist in this tree. Don't use it to force a
hunk whose changed lines don't match.

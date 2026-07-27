# Canonical flow: working with the filespace

Load this when navigating, reading, or editing files in a LucidLink filespace.

## Session start

- Lead with `list_filespaces` so the choice is explicit (already-linked
  filespaces are marked): if there is exactly one, link it; if there are
  several, ask the user which one(s) before linking. Confirm with
  `current_filespace`.
- Linking is ADDITIVE: linking another filespace keeps existing links live
  and just changes which one is current. File tools always act on the
  CURRENT filespace - when several are linked, state which filespace you are
  acting on whenever you read or write, and check `current_filespace` when
  in doubt.
- Concurrent links are capped (default 4 per account, each holding a ~1 GB
  local cache). `unlink_filespace(name=...)` frees one you no longer need -
  but unlinking drops that filespace's locks and subscriptions, so finish
  and release work there first.
- The server auto-links on the first file operation when the choice is
  unambiguous - but don't rely on that when several exist; an operation will
  just report no/ambiguous link.
- With several configured accounts, `link_filespace` resolves across ALL of
  them and switches account automatically; `use_account` is only for
  changing identity without linking. Address filespaces by short name, id,
  or the unique full name `<filespace>.<workspace>` - use the full name when
  short names collide across workspaces.
- `whoami` shows identity, token status, and what is linked - use it when
  unsure about session state.

## Orientation before work

- Map unfamiliar territory with `tree` (set `max_depth`) or `list_files`.
- Before grepping a large area, gauge it with `count_files`. Walking the
  tree is slower than local disk: always narrow `grep_files` / `find_files`
  with `path_prefix` and `include_pattern`.

## Reading

- Text files: use `read_lines` (plaintext, 1-indexed, numbered). Read the
  range you need, not the whole file.
- `read_file` returns plain text for text content and base64 for binary (an
  `encoding` parameter forces either). Use it for binary content or exact
  whole-file bytes; for text sections prefer `read_lines`.

## Writing and editing

- Edit existing text files with `edit_lines` (line-range replace) or
  `search_replace` (literal or regex). Both accept plaintext and support
  `dry_run=true` - preview any multi-line or repeated replacement first.
- Create new files with `write_file`: pass plain `content` for text, or
  `content_base64` for binary (missing parent directories are created
  automatically; `create_parents=false` to error instead). Appends:
  `append_file`, same two content forms.
- Paths are POSIX-style and absolute, rooted at the filespace: `/dir/file`.
- Every write syncs to the hub automatically - teammates see it within
  seconds. There is no "draft" state: write what is meant to be shared.
- Any edit to an EXISTING path has two preconditions before you write - hold a
  claim AND dry-run multi-line/multi-occurrence changes first. Both live
  together in `editing-safely.md`; load it before any write.

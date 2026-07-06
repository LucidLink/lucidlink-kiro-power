# Editing safely: locking + dry-run

Load this before ANY write, edit, move, or delete on an existing path. The
filespace is multi-writer - humans and other agents may be editing the same
files right now - and edits are not undoable, so two preconditions gate every
write. Both apply to the SAME action; clear one and you still owe the other.

## The two preconditions - check BOTH before every write call

**1. CLAIM.** Do I hold an active claim handle for this path? These tools
require one when the path already exists: `append_file`, `edit_lines`,
`search_replace`, `write_file`, `copy_file` (existing destination),
`move_path`, `delete_path`. Calling any of them on an existing path without a
claim is a protocol violation - there is no "quick edit" or "harmless append"
exemption. The only exemption is creating a path that does not exist yet.

**2. DRY-RUN.** Will this `search_replace` or `edit_lines` change more than one
line or occurrence? If so, I MUST first call the exact same tool with
`dry_run=true` on THIS file, read the preview, and confirm the match count and
context are what I intend - BEFORE the real call. This is not optional and not
satisfied by a dry run on a different file: the same pattern that was safe in
one file can be destructive in the next. If you are about to issue a real
multi-line/multi-occurrence edit and have not just read its dry-run preview,
STOP and dry-run it first.

Before issuing any write tool call, restate to yourself: "I hold claim
<handle> for <path>" and, for multi-line edits, "I dry-ran this and the preview
showed <N> expected changes." If you cannot fill in both blanks, you are not
ready to write.

## The claim protocol

1. Claim: `claim_file(path, mode="exclusive", blocking=false)`. Keep the
   returned `handle_id`.
2. If the result is `BLOCKED`: another writer holds the file. Do not fail and
   do not write anyway. Retry the claim a few times with short pauses; if it
   stays blocked, tell the user who/what might be editing and ask how to
   proceed.
3. Make the edit (dry-run first per precondition 2, then the real call).
4. Release immediately: `release_file(handle_id)`. Never hold a claim while
   doing unrelated work, waiting on the user, or after the edit is done.

Claim facts:
- `list_locks_held` shows only THIS session's locks. Other agents' locks are
  invisible to it - the only way to discover contention is a non-blocking
  claim attempt coming back `BLOCKED`.
- Never use `blocking=true` claims: they stall the whole session indefinitely
  with no way to report progress to the user.
- `claim_file` covers the file's size at claim time; bytes appended afterwards
  are not covered. For a file that may grow during the edit, use
  `lock_byte_range` with an explicit, generous length instead.
- New files need no claim (nothing to collide with until they exist), but
  re-opening and editing them later follows the protocol like any file.

## Multi-file batches

Work file by file: claim → dry-run → check the preview matches intent → apply →
release → next file. Re-run the dry-run for EACH file; never carry one file's
preview over to another. If any preview looks unexpected (wrong count, wrong
context), stop and show the user before applying anywhere else.

## Destructive operations

- `delete_path`: only with the user's explicit confirmation in this
  conversation, even though the host prompts too. State exactly what will be
  deleted (use `count_files` for directories) before asking.
- `copy_file` refuses to overwrite by default - keep it that way. Before
  writing to a path you did not create, check it with `get_entry`.
- `move_path` over an existing destination is an overwrite: check first.

## Zero-knowledge encryption

- File content is end-to-end encrypted; plaintext exists only in this MCP
  session. Do not forward filespace content to external services, logs, or
  other tools unless the task requires it and the user knows.
- Filenames on the hub are HMAC-keyed indexes - but treat names as potentially
  sensitive anyway.

## Resource discipline

- Reads are capped (server-side read cap). Don't slurp large files: use
  `read_lines` ranges, `grep_files` with narrow `path_prefix` and
  `include_pattern`, and check sizes with `get_entry` first.
- Media and other large binaries: operate on metadata and paths; don't read
  content through the MCP unless explicitly needed.
- Do not `link_filespace` to a different filespace, or `unlink_filespace`,
  unless the user asks - switching mid-task silently changes what every path
  refers to.

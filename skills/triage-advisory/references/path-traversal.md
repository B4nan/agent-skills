# Path traversal

Reports in this family claim an attacker-influenced value becomes part of a file path the code reads or writes.

## What to check

- Which part of the path the value controls: a file name, a directory segment, or the whole path. `..` segments defeat both `path.join` and `path.resolve` (`path.join('/base', '../etc/passwd')` → `/etc/passwd`); absolute paths, drive letters and UNC paths defeat only `path.resolve(base, value)`, because a rooted later argument replaces the base (`path.resolve('/base', '/etc/passwd')` → `/etc/passwd`). Either way, check containment on the resolved result.
- Whether the code resolves and contains the result: `path.resolve(base, value)` with `base` itself resolved, then `resolved === base || resolved.startsWith(base + path.sep)` (the separator keeps `/base-evil` from passing a `/base` prefix check), or `rel = path.relative(base, resolved)` rejected when `rel === '..' || rel.startsWith('..' + path.sep) || path.isAbsolute(rel)` (a bare `startsWith('..')` also rejects a legitimate child named `..foo`).
- Symlinks inside the base directory, if the attacker can create them.
- Read vs write. Writes that land on a file the application later loads or executes are the severe case, and pair with any report that controls file *contents* (see the chain section in SKILL.md).

## Telling severity

Who sets the value decides most of these (gate 2 in SKILL.md). Paths taken from developer config or CLI flags on a developer's own machine cross no boundary. Paths built from request data in a running server do.

## Narrowest fix

Resolve, then enforce containment at the single place the path is built, or reject separators, NUL, and the exact names `.` and `..` in values that are meant to be a single file name. Prefer validating the value's shape over trying to sanitise it.

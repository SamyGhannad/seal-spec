# Seal: Bundle Integrity Verification Specification

**Version:** 1.0
**Author:** Samy Ghannad

---

## Problem

AI coding agents load extensions such as skills, hooks, prompts, tool definitions, scripts, templates, and related resources. These files can change how the agent behaves inside whatever permission boundary the host grants: reading files, writing code, invoking tools, running commands, or shaping deployment workflows. There is no mechanism to verify that the extension files on disk match what was reviewed or approved. A compromised extension can redirect tool use, exfiltrate accessible data, inject backdoors, or modify agent behavior silently.

## Terminology

Seal uses these terms:

- **Bundle**: A loadable agent extension with a single top-level identity and a sealed root directory. A bundle may contain skills, hooks, prompts, scripts, templates, tool definitions, MCP configs, assets, and related resources. Bundles are the primary subject of Seal v1.
- **Component**: A file or sub-resource within a bundle, such as a skill, hook definition, prompt, script, or template. Components are tracked as part of their containing bundle. A component does not receive a separate Seal entry merely because the host can use it through that bundle. A component has independent Seal identity only if the implementation can discover and load it directly as its own unit.
- **Project-local loadable unit**: A loadable unit whose sealed root is inside the project root containing `seal.json`. Seal v1 lockfiles authorize the project-local extension surface.
- **Project-external extension**: A loadable unit whose sealed root is outside the project root, such as a user-level, home-directory, organization-level, global, admin, or cache-installed extension. Host-bundled runtime capabilities are outside Seal v1 unless the implementation exposes them as loadable extensions.

In this specification, `plugin` refers to a host-specific kind of bundle. Seal uses `bundle` for the portable concept and uses `plugin` only when discussing host-specific formats.

This is a supply chain problem. `package-lock.json` and `go.sum` solved it for libraries. Seal applies the same proven pattern to AI agent bundles and related loadable artifacts, where the stakes are higher: these files can steer code changes, tool calls, and automation, yet often sit outside normal dependency governance.

## Agent Agnostic by Design

Seal is a specification, not a tool for any specific agent. Any AI coding agent or harness can implement it. If multiple agents load project-local bundles from the same repository, they use the same committed `seal.json` and the same project-relative bundle keys.

## Threat Model

AI agent bundles are external, untrusted inputs to agent behavior and tool use. They may be interpreted, executed, or used to configure a host that has its own sandbox and permission model. Seal treats **every change** to bundle files, whether malicious or legitimate, as unverified until explicitly re-pinned. Implementations detect the change and either block the bundle from loading or alert the user, depending on configuration.

### In Scope

**Supply chain compromise.** A bundle author's account is compromised, or a malicious update is pushed upstream. The bundle files on disk no longer match what was originally reviewed. Seal detects the hash mismatch before the agent loads the bundle.

**Silent updates.** A bundle author pushes a legitimate update that adds a new hook, modifies a system prompt, or changes tool behavior. Without Seal, the agent loads the new code silently. With Seal, the change is detected and either blocked or flagged, depending on the configured policy.

### Out of Scope

**Malicious bundles at first install.** Seal verifies that files haven't changed since they were pinned. It does not assess whether a bundle is safe to install in the first place. First-install review is the user's responsibility.

**Lockfile tampering.** `seal.json` lives in your repository, not the bundle's. If an attacker can modify your lockfile, they already have write access to your codebase. That's a different threat, mitigated by code review and git access controls.

**Runtime behavior.** Seal verifies files at rest. It does not monitor what a bundle does during execution. A bundle that passes verification can still behave maliciously if its reviewed code was malicious to begin with.

## Approach

Cryptographic hashing with a committed lockfile.

Seal is an allowlist for loadable artifacts.

Seal applies to the directly loadable unit. If an implementation loads an enclosing bundle, the bundle is the unit that is classified and verified against `seal.json`. A bundle key MUST NOT be a subdirectory of another bundle key present in `seal.json`. To determine this, implementations MUST evaluate paths using discrete directory segments rather than raw string prefixes. If an artifact is physically nested within a bundle's sealed root, it is a component of that bundle and MUST NOT be sealed as a separate, independent entry. Implementations MUST treat the lockfile as invalid if nested bundle keys exist.

Seal verifies sealed roots, not arbitrary dependency graphs. A Seal entry covers only files under that entry's sealed root. Files outside the sealed root are not hashed, verified, authorized, or blocked by that Seal entry, even if bundle code reads or executes them at runtime.

If an implementation independently discovers and loads another project-local bundle outside that sealed root, it MUST classify and verify that bundle separately. Seal v1 does not require implementations to scan bundle contents for references to external files.

For a given implementation, `seal.json` authorizes only the project-local artifacts that implementation can discover, identify, and load for the current project. If an implementation can discover and load a project-local artifact, it MUST classify that artifact against `seal.json` and apply `policy` before loading it. An implementation MUST NOT silently ignore a discovered project-local artifact that it would otherwise load merely because it cannot map it to a bundle key under this specification; such an artifact is unverified.

Project-external extensions are outside the project lockfile scope of Seal v1. To guarantee the lockfile acts as a strict security boundary, a sealed project is isolated from the host environment: implementations MUST NOT load or execute any project-external extensions. Artifacts the implementation cannot load are outside that implementation's verification surface and do not affect its verification result.

Verification is performed against the installed on-disk directly loadable unit. Implementations MUST hash files from that unit's local sealed root and compare those hashes to `seal.json`. Implementations MUST NOT fetch remote content during verification. Bundle keys in Seal v1 identify project-local sealed roots; they are not instructions to download or resolve bytes from a remote origin.

`go.sum` and `package-lock.json` turned dependency integrity into a project-level artifact: trackable through version control, and enforceable in CI. Seal applies that same pattern to AI agent bundles. Instead of pinning library artifacts for a language runtime, `seal.json` pins the files that make up a loadable agent extension bundle.

**How Seal achieves integrity:**

- `seal.json` records the exact bundle contents a project reviewed and approved
- verification recomputes hashes on disk and detects any added, removed, or modified file
- no key management infrastructure required
- works offline, works in CI, works locally
- the lockfile is the trust anchor, trackable through version control and reviewed in code review

**Why a lockfile over runtime checks:**

- Lockfiles are diffable in code review
- Teams share the same verified state through version control
- CI can gate on verification without running the agent
- Explicit pin/update cycle forces conscious decisions about changes

## Lockfile Location

```
my-project/
├── seal.json
└── ...
```

One lockfile per project: `seal.json` at the project root. Tracked using version control, reviewable in code review, and verifiable in CI. Within a project that has `seal.json`, the lockfile is the authoritative allowlist for loading project-local sealed bundles.

If no `seal.json` is present at the project root, Seal is inactive for that project. Implementations behave normally unless another policy outside this specification requires Seal.

Implementations MAY discover bundles from project, user, organization, bundled, or other locations. Discovery does not imply authorization. In Seal v1, `seal.json` authorizes only project-local loadable units.

For a sealed project, an implementation's verification set is the set of discovered project-local bundles that the implementation can identify and load for that project. Before loading a bundle in that set, the implementation MUST normalize it to its bundle key, classify it against `seal.json`, and apply `policy`.

**Standalone Tool Verification Set:**
Because standalone CLI and CI tools do not have an agent's native loading context, they MUST build their verification set by discovering directories on disk to catch unpinned malicious additions.

A standalone tool determines the loadable surface by combining:

1. **Existing Keys:** The bundle keys already present in the `bundles` object.
2. **Explicit discovery:** Any discovery patterns defined in the optional `discovery` array in `seal.json`.

Existing keys remain part of the verification set so tools can report **Removed** entries when a previously pinned sealed root is absent from disk. However, existing keys do not independently define the insertion fence or bundle-boundary model. In a valid lockfile, every existing key MUST be statically produced by `discovery`; a recorded key that is not statically produced is a lockfile validity error, not a **Removed** or **Drift** state.

The standalone tool MUST scan all filesystem entries matching the `discovery` array. The `discovery` array MUST contain string patterns evaluated against the project root. To ensure deterministic verification sets across different tools, implementations MUST evaluate these patterns using the following strict rules:

- The `*` wildcard matches zero or more characters within a single path segment. It MUST NOT match directory separators (`/`).
- The `*` wildcard MUST match hidden files and directories (names starting with `.`).
- Characters such as `?`, `[`, `]`, `{`, and `}` MUST be treated as literal filename characters. Only the `*` character functions as a wildcard.
- Implementations MUST treat regular files matched by discovery patterns as single-file sealed roots. Directories matched by discovery patterns are directory sealed roots.
- Implementations MUST silently ignore completely empty directories, or directories containing only excluded files, during discovery and bulk pin operations.
- The `**` wildcard (recursive matching across directory separators) is NOT supported in Seal v1. Implementations MUST reject lockfiles containing `**` in a discovery pattern as invalid.
- Patterns MUST use forward slashes (`/`) and MUST NOT start with `./` or `/`.
- Patterns MUST NOT contain repeated separators, `.` or `..` path segments, or a trailing slash. For example, `skills/` is invalid; use `skills` to produce `./skills` as one sealed root, or `skills/*` to produce each direct child as its own sealed root.

Every directory path or regular file path that directly results from this deterministic glob expansion MUST be treated as exactly one independent bundle sealed root.

Tools MUST NOT recursively traverse inside these matched directories looking for further nested bundles. If a nested bundle exists, it is treated as a component of the matched parent directory.

If a directory or regular file matched by the glob expansion does not have a corresponding exact bundle key in `seal.json`, the entire sealed root MUST be flagged as **Unverified**.

To guarantee deterministic verification across different tools and environments (such as CI), standalone implementations MUST NOT use built-in heuristic lists during verification. Tool-maintained heuristics (e.g., checking for well-known directories like `./.agents/*` or `./.claude/*`) MAY be used during authoring workflows (such as initialization or pin commands) to suggest or automatically populate the `discovery` array. This allows agent-specific CLI tools to provide tailored onboarding for their native extension layouts. Because verification strictly follows the written lockfile, differences in initialization heuristics across tools do not affect the deterministic outcome of CI verification. The `seal.json` file remains the sole source of truth for CI discovery.

If an implementation discovers a project-local loadable artifact or bundle that it would otherwise load, but cannot identify its project-local sealed root and derive a bundle key, it MUST treat that artifact as unverified rather than excluding it from the verification set.

User-level, home-directory, organization-level, global, admin, cache-installed, or other project-external extensions are not authorized by `seal.json` in Seal v1. In a sealed project, implementations MUST unconditionally suppress project-external extensions before loading.

Entries in `seal.json` that refer to bundles outside the current implementation's verification set do not affect that implementation's verification result.

This means Seal does secure nested components, but normally through the sealed root of the directly loadable unit. For example, if a bundle contains `skills/review/SKILL.md` and the host loads that skill only through the bundle, the bundle is pinned and the skill is covered by the bundle's `files` and `contentHash`. If the same skill directory is loaded directly as its own project-local unit, that skill directory is pinned as its own entry instead.

Any applicable project-local bundle not in the lockfile is flagged as unverified. Any applicable project-local bundle entry in the lockfile that is not present on disk is flagged as removed.

`Removed` indicates a presence mismatch between `seal.json` and the current machine. Because no bundle code is present on disk to load, `Removed` does not by itself block bundle loading under this specification.

**Example:** A repository may pin both a Claude-oriented project-local bundle and a Codex-oriented project-local bundle in the same `seal.json`. When the project is opened in Codex, Codex verifies the project-local bundles it can identify and load. The Claude-oriented bundle does not become trusted by Codex merely because it appears in `seal.json`, and it does not count as drift for Codex if Codex cannot load it. If Codex discovers a project-local bundle that Codex can load, Codex MUST classify that bundle against `seal.json` and apply `policy` before loading it, regardless of whether the path is under `.codex`, `.agents`, `.claude`, `.opencode`, or another project-local location. Claude applies the same rule to Claude-loadable project-local bundles. This does not permit silent bypass: if Codex encounters a project-local bundle or artifact that Codex would load, but cannot map to a supported bundle key, Codex MUST treat it as unverified.

## File Format

```json
{
  "version": 1,
  "discovery": [
    ".agents/skills/*"
  ],
  "policy": "block",
  "bundles": {
    "./.agents/skills/code-review": {
      "contentHash": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "files": {
        "SKILL.md": "sha256:3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b",
        "scripts/review.sh": "sha256:7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730"
      }
    }
  }
}
```

### Top-Level Fields

| Field       | Required | Description                                                                                                                    |
| ----------- | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `version`   | yes      | Schema version. Currently `1`.                                                                                                 |
| `discovery` | no       | Array of glob patterns defining project-local sealed roots. Used by standalone tools to find unpinned bundles and establish bundle-boundary authority. |
| `policy`    | yes      | Enforcement policy for project-local verification failures (`"block"` or `"warn"`).                                            |
| `bundles`   | yes      | Map of bundle key to bundle entry.                                                                                             |

The `discovery` field is structurally optional so an empty lockfile can omit it. If `bundles` is non-empty, `discovery` MUST be present and MUST statically produce every recorded bundle key.

The `policy` field determines the behavior when unverified or mismatched project-local bundles are discovered. It MUST be either `"block"` or `"warn"`. Removed bundles are reported as drift but do not by themselves block loading. Project-external extensions are always blocked regardless of this policy.

### Lockfile Validity

If `seal.json` is present, it MUST be a valid Seal lockfile before any bundle verification begins.

A lockfile is invalid if any of the following are true:

- the file is not valid JSON
- the file contains unrecognized fields or properties at any level
- a `discovery` pattern contains the unsupported `**` wildcard, starts with `./` or `/`, uses backslashes (`\`), contains repeated separators, contains `.` or `..` path segments, or has a trailing slash
- two or more `discovery` patterns can produce sealed roots where one produced root is a directory-segment ancestor of another produced root
- `version` is missing, not an integer, or not supported by the implementation
- required top-level fields are missing or have the wrong type
- a bundle key is duplicated after parsing or normalization
- a bundle key is invalid under this specification
- a bundle key is not statically produced by any `discovery` pattern
- a bundle key is a directory-segment ancestor of another bundle key
- a required pin field is missing or has the wrong type
- a recorded hash is not in the required `sha256:<lowercase-hex>` format
- a recorded path is invalid under this specification
- a bundle's recorded `contentHash` does not mathematically match the aggregate hash computed from its recorded `files` map

A bundle key is statically produced by a `discovery` pattern if evaluating that pattern against a filesystem containing the corresponding path would produce that exact bundle key as a sealed root. Static production is evaluated without reading the filesystem.

Static production is about bundle boundaries, not recursive hash coverage. For example, `discovery: ["skills"]` produces the bundle key `./skills`. That bundle recursively hashes files under `skills/`, so additions such as `skills/foo` are caught as a **Mismatch**. However, `skills` does not statically produce the separate bundle key `./skills/foo`; `skills/*` does.

Static production MUST be evaluated for every recorded bundle key, including keys whose sealed root is not present on disk and would otherwise be classified as **Removed**. For example, `skills/*` statically produces `./skills/foo` even when `skills/foo` is absent.

Discovery patterns can also be invalid in relation to each other. A lockfile is invalid if two discovery patterns can produce sealed roots where one root is a directory-segment ancestor of the other. For example, `skills` and `skills/*` are invalid together because they express incompatible bundle boundaries for the same subtree. Exact duplicate production is not a nested-root error and MAY be deduplicated by implementations.

Implementations SHOULD make static-production failures actionable. Diagnostics SHOULD name the uncovered bundle key, explain which discovery pattern shape would produce it, and distinguish whole-root discovery from per-child discovery. For example, if `bundles` contains `./skills/foo` but `discovery` contains only `skills`, a CLI SHOULD report that `skills` produces `./skills` as one recursive bundle and does not produce `./skills/foo`; use `skills/*` for per-child bundle roots, or re-pin `skills` as the single recorded bundle.

If `seal.json` is invalid, the implementation MUST fail closed for the project: it MUST NOT load or execute any project-local bundle, project-local loadable artifact, or project-external extension that would otherwise load for that project.

This failure is independent of `policy`. If the lockfile is invalid, the implementation cannot trust `policy`, so it MUST use the strictest behavior. The policy controls the handling of unverified or mismatched project-local bundles only after a valid lockfile has been loaded.

### Companion JSON Schema

An official JSON Schema is provided in `seal.schema.json` as an informative companion for tooling, editor support, and structural validation.

A lockfile that fails the official JSON Schema is invalid. Passing the schema is necessary but not sufficient for conformance.

The prose in this specification remains normative. Conforming implementations MUST enforce all normative requirements in this specification, including requirements not fully expressible in JSON Schema such as path normalization, duplicate keys after normalization, static production by `discovery`, nested bundle-key rejection, overlapping discovery-pattern rejection, recomputing `contentHash`, bundle-key derivation, and verification semantics.

### Lockfile Serialization

Because `seal.json` is tracked in version control and reviewed by humans, identical lockfile states MUST produce identical byte-for-byte files on disk to prevent Git diff churn.

While JSON parsers MUST treat objects as unordered, tools that write or update `seal.json` MUST serialize it deterministically:

1. **Encoding:** UTF-8 without a Byte Order Mark (BOM).
2. **Line Endings:** All line breaks MUST be a single Line Feed character (`\n` / `U+000A`). Carriage Returns (`\r`) MUST NOT be used.
3. **Trailing Newline:** The file MUST end with exactly one `\n`.
4. **Indentation:** Nested levels MUST be indented using exactly two Space characters (`U+0020`) per level. Tabs MUST NOT be used. Empty objects and arrays (`{}` and `[]`) MUST be serialized inline without internal whitespace or line breaks.
5. **Key Ordering:**
   - Top-level keys MUST be written in this exact order: `"version"`, `"discovery"`, `"policy"`, `"bundles"` (omitting `"discovery"` if it is not present).
   - Keys within a bundle entry object MUST be written in this exact order: `"revision"`, `"contentHash"`, `"files"` (omitting `"revision"` if it is not present).
   - Keys within the `"bundles"` object, and keys within each bundle's `"files"` object, MUST be sorted lexicographically by their UTF-8 byte sequence before writing.
6. **Spacing:** A single space MUST follow the colon `:` separating keys and values. Arrays and Objects MUST NOT have trailing commas.

### Bundle Entry Fields

| Field         | Required | Description                                                                                                                                            |
| ------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `revision`    | no       | Informational metadata only. A source-specific version identifier recorded at pin time, e.g. git commit SHA. It is not part of integrity verification. |
| `contentHash` | yes      | Aggregate SHA-256 hash of all tracked files. See [Content Hash](#content-hash).                                                                        |
| `files`       | yes      | Map of relative file paths to their individual `sha256:<hex>` hashes.                                                                                  |

`revision` is advisory metadata for auditability, review, and pin workflows. It MAY be recorded, displayed, or updated by tooling, but Seal authorization is determined by the bundle key, `files`, and `contentHash`, not by this field.

If an implementation can derive `revision` for an installed unit and finds that the derived value differs from the value recorded in `seal.json`, it MAY report that difference as diagnostic metadata drift. Such a difference MUST NOT by itself change the verification result, make the lockfile invalid, or authorize content that would otherwise fail verification.

### Bundle Key

Format: `./<normalized-project-root-relative-path-to-sealed-root>`

A bundle key identifies the project-local sealed root of a directly loadable unit. Seal v1 supports project-local directory roots and regular-file roots: if an implementation would otherwise load a project-local directory or regular file as a bundle, skill, plugin, prompt, command, rule, instruction, or other extension unit, that path is the sealed root. The bundle key does not create separate lockfile identities for nested components that are loaded solely as part of a containing bundle.

Path normalization ensures that independent implementations produce the same key for the same project-local sealed root:

| Input                                   | Normalized                |
| --------------------------------------- | ------------------------- |
| Local bundle at `plugins/foo`           | `./plugins/foo`           |
| Local bundle at `./plugins/foo/`        | `./plugins/foo`           |
| Local bundle at `.agents/skills/review` | `./.agents/skills/review` |

The normalization rule is: use the sealed root's normalized project-root-relative path, prefixed with `./` and using forward slashes (`/`). Seal v1 does not support the project root itself as a sealed root; the exact bundle key `.` is invalid. Bundle keys MUST NOT contain `..`, repeated separators, or a trailing slash.

A bundle key identifies the local sealed root within the current project, not the bundle's upstream provenance. During verification, the implementation derives the bundle key from the project-local sealed root it would otherwise load, then hashes that sealed root. If an implementation would otherwise load a project-local unit but cannot identify its sealed root as a project-local directory and derive a bundle key for it, it MUST treat that unit as unverified.

### Content Hash

Aggregate hash computed deterministically from per-file hashes:

1. Normalize each tracked path to a sealed-root-relative path that uses forward slashes (`/`). For a regular-file sealed root, the sole tracked path is `.`.
2. Sort the normalized paths lexicographically by their UTF-8 byte sequence
3. For each file, concatenate the normalized path, a colon, and the file's hash (e.g. `SKILL.md:sha256:3a7b...`)
4. Join all entries with a single Line Feed character (`\n` / `U+000A`)
5. Encode the resulting joined string as UTF-8 bytes
6. Compute the SHA-256 hash of those UTF-8 bytes

```
contentHash = SHA-256(UTF8_ENCODE(sort(normalizedPaths).map(k => k + ":" + files[k]).join("\n")))
```

The newline delimiter prevents concatenation collisions between entries. The content hash changes if any file is added, removed, or modified.

### Canonicalization Rules

To ensure independent implementations produce the same lockfile, Seal defines the hash input precisely:

- **File content hashing:** Per-file hashes are computed over the raw file bytes exactly as read from disk. Implementations MUST NOT normalize line endings, decode text, strip whitespace, parse structured files, or otherwise transform the content before hashing.
- **Hash encoding:** All hashes use SHA-256, encoded as lowercase hexadecimal, prefixed with `sha256:`.
- **Out-of-scope metadata:** Seal v1 hashes file contents only. File metadata, including permissions, executable bits, ownership, timestamps, ACLs, extended attributes, and similar filesystem metadata, is out of scope for integrity verification. Differences in such metadata MUST NOT by themselves change the verification result.
- **Path format:** Paths in `files` are relative to the sealed root and MUST use forward slashes (`/`) on all platforms.
- **Unicode and Case:** Implementations MUST normalize all path strings to Unicode Normalization Form C (NFC) before sorting, hashing, or writing to the lockfile. Paths MUST be treated as strictly case-sensitive.
- **Path normalization:** Before recording or comparing a path, implementations MUST remove any leading `./`, collapse repeated separators, and reject absolute paths. Implementations MUST reject any path containing `..` segments, or `.` segments used as intermediate directories (e.g., `foo/./bar`). The exact bundle key `.` is invalid in Seal v1. The exact file path `.` is permitted only as the sole `files` entry for a regular-file sealed root.
- **Internal consistency:** A lockfile is internally inconsistent and invalid if any bundle's `files` map contains path keys that collapse to identical normalized paths, or if a bundle's recorded `contentHash` does not equal the cryptographic aggregate of its recorded `files` map.
- **Tracked entries:** Directories are not hashed directly. Only tracked file entries appear in `files`. A bundle MUST contain at least one trackable file. Implementations MUST reject pinning or updating a bundle if it results in an empty `"files"` map, and a lockfile is invalid if any bundle's `"files"` map is empty.
- **Unsupported file types:** Symlinks and other non-regular files are not supported in v1. Implementations MUST NOT follow, dereference, or silently skip these entries. If an unsupported file is encountered during `verify`, the implementation MUST classify that bundle as a **Mismatch**. If encountered during `pin`, the implementation MUST abort the operation and leave `seal.json` unmodified.

### What Gets Hashed

For directory sealed roots, all regular files in the bundle directory tree are hashed, with the following exclusions:

1. Any file or directory exactly named `.git` (at any depth within the sealed root) is always excluded, along with its contents. This ensures version-control metadata (including submodules and worktree pointers) does not cause hash mismatches.
2. The `seal.json` lockfile is excluded, but ONLY if the bundle's sealed root contains the lockfile (to prevent circular hashing dependencies).

Verification hashes are always computed from the project-local files under the sealed root. For example, an entry keyed as `./.agents/skills/review` is verified by hashing the project-local bundle at `.agents/skills/review`, not by downloading files from GitHub or any other remote origin at verification time.

For regular-file sealed roots, the file itself is hashed as the sole tracked entry and recorded in `files` under the path `.`. For example, a project-local file root at `skills/review.md` is represented by bundle key `./skills/review.md` and `files: { ".": "sha256:..." }`.

Seal treats the bundle directory as a sealed content root. Bundle authors and implementations MUST store runtime-generated state (for example caches, logs, downloads, temp files, or generated artifacts) outside the bundle root.

Files outside the sealed root are outside Seal v1 protection unless they are covered by another applicable Seal entry. Bundle authors SHOULD keep behavior-affecting files inside the sealed root.

Host and runtime dependencies are outside the sealed content root for v1. This includes, for example, the agent runtime, `/bin/sh`, interpreter binaries, and system libraries, unless the bundle vendors those files inside its own sealed root.

When a bundle is the directly loadable unit, all supported components within its sealed root are covered by that bundle's hashes. Nested components are not excluded from verification; they are verified through the containing bundle unless they are also loaded directly as independent units.

If a bundle writes runtime state into its own root after pinning, those changes are treated like any other added or modified file and cause verification to fail until the state is removed or the bundle is re-pinned.

Everything else is tracked, including dependency directories (`node_modules`, `vendor`, etc.). If a bundle ships dependencies, changes to those dependencies are detected. Bundle authors who want clean diffs should not ship dependency directories.

Seal v1 intentionally prefers a simple fail-closed rule over format-specific exclusion lists. This may create operational noise for bundles that vendor dependencies or write mutable state into their sealed root. That noise is a deployment and packaging concern, not an integrity exception. Implementations MAY provide guidance or tooling to reduce review friction, but MUST NOT silently exclude files under the sealed root from verification unless this specification explicitly allows it.

Verification detects three types of changes:

- **Modified file:** file exists on disk and in the lockfile, but the hash differs.
- **Added file:** file exists on disk but not in the lockfile.
- **Removed file:** file exists in the lockfile but not on disk.

All three cases produce a content hash mismatch. If a tracked file cannot be read during the hashing phase (e.g., an OS read error because the file was deleted after directory traversal), the implementation MUST treat that read failure as a **Modified file** rather than crashing. False positives (noisy diffs on update) are better than false negatives (missing a compromised file).

## Workflow

Any change to `seal.json` should be tracked using version control as part of the corresponding project change.

```
pin project-local bundles → verify before load → review changes → pin
```

An implementation MAY verify on session start to surface mismatches early, but session-start verification alone is not sufficient. To provide time-of-check/time-of-use protection, an implementation MUST verify a bundle immediately before loading or executing any code, prompt, hook, script, template, tool definition, or other loadable resource from that bundle.

## Verification Outcomes

Verification compares two sets for the current project and implementation:

- the applicable project-local bundles installed on disk
- the applicable project-local bundle entries recorded in `seal.json`

An installed bundle is applicable if the current implementation can identify it and would load it for the current project, and its sealed root is inside the project root.

A lockfile entry is applicable if its bundle key refers to a project-local sealed root that belongs to the current implementation's loadable surface. A lockfile entry can be applicable even when its sealed root is not present on disk; in that case it is classified as **Removed**.

Project-external extensions that would otherwise load MUST be suppressed. Because they are unconditionally blocked, they do not affect the verification state of the project.

If an implementation supports enabling or disabling bundles, disabling a bundle does not remove it from verification while it remains installed on disk and belongs to that implementation's project-local loadable surface. Disabled bundles can be re-enabled later or referenced by other loadable units, so they remain subject to Seal verification.

If the lockfile itself is invalid, verification stops before bundle classification and the implementation MUST fail closed for the project.

Each applicable bundle then falls into one of these states:

| State          | Description                                                                                                                                                    |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Verified**   | The bundle is present on disk, present in `seal.json`, and all tracked hashes match.                                                                           |
| **Unverified** | The bundle is present on disk but has no entry in `seal.json`.                                                                                                 |
| **Removed**    | The bundle has an entry in `seal.json` but is not present on disk. `Removed` indicates a presence mismatch, not an executable integrity failure.               |
| **Mismatch**   | The bundle is present on disk and in `seal.json`, but its current contents do not match the lockfile or it cannot be verified according to this specification. |

After classifying states, the implementation applies policy:

- `policy` controls project-local **Unverified** and **Mismatch** states.
- **Removed** entries are reported as drift and do not by themselves block loading.

Verification produces one of four overall results:

| Outcome      | Description                                                                                                                                                                                                                                                                                                                         |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Verified** | Every installed project-local bundle is verified, and there are no unverified, removed, or mismatched project-local bundles.                                                                                                                                                                                                        |
| **Drift**    | Every installed project-local bundle is verified, but one or more project-local bundle entries recorded in `seal.json` are not present on disk. The user is alerted. Removed bundles do not by themselves block loading.                                                                                                            |
| **Warning**  | One or more project-local bundles are unverified or mismatched while `policy` is `"warn"`. The user is alerted, but those bundles may still be loaded. Removed bundles are also reported, but do not affect blocking behavior.                                                                                                      |
| **Blocked**  | One or more project-local bundles are unverified or mismatched while `policy` is `"block"`, or the implementation was unable to suppress project-external extensions. Any unverified or mismatched project-local bundles MUST NOT be loaded or executed. Removed bundles are also reported, but do not by themselves block loading. |

The `policy` field in `seal.json` determines the behavior for unverified or mismatched project-local bundles. If an implementation provides its own override (e.g. through project or organization policy), the strictest value wins. A separate policy MAY require the presence of specific bundles, but presence enforcement is outside the scope of v1.

## Implementation Notes

**For agent developers adding native support:**

1. Verify each project-local bundle immediately before loading any code, prompt, hook, script, template, tool definition, or other loadable resource from that bundle.
2. Apply `policy` to unverified or mismatched project-local bundles.
3. Unconditionally suppress all project-external extensions.
4. Expose pin and verify workflows.

**CLI Tool Behaviors:**
To guarantee identical operation across different implementations, standalone CLI tools MUST enforce the following behaviors:

- **Execution Context:** Standalone CLI tools MUST be executed from the project root containing `seal.json`. If `seal.json` is not present in the current working directory, the tool MUST fail and MUST NOT traverse parent directories to locate it.
- **Output Streams:** Implementations MUST write all diagnostic alerts, interactive prompts, warnings, and human-readable text to standard error (`stderr`). Standard output (`stdout`) MUST be reserved exclusively for requested machine-readable output (e.g., the `--json` flag).
- **Missing Lockfile:** If `seal.json` is not present at the project root, standalone CLI tools MUST exit with code `2` (Fatal Error) rather than silently succeeding, to prevent accidental lockfile deletion from bypassing CI checks.
- **True-Case Path Resolution:** Because paths in Seal are strictly case-sensitive, implementations MUST NOT rely on case-insensitive OS fallback resolution to verify or pin a bundle. The case of a bundle key in `seal.json` MUST exactly match the true string case of the directory entry as reported by the filesystem. If the lockfile contains `./plugins/MySkill` but the filesystem directory listing reports `myskill`, the implementation MUST treat the lockfile key as **Removed** and the discovered directory as a distinct, **Unverified** bundle.
- **Verification Output:** Tools MUST provide a mechanism (e.g., a `--json` flag) to output detailed verification results in a machine-readable JSON format, allowing downstream CI systems to deterministically parse failure reasons. When requested, this JSON output MUST be written to `stdout`. The output MUST be a JSON object containing a `"status"` string matching one of the overall outcomes (`"Verified"`, `"Drift"`, `"Warning"`, or `"Blocked"`). The object MUST also contain arrays of bundle keys for the `"unverified"`, `"removed"`, and `"mismatch"` states. The implementation MUST always include these three keys, mapping them to empty arrays `[]` if no bundles are in that state. The object MAY optionally include a `"verified"` array of bundle keys. Implementations MAY include additional diagnostic metadata within the output object. If the CLI exits with code `2` (Fatal Error), it SHOULD NOT output JSON to `stdout`.
- **Pin Scope and Semantics:** The `pin` command acts as the unified mechanism for recording authorized state. It MUST support the following operational modes:
  - **Targeted Pin (`pin <path>...`):** If invoked with positional arguments, the implementation MUST first resolve each path to its true filesystem string case before normalizing it to a bundle key, to prevent case-insensitive OS fallbacks from recording incorrect keys.

    If the resulting key is inside an existing recorded sealed root, the tool MUST reject the operation because it would create a nested bundle key; the user must re-pin the containing root or intentionally change the discovery boundary first. If the resulting key does not exist in `seal.json`, it is added as a new entry. If no existing `discovery` pattern statically produces the resulting key, the tool MUST add the narrowest exact discovery pattern that statically produces that key in the same lockfile update.

    If the key already exists, the implementation MUST recalculate its hashes. If the newly computed `contentHash` differs from the recorded one, the tool MUST prompt for confirmation before updating the entry.

  - **Bulk Pin (`pin` with no arguments):** It MUST evaluate the `discovery` array to find unpinned sealed roots and add them. It MUST also recalculate and update hashes for all bundle keys currently in `seal.json` that are present on disk.

    Before writing, the tool MUST validate that every recorded bundle key is statically produced by `discovery`, that no recorded bundle key is a directory-segment ancestor of another recorded bundle key, and that no pair of discovery patterns can produce roots where one root is a directory-segment ancestor of the other. If the discovery patterns would cause bulk pin to materialize nested bundle keys, the tool MUST abort and leave `seal.json` unmodified.

- **Pruning and Removal:** By default, if a bundle key recorded in the lockfile no longer exists on disk (the Removed state), the implementation MUST preserve its existing lockfile entry unmodified. Tools SHOULD provide a mechanism (e.g., a `--prune` flag) to explicitly remove these missing keys from `seal.json` during a bulk pin operation.
- **Interactive Supervision:** Pinning is strictly a user-supervised task. For any operation that modifies `seal.json` (adding new keys, updating hashes, or pruning), the tool MUST summarize the changes and prompt for explicit user confirmation. Implementations MUST NOT provide a mechanism (e.g., `-y` or `--force`) to bypass this confirmation. If the tool is executed in a non-interactive environment (such as CI) and a mutation to `seal.json` is required, the tool MUST abort with code `2`.
- **File Locking:** During any write operation (`pin`), the implementation MUST acquire an exclusive file lock on `seal.json` (or use atomic filesystem renaming) to prevent corruption when multiple agents or CI jobs attempt concurrent writes.
- **Read Errors on Write:** If an OS read error (e.g., permission denied) occurs on any tracked file during a `pin` operation, the tool MUST abort the operation and leave `seal.json` unmodified.
- **Strict Schema Adherence:** Implementations MUST NOT silently strip or preserve unknown fields during a pin operation; if an existing lockfile contains unknown fields, it is invalid and the tool MUST exit with code `2`. When a pin operation initializes a new `seal.json` file, the tool MUST set `"policy"` to `"block"` by default.

**Reference tooling:**

A standalone verification tool is planned to enable adoption independent of agent support. Teams will be able to pin and verify `seal.json` via CLI or CI, without waiting for their coding agent to implement Seal natively.

**CLI Exit Codes:**

To ensure continuous integration (CI) systems can reliably gate on verification regardless of the underlying tool, implementations providing a command-line interface MUST use the following POSIX exit codes:

| Code    | Name                | `verify` semantics                                                                   | `pin` semantics                                                                                                                          |
| ------- | ------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **`0`** | Success / Permitted | Outcome is **Verified**, **Drift**, or **Warning**. Not blocked by policy.           | Operation completed successfully and `seal.json` was updated, or no changes were required.                                               |
| **`1`** | Blocked             | Outcome is **Blocked**. Unverified or mismatched bundles found under "block" policy. | _(Not used)_                                                                                                                             |
| **`2`** | Fatal Error         | `seal.json` is missing or invalid, or a system error occurred.                       | `seal.json` is invalid, the operation was aborted due to read errors or unsupported files, or confirmation was required but unavailable. |

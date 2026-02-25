# DeepLake OS — Technical Specification & Vision Document

## References

Special thanks and consideration is due to the following people who have paved this path over countless times.

Thank you Nayuki of the RelFS Whitepaper, the ideas presented here fascinated me for an entire summer.

Special Thanks to Hilel Cooperman, Quentin Clark, Hadi Partovi, and so many more WinFS team members. Your enthusiasm for the project shines through to this day in archival blog posts and video.

Special thanks to the original Be engineering team and the team at Haiku for their open source implementation of BeFS.

Special thanks to the team that toiled on OLE during Cairo development. It's your road Microsoft paves over every time it tries again.

Special thanks to the KDE team for the NEPOMUK Personal Information Manager Desktop which shows how powerful next gen software can be with the proper tech underlying it.

Thank you to Linus Torvalds for GIT and the Linux kernel as we know it.

Special thanks to the Bup team for their take on content chunking helping me tie the whole idea together.

Special thanks to the team building LakeFS! You guys rock!

Special thanks to the Palm WebOS team and the people responsible for UniversalSearchManager. Extra Special Thanks to the executives at HP who approved WebOS OSE.

---

## Executive Summary

DeepLake OS is a Linux distribution built around a novel filesystem that combines three ideas never before unified at the OS level:

1. **Content-addressed hash lake** (Git's object model as the root filesystem)
2. **Relational semantic metadata with computed views** (WinFS reimagined)
3. **Full version history as a first-class filesystem feature** (Git's commit DAG)

An optional AI enrichment layer — local or cloud — automatically tags, links, and adds semantic meaning to user files. The POSIX directory hierarchy is a computed illusion projected from the lake, enabling per-process virtual namespaces, seamless app sandboxing, and instant rollback of any app or the entire system.

The POSIX filesystem was born on the PDP-11 — a machine with 256KB of RAM. Its constraints became Unix's design, Unix's design became a standard, and the standard went unchallenged for over fifty years. Every modern feature — versioning, search, metadata, cloud sync, sandboxing, backup, deduplication, integrity — has been bolted onto a model designed for single users on PDP-11 hardware. DeepLake inverts this: POSIX is a compatibility shim projected from a modern foundation, not the foundation itself.

---


## 1. The Hash Lake (On-Disk Foundation)

At the absolute root there is no directory tree. There is a flat, content-addressed store where every file is a BLAKE3 hash of its contents, stored once.

```
/lake/
  b3:7f2a91c3e4...  → raw bytes
  b3:a83bf20d11...  → raw bytes
  b3:55e901aacf...  → raw bytes
```

Everything above this is a view — a computed projection of the lake.

The lake is content-agnostic. It stores bytes. It does not know or care whether something is a text file, an image, a video, or a database. Chunking, hashing, versioning, and diffing all operate on raw bytes. Content type awareness is a metadata layer concern. The lake's agnosticism is a feature, not a limitation.

### On-Disk Layout

```
Disk:
├── /dev/sda1  —  ESP (FAT32, 512MB)
│                  UEFI bootloader, optionally kernel+initramfs
├── /dev/sda2  —  /boot (ext4, 1GB) [optional with direct UEFI boot]
│                  Kernel, initramfs containing varvefs.ko
└── /dev/sda3  —  Lake (varvefs, rest of disk)
                   ├── Superblock + boot-critical cache
                   ├── Object store (the hash lake)
                   ├── Metadata store (relational B-trees)
                   ├── WAL / Journal
                   └── Free space map
```

### Superblock

- Magic number, version, BLAKE3 parameters
- Pointer to metadata B-tree root
- Pointer to free space bitmap
- Lake statistics (object count, total size)

### Object Store

Every object on disk carries a single-byte type prefix (not included in the hash):

| Prefix | Type | Description |
|---|---|---|
| `0x01` | Chunk | Raw byte content |
| `0x02` | Manifest | JSON-serialized ordered list of chunk references |
| `0x03` | Commit | Global DAG for point-in-time snapshots (future) |
| `0x04` | Tree | Directory snapshots (future) |
| `0x05` | Tag | Named references (future) |

The hash is always of the pure content. The prefix is added at write time and stripped at read time by the object store. Nothing outside the store ever sees it.

```
On disk:    [type_prefix: 1 byte][content: N bytes]
Hash:       BLAKE3(content)  — prefix excluded from hash
```

Content-addressed blocks are organized in a B-tree keyed by BLAKE3 digest. Large objects are chunked with content-defined chunking (FastCDC) for sub-file deduplication. FastCDC defines chunk boundaries; BLAKE3 fingerprints each resulting chunk (it is not the rolling chunker). Reference counting supports garbage collection.

### Manifests as the File Abstraction

A "file" in the lake is a manifest — an ordered list of chunk references whose own identity is the BLAKE3 hash of the concatenated chunk hashes. The manifest is what the FUSE layer resolves, what the version chain tracks, and what diffing compares.

Manifests store `ChunkRef { hash, size }` rather than bare hashes. This makes manifests self-contained for offset calculation, diffing, and file reassembly without hitting the object store for chunk metadata.

Only the most recent manifest (HEAD) is visible through the normal filesystem view. Historical manifests exist in the lake and are browsable through the virtual history namespace or the CLI.

### Metadata Store

Embedded relational database baked into the filesystem format:

- Tables: objects, types, properties, relationships, views
- B-tree indexes on type, property values, relationship edges
- Every object hash has one or more typed metadata records

### View Definitions

Stored in the metadata layer. A "directory" is a saved query: "objects where parent_view = X, ordered by name." The POSIX hierarchy is a default view, auto-maintained.

### Journal / WAL

Write-ahead log covering both the object store and metadata store. Atomic crash consistency across both layers.

### Inherent Properties

- **Automatic total deduplication.** Same content = same hash = one copy.
- **Moves and renames are metadata-only.** Instant regardless of file size.
- **Built-in integrity.** Every read verifiable against its BLAKE3 hash. Silent data corruption detectable by default.
- **Trivial diff-based backups.** Sync the delta of the lake plus metadata changes.
- **Optional hierarchy.** Power users abandon folders entirely; traditional users see exactly what they expect.

### Immutability Guarantees

A critical distinction: the hash provides verification and identity, not immutability. The append-only write policy of the object store provides immutability.

BLAKE3 guarantees that if content is present, it has not been tampered with. Any bit flip produces a completely different digest — corruption is immediately and mathematically detectable. But the hash is a one-way function; content cannot be reconstructed from a hash alone.

Immutability is preserved by the policy that the lake never overwrites existing blobs. Old blobs remain exactly where they are until diagenesis (garbage collection) explicitly reclaims them through deliberate GC with confirmed zero reference counts.

The immutability guarantee is therefore policy-dependent rather than mathematically absolute. Users should understand:

- What protects their history (append-only store + BLAKE3 verification)
- Under what conditions that protection can be explicitly waived (diagenesis policy)
- What the hash alone can and cannot do

### WORM Storage Properties

The lake is fundamentally write-once-read-many at the storage layer with a thin mutable index on top. This gives:

- **Tamper evidence** — any modification to stored content breaks its hash
- **Full audit trail** — the version table is append-only
- **Guaranteed recoverability** — old versions are manifests that still exist pointing at chunks that still exist

The mutability users experience is an illusion projected by the path table and view layer. For enterprise and compliance contexts, disabling garbage collection creates a true append-only audit log of every file state that ever existed.

### Corruption Handling

Detection is immediate and free — BLAKE3 verification occurs at read time. Silent data corruption is impossible; corruption is made loud at the moment of first read.

**Recovery Hierarchy:**

1. **Chunk exists elsewhere in the lake** via deduplication — automatic transparent repair, no user involvement.
2. **Chunk exists on a remote lake** — fetch, verify against hash, repair. The hash makes retrieval trustworthy regardless of source.
3. **Chunk is unique and unrecoverable** — genuine worst case. Fall back to most recent verified micro-hash, which in normal operation is seconds behind the lost state.

The lake can never silently serve corrupted data. That guarantee holds regardless of recovery outcome.

For comparison: mainstream filesystem corruption worst case involves silent corruption running for weeks, discovered when backups are also corrupted, requiring professional data recovery at significant cost with no success guarantee. DeepLake worst case is loss of seconds of work on a unique chunk on an isolated single-drive machine with no remote. Detection is immediate, scope is limited to exactly the affected chunk, and recovery tooling is built in.

---

## 2. User-Afforded Expectation Simulation (The View Engine)

What users think of as "files in folders" is one possible projection of the lake. The relational metadata layer constructs whatever reality the user or application expects.

- **POSIX view:** Synthesized traditional `/home/user/Documents/report.pdf` hierarchy, generated on-the-fly. `ls`, `cat`, `open()` all work. Apps don't know the difference.
- **Semantic view:** "All photos from June 2025 involving Sarah" — not a search, a mount point.
- **App-native views:** Email client sees mailboxes. Music player sees albums. Code editor sees projects. All pointing to the same underlying hashes.
- **Temporal views:** "My filesystem as it looked last Tuesday." Nearly free since the lake is content-addressed and append-friendly.

POSIX is a view. The directory hierarchy is computed from the path table. It is one possible projection of the lake.

---

## 3. Git-Native Version History

The filesystem is a Git-like object store. Every file has full history by default — not through whole-volume snapshots but per-object, via a commit DAG in the metadata layer.

### The Path Table (The Only Mutable State)

Everything in the lake is immutable and append-only. The sole mutable component is the path table, which maps POSIX paths to HEAD manifest hashes.

Schema (two tables):

- **paths** — `path → head_manifest, created_at, modified_at, deleted`
- **versions** — `path, manifest_hash, version_num, timestamp, trigger, message`

Separate tables because version history is append-only and queried differently from path lookups.

### Per-File Sequential Version Numbers

Each file has an independent counter: v1, v2, v3. Incremented atomically on every promoted commit. The DAG uses hashes internally — the version number is a display/index alias for human-friendly access.

This enables queries like:

- `lakectl diff /path v2 v5` — diff any two versions
- `lakectl cat /path v3` — read content at a specific version
- `lakectl rewind /path v2` — rewind (creates a new version entry, non-destructive)

### Soft Delete (Detritus)

Deleting a file sets `deleted = 1` on the path entry. History is preserved. The file disappears from normal directory listings but remains visible in the virtual history namespace and restorable via `lakectl restore`. The cost is a `WHERE deleted = 0` on every FUSE lookup — trivial.

The lake philosophy: nothing is lost unless explicitly destroyed through deliberate garbage collection policy. See §3.1 for the full deletion lifecycle.

### Implicit Directories

Directories do not have their own entries in the path table. A directory "exists" if any file path starts with it as a prefix. `readdir()` uses prefix queries to find immediate children. This eliminates bookkeeping for directory create/delete/move.

### Branching

```
lakectl branch create ~/projects/myapp --name experiment
# Work freely, all changes tracked
lakectl merge experiment → main
```

The filesystem understands branches. Developers branch entire working environments. Writers branch novel drafts. Artists branch projects.

### Native Diffing

Content-addressed chunks make computing diffs between any two history points efficient. "What changed between Tuesday and today?" is a filesystem operation. The diff engine performs structural comparison at the chunk level with byte offset precision.

### Merge Conflict Resolution

Two apps or users modify the same file — the filesystem detects the conflict and presents merge tooling at the OS level.

### Multi-File Atomic Commits

"Renamed three files, edited one, deleted one — one atomic commit." The metadata layer records intent, not just individual changes.

### Distributed Collaboration

Pushing and pulling between machines is a filesystem operation. Add a remote, sync at the lake level. Dedup means only novel content transfers. Multi-user collaboration with full branching and merging is built into the OS.

---

## 3.1 Deletion Architecture — Detritus & Diagenesis

Deletion in DeepLake is a two-phase process, not an erasure event.

### Phase 1 — Detritus (Soft Delete / Dereference)

When a user deletes a file, no blob is touched. The metadata reference is dropped and the object enters detritus state — unattached material present in the lake but no longer part of the active structure. Still present, still recoverable, simply dereferenced.

### Phase 2 — Diagenesis (Garbage Collection / Compaction)

The background garbage collection daemon (`diagenesisd`) runs the compaction process. It confirms reference counts are truly zero across all branches and views before reclaiming blob storage.

### Metadata Persistence After Reclamation

Even after full blob reclamation, the metadata record persists. The knowledge graph never develops holes. A fully reclaimed object leaves a tombstone node with:

- Its BLAKE3 digest (identity)
- All relationship edges to other objects
- Timestamps of active and detritus state
- Type and semantic metadata

Deletion is informative rather than destructive. Queries like "what documents referencing Acme Corp did we delete last quarter" remain answerable after blob reclamation. This has significant implications for the enterprise compliance use case.

### Diagenesis Depth Policy

Retention policy is expressed as stratigraphic depth:

- **Recent layers** — full blob retention
- **Middle history** — blobs retained for referenced objects only
- **Deep history** — metadata skeleton only, blobs reclaimed

Diagenesis at certain depth thresholds requires explicit user intent rather than running automatically. The lake should be reluctant to forget.

---

## 4. Boot Chain

### The Bootstrap Problem

The lake filesystem requires its kernel module to be readable, but loading the kernel requires reading something. GRUB cannot read the custom format.

### Boot Sequence

```
Stage 0: UEFI System Partition (ESP)
  Standard FAT32, holds UEFI bootloader stub. Non-negotiable.

Stage 1: /boot (ext4) or kernel on ESP directly
  Kernel image, initramfs containing varvefs.ko.
  GRUB or systemd-boot loads the kernel.

Stage 2: initramfs (where the magic happens)
  1. Kernel loads, unpacks initramfs
  2. initramfs contains: varvefs.ko, busybox, lakectl, laked (metadata daemon)
  3. insmod varvefs.ko
  4. Start laked against the lake partition
  5. laked reconstructs the POSIX root view
  6. mount -t varvefs /dev/sda3 /sysroot
  7. pivot_root into /sysroot
  8. Hand off to systemd/init
```

### Cold Start Performance

- **Hot cache serialization.** On clean shutdown, serialize the POSIX view tree into a compact binary snapshot. On boot, load directly instead of reconstructing from the relational layer.
- **Lazy view resolution.** Mount with only top-level paths resolved (`/usr`, `/etc`, `/var`, `/home`). Deeper paths resolve on first access.
- **Boot-critical object pinning.** Mark critical objects and view paths. Store them in a contiguous fast-access disk region with view mappings cached in the superblock area. Kernel module resolves these without the full metadata daemon.

### Crash Recovery

- WAL recovers both object store and metadata atomically
- `lakectl repair` can rebuild metadata from the object store as a last resort
- Objects are self-verifying via BLAKE3
- Worst case: all files intact, semantic metadata lost, flat listing of hashes
- Optional: minimal POSIX path hint stored inside each object's header as a recovery breadcrumb

---

## 5. Continuous Micro-Hashing (No Work Ever Lost)

### The Mutable Staging Area

When a file is opened for writing, it enters a staging state in a mutable scratchpad alongside the lake:

```
Lake (immutable):      b3:7f2a91c3e4...  ← last committed version
Staging (mutable):     staging:a8f3c2...  ← active working copy
```

The kernel module intercepts `open()` with write flags, copies/COWs into staging. The app sees a normal mutable file.

### Micro-Hash Timeline

The kernel tracks dirty pages and periodically snapshots current state without waiting for the app to save:

```
t+0s:    Staging copy created
t+5s:    Dirty pages → micro-hash → b3:aa11...
t+12s:   Changes → micro-hash → b3:bb22...
t+30s:   Changes → micro-hash → b3:cc33...
t+45s:   fsync() → micro-hash → b3:dd44...
t+120s:  App save → full hash → b3:ee55... → promoted to lake
close(): Final hash → committed to lake
```

Each micro-hash stores only chunked diffs against the last, via content-defined chunking. Space-efficient.

### BLAKE3 Incremental Hashing

BLAKE3 is a tree hash — incrementally updatable as chunks change. A 4KB change in a 500MB file rehashes a tiny fraction. FastCDC still decides where chunk boundaries are during ingestion/change detection.

### Tiered Frequency

- Small text/code/documents: micro-hash every few seconds
- Large binaries: micro-hash on fsync or every 30-60 seconds
- Huge files: only on explicit save, or chunk-level tracking with longer intervals

Policy stored per-type in the metadata layer. The AI enrichment layer can learn patterns and adjust.

### Crash Recovery With Micro-Hashes

```
1. Lake intact (immutable, BLAKE3 verified)
2. Check staging journal
3. Found uncommitted staging for report.docx
4. Last micro-hash: b3:cc33... from 12 seconds before crash
5. Options: auto-recover to last micro-hash, or present timeline to user
```

### Application Integration (Optional)

```c
lakefs_commit(fd, "User pressed Ctrl+S");      // meaningful save point
lakefs_commit(fd, "Exported final render");     // semantic milestone
lakefs_history(fd, &versions, &count);          // query history
```

No app needs to know about any of this. Micro-hashing works at the VFS layer. Legacy apps get continuous versioning for free.

---

## 6. Zone Architecture

### The Two Worlds

System/app files need POSIX compliance and speed with zero metadata overhead. User/knowledge files get full lake treatment. The filesystem partitions into zones:

```
Zone: POSIX (near-zero overhead)
├── /usr, /bin, /sbin    ← package manager owned
├── /etc                 ← system config
├── /var                 ← runtime state, logs, caches
├── /tmp                 ← ephemeral, never hashed
├── /opt                 ← third-party installs
└── /run                 ← tmpfs

Zone: LAKE (full semantics)
├── /home                ← user files
├── /projects            ← or whatever views users create
└── /shared              ← multi-user collaborative

Zone: HYBRID (lake storage, minimal metadata)
├── /var/mail            ← worth indexing
├── /srv                 ← depends on use case
└── /opt/data            ← app data the user cares about
```

### Kernel-Level Zone Dispatch

```c
switch (li->zone) {
case ZONE_POSIX:
    // Direct write, no hashing, no staging, no metadata
    // Behaves exactly like ext4
    return lakefs_posix_write(f, buf, count, pos);
case ZONE_LAKE:
    // Full lake semantics: staging, dirty page tracking, micro-hash
    return lakefs_lake_write(f, buf, count, pos);
case ZONE_HYBRID:
    // Content-addressed (dedup + integrity) but minimal metadata
    return lakefs_hybrid_write(f, buf, count, pos);
}
```

### POSIX Zone Implementation

Files still go into the content-addressed lake (getting dedup and BLAKE3 integrity for free) but the metadata layer stores only bare minimum — POSIX path mapping and standard stat fields. No enrichment, no micro-hashing, no version history. Hash cost paid once at install by the package manager.

Benefits: instant system integrity verification, package updates as lake operations, trivial rollbacks.

### Home Directory Sub-Zones

```
/home/user/
├── .cache/          → POSIX (ephemeral, never index)
├── .config/         → HYBRID (track changes, minimal metadata)
├── .local/share/    → HYBRID (some app data worth indexing)
├── .local/bin/      → POSIX
├── Documents/       → LAKE
├── Projects/        → LAKE
└── Downloads/       → LAKE (with AI auto-triage)
```

Dotfile convention: paths starting with `.` default to HYBRID unless explicitly promoted.

### Zone Promotion and Demotion

```
lakectl zone set /etc/nginx LAKE       # version history on nginx configs
lakectl zone set ~/Projects/myapp/node_modules POSIX  # stop indexing junk
```

The AI can auto-suggest promotions: "tax_return_2025.pdf looks important. Promote to LAKE?"

### Performance Guarantee

Anything in the POSIX zone must be indistinguishable in performance from ext4.

---

## 7. Per-Process POSIX Virtualization

Since the POSIX hierarchy is a computed view, different processes can see different views. The filesystem is a POSIX virtualization layer.

### App Sandbox Transparency

```
What Firefox sees:                    What the Lake stores:
~/.var/app/.../bookmarks.json    →    b3:ff82a1...
                                       metadata: {
                                         type: "browser_data",
                                         app: "firefox",
                                         posix_views: [
                                           "~/.var/app/.../bookmarks.json",
                                           "/views/browser/bookmarks/firefox.json"
                                         ]
                                       }
```

The app sees its expected sandbox. The user sees their unified view. The lake sees content. Three projections of the same reality.

### Process View Resolution

```c
static struct dentry *lakefs_lookup(struct inode *dir,
                                     struct dentry *dentry,
                                     unsigned int flags)
{
    struct lakefs_view *view;
    view = lakefs_resolve_process_view(current->pid);
    return lakefs_view_lookup(view, dir, dentry);
}
```

The kernel module identifies the calling process (PID → exe → sandbox profile) and resolves paths within the appropriate view.

### Process View Table

```
┌─────────────┬──────────────────────────────────┬─────────────┐
│ Process/App  │ POSIX Path                       │ Lake Target  │
├─────────────┼──────────────────────────────────┼─────────────┤
│ firefox     │ ~/.var/app/.../bookmarks.json    │ b3:ff82a1...│
│ user:shell  │ ~/Documents/bookmarks.json       │ b3:ff82a1...│
│ thunderbird │ ~/.var/app/.../address_book.db   │ b3:92cc10...│
│ user:shell  │ ~/Contacts/thunderbird.db        │ b3:92cc10...│
│ flatpak:*   │ ~/.var/app/$APPID/*              │ auto-mapped │
│ snap:*      │ ~/snap/$APPNAME/*                │ auto-mapped │
└─────────────┴──────────────────────────────────┴─────────────┘
```

### What This Unlocks

- **App data liberation.** Firefox bookmarks, Chrome bookmarks — all in the lake as `type:bookmark`. Unified user view. Each browser sees only its own data.
- **No more "where did the app put my file?"** The lake catches everything; the user finds it through semantic search.
- **Invisible sandboxing.** Flatpak/Snap scatter data across ugly paths. Users never see it. Their view shows content organized by meaning.
- **Cross-app relationships.** The AI notices a PDF downloaded in Firefox is the same document being edited in LibreOffice and referenced in Thunderbird email. Three sandbox paths, one lake object, relationship edges to all three.

### App Database Extraction

Apps using SQLite or other databases get format-aware enrichment:

```
enrichd plugin: sqlite_extractor
  ├── Detects SQLite in app sandbox zones
  ├── Knows common schemas (Firefox places.sqlite, etc.)
  ├── Extracts semantic content into metadata layer
  └── Re-extracts on change (debounced)
```

The lake object is the opaque database file. The metadata layer has queryable extracted content.

### Lake Manifest (Optional App Integration)

```yaml
# org.mozilla.firefox.lake-manifest
app_id: org.mozilla.firefox
data_paths:
  - path: data/places.sqlite
    type: browser_history_bookmarks
    extractor: sqlite_firefox_places
    zone: hybrid
  - path: cache/
    zone: posix
    ephemeral: true
```

Apps without a manifest get sensible defaults. Apps with a manifest get perfect integration. Incentivizes adoption.

---

## 8. App & System Rollback

Every write is content-addressed. Every state change is in the commit DAG. The POSIX view is computed. Rolling back means pointing the view at older hashes.

### App-Scoped Commits

Every app launch/close auto-commits a snapshot of that app's entire state:

```
commit: a8f3c2 "org.mozilla.firefox state"
  timestamp: 2026-02-05T14:30:00
  trigger: app_close
  objects:
    ├── b3:aa11... (places.sqlite)
    ├── b3:bb22... (prefs.js)
    ├── b3:cc33... (extensions.json)
    └── b3:dd44... (logins.json)
```

### Rollback

```
lakectl rollback --app org.mozilla.firefox --to tuesday
```

Old objects still in the lake. Nothing deleted. View rewound. Roll forward again if the rollback was wrong.

### Selective Rollback

```bash
# Roll back only extensions, keep bookmarks current
lakectl rollback --app org.mozilla.firefox \
  --filter "extensions.*" --to tuesday

# Roll back one file
lakectl rollback --app org.mozilla.firefox \
  --file prefs.js --to "before:2026-02-05T10:00"
```

### System-Wide Rollback

```bash
# Bad system update
lakectl rollback --zone posix --to "before:last-apt-upgrade"
# Rolls back /usr, /etc — leaves /home untouched (different zone)
```

### What This Replaces

- **Timeshift/Snapper** (Btrfs snapshots): whole-volume, rolls back everything
- **Flatpak/Snap:** no rollback of app data, only binaries
- **Manual backups:** `cp -r ~/.mozilla ~/.mozilla.bak`
- **Time Machine:** separate system, hourly granularity, clunky restore UI

DeepLake does it with one command, any granularity, scoped precisely, instantly, no prior setup.

### Desktop UX: Timeline

```
┌─────────────────────────────────────────────┐
│  Timeline                                    │
├─────────────────────────────────────────────┤
│  Firefox          ←────────●────────→        │
│  Feb 3    Feb 4    [Feb 5]   Feb 6           │
│  12 states captured today                    │
│  Last: 2:30 PM (app closed)                  │
│  [Preview State]  [Rollback]  [Compare]      │
├─────────────────────────────────────────────┤
│  LibreOffice      ←────────●────────→        │
│  System           ←────────●────────→        │
│  VS Code          ←────────●────────→        │
└─────────────────────────────────────────────┘
```

AI enrichment annotates the timeline: "Extension updated", "15 new bookmarks", "preferences reset detected."

---

## 9. AI Enrichment Layer

### Pipeline

Every file landing in the lake optionally passes through an AI pipeline that reads, understands, and enriches the metadata layer automatically.

A user drops a PDF. Without any action:

1. Hashed and stored
2. AI identifies it as a contract with Acme Corp
3. Tagged: `type:legal`, `entity:AcmeCorp`, `date:2026-03-15`
4. Related to existing emails mentioning Acme Corp via relationship edges
5. Linked to another contract with similar clauses
6. Plain-English summary generated as a metadata property
7. Deadline surfaced to calendar integration

### Architecture

```
          ┌───────────────────────────────────────┐
          │         Metadata Layer                 │
          └───┬───────────────────────────────┬───┘
              │                               │
     ┌────────▼────────┐           ┌──────────▼──────────┐
     │   Human Input    │           │   AI Enrichment     │
     │                  │           │   Daemon (enrichd)  │
     └─────────────────┘           │                     │
                                   │  Local: llama.cpp   │
                                   │  Cloud: Claude API  │
                                   │  Hybrid: both       │
                                   └──────────┬──────────┘
                                              │
                                   ┌──────────▼──────────┐
                                   │   BLAKE3 Hash Lake   │
                                   └─────────────────────┘
```

### Enrichment Stages

1. **Identification:** Beyond MIME — "this is an invoice" not just "this is a PDF"
2. **Extraction:** Names, dates, amounts, locations, topics, sentiment, language
3. **Relationship Discovery:** Scans the entire lake for connections. "This spreadsheet references the same project as these emails and this slide deck." Builds a knowledge graph automatically.
4. **Summary & Embeddings:** Natural-language summary per object. Vector embeddings for true semantic search: "find that supply chain optimization thing" works even if no file contains that phrase.

### CLI

```bash
lakectl ai set-provider local --model llama3.2
lakectl ai set-provider cloud --endpoint https://api.anthropic.com --model claude-sonnet
lakectl ai set-provider hybrid --local-first --cloud-fallback

lakectl ai enrich b3:7f2a91c3e4...
lakectl ai enrich --view ~/Documents/Projects/
lakectl ai ask "what contracts are expiring this quarter?"
lakectl ai watch --on --filter "type:document OR type:image"
```

### Privacy Architecture

- **Local mode (default):** On-device via quantized model. Data never leaves the machine.
- **Cloud mode:** Sent to Claude/GPT/etc. Explicit opt-in per view or type.
- **Hybrid mode:** Local first-pass. Uncertain items queued for cloud. User approves queue.
- **Zero-knowledge cloud (ambitious):** Only embeddings and extracted text sent, never raw files.

The user always owns the metadata. Switch providers or go fully local — all enrichment persists.

### Zone Awareness

```ini
[zones]
posix = ignore
lake = full
hybrid = basic

[lake.filters]
exclude = ["*.tmp", "*.swp", "node_modules/*", ".git/*"]
defer_large = 500MB
```

The enrichment daemon never sees POSIX zone files.

---

## 9.1 Transclusion as an Emergent Property

Content addressing creates the mechanical foundation for Ted Nelson's transclusion concept as an emergent property. Because identity is derived from content, identical content is physically impossible to duplicate in the lake. Any two documents sharing content are already pointing at the same bytes.

What remains absent is the semantic layer that distinguishes coincidental sharing from intentional sharing. This belongs in the metadata layer — the lake provides exact dedup, the metadata layer formalizes named reusable fragments, and the AI layer discovers near-duplicates via embeddings.

The layered approach to similarity:

| Layer | Question | Method | Cost |
|---|---|---|---|
| Lake | Are these bytes identical? | Binary hash comparison | Free |
| Diff engine | Which chunks changed? | Structural chunk-level comparison | Cheap |
| Metadata layer (future) | Are these objects semantically related? | Typed relationships, queryable | Moderate |
| AI enrichment (future) | Do these objects mean similar things? | Vector embeddings, similarity search | Expensive |

Each layer answers a harder question using the appropriate tool. The lake never attempts fuzzy matching. That is a higher-layer concern. Each layer creates capabilities that the layer above can choose to formalize or ignore.

---

## 10. Two-Drive Architecture

A second physical drive is recommended as a default configuration. Unlike RAID, the second drive serves multiple architectural roles simultaneously rather than providing pure redundancy.

### Drive Roles

**Drive 1 — The Lake**
- Immutable, append-only object store
- Metadata layer
- Purely settled, consolidated content

**Drive 2 — Staging & Buffer** (the delta layer)
- Mutable staging area for active writes
- Write-ahead log / journal
- Recent object buffer (delta of promoted objects pending remote confirmation)
- Absorbs write patterns unfriendly to content-addressed storage

### Failure Mode Analysis

- **Lake drive fails:** staging drive has recent history, remote has the rest
- **Staging drive fails:** lake drive fully intact and immutable, lose only uncommitted working state
- **Both fail:** remote coverage

### Installer Configuration Tiers

- **One drive** — supported, tradeoffs disclosed
- **Two drives** — recommended, benefits explained
- **Drive plus remote** — ideal, architecture explained

The second drive is not redundancy — every function it serves would need to exist somewhere in the architecture regardless. Physical separation is an elegant home for concerns that are already logically separate.

---

## 11. Remote Sync

A **remote** is any external lake outside the local machine: NAS, another DeepLake machine, cloud endpoint, or trusted peer.

Sync is not traditional backup. It is lake-to-lake delta transfer — only novel object hashes transfer, content-addressing ensures the remote verifies everything received, deduplication means no redundant transfers.

### Remote Types

- **Local NAS** — fast, user-controlled, continuous or scheduled sync
- **Another DeepLake machine** — peer collaboration that doubles as redundancy
- **Cloud endpoint** — S3 or managed DeepLake service, geographically separate
- **Trusted peer** — encrypted lake objects stored by peers without read access (ambitious)

### Composition with Two-Drive Architecture

- Staging drive protects the seconds-to-minutes window
- Remote protects against total drive failure
- Together they cover the full durability envelope with no single point of failure

---

## 12. The WinFS Lesson (Architectural Principle)

What killed WinFS was coupling. The type system was coupled to SQL Server, coupled to the Windows Shell, coupled to .NET, coupled to the application model. Changing anything meant changing everything.

DeepLake avoids this because the lake does not care about types — it stores hashes. The metadata layer cares about types. The view engine cares about queries over types. The enrichment layer cares about assigning types. These can all evolve independently. If the type system for audio is wrong, the objects are re-enriched. The lake never changes.

This decoupling is the single most important structural advantage over WinFS and must be preserved as higher layers are designed.

Content addressing means identity is derived from content, not location. The moment that choice is made, it becomes impossible for the same content to exist twice. The lake is just hashes. The metadata layer cares about types. The view engine cares about queries over types. The enrichment layer cares about assigning types. These can all evolve independently.

---

## 13. Business Model

### Open Source Core

The filesystem, kernel module, POSIX simulation, view engine, metadata layer — all open. GPLv2 for the kernel module, LGPL or MPL for userspace.

### Revenue: Paid Cloud Enrichment

Managed AI enrichment API. Better models, faster processing, subscription or per-GB pricing.

### Revenue: Enterprise Tier

Team lakes with shared semantic metadata. Organization-wide semantic search across all employees' filesystems. Compliance tagging, automatic classification, retention policies — all AI-driven.

### Revenue: Schema & Plugin Marketplace

Third-party enrichment pipelines for specific domains — legal, medical, engineering, academic.

---

## 14. Development Phases

### Phase 1 — Alluvium: FUSE Prototype (3-6 months)

- FUSE filesystem with SQLite metadata backend
- Content-addressed storage with BLAKE3 hashing
- FastCDC chunking with sub-file deduplication
- Per-file version history with sequential version numbers
- Structural diffing at the chunk level with byte offset precision
- Rewind to any previous version (non-destructive)
- Storage efficiency reporting (dedup ratio, space savings)
- Virtual history namespace (`.alluvium/`) browsable with standard Unix tools
- Basic local LLM enrichment pipeline
- CLI query tool (`lakectl`)

**Implementation: Rust.** Chosen for long-term investment — the prototype becomes the foundation, the foundation becomes the kernel module. Ecosystem alignment: BLAKE3 (native Rust), FastCDC (solid crate), fuser (best FUSE library), rusqlite (SQLite bindings), serde (serialization), clap (CLI), colored (output formatting).

**Crate Structure:**

```
alluvium/
├── Cargo.toml                  # workspace
└── crates/
    ├── lake-core/              # all core logic, testable without FUSE
    │   ├── types.rs            # B3Hash, Chunk, ChunkRef, Manifest, Version
    │   ├── store.rs            # content-addressed object store
    │   ├── ingest.rs           # FastCDC chunking pipeline
    │   ├── diff.rs             # LCS-based positional diff engine
    │   ├── path_table.rs       # SQLite path → HEAD mapping + versions
    │   └── history.rs          # version chain coordination layer
    ├── lake-fuse/              # FUSE mount + .alluvium virtual namespace
    └── lakectl/                # CLI: history, diff, rewind, stats, restore, cat
```

**On-Disk Layout (Prototype):**

```
.lake/
├── objects/                    # content-addressed store
│   ├── 7f/
│   │   └── 2a91c3e4...       # [0x01][chunk bytes] or [0x02][manifest JSON]
│   └── aa/
│       └── bb11cc33...
├── paths.db                   # SQLite: path table + version history
└── inodes.db                  # SQLite: persistent inode ↔ path mapping
```

**Chunking Parameters (Prototype Defaults):** Min 8 KB, Average 16 KB, Max 64 KB. Future: per-file or per-type chunking policies stored in the metadata layer.

**The prototype is intentionally limited to the storage layer and version history.** No metadata layer. No type system. No AI enrichment. No relationship model. This is a deliberate architectural decision. The demo tells a complete story without those layers. The metadata and AI layers are presented after the demo: "Now imagine this same system knows what's inside the files." The foundation sells itself before the higher layers are mentioned.

### Phase 2 — Varvefs: Kernel Module (6-12 months)

- Move hot path into a real kernel module
- Userspace metadata daemon
- Change notifications
- Query virtual filesystem
- Seek-based chunk resolution and LRU caching for large file performance
- Micro-hash implementation, staging area design, tiered frequency policies

### Phase 3 — Desktop Integration (6-12 months)

- Custom or forked file manager with semantic browsing
- KDE Plasma integration or custom DE
- D-Bus API for app developers
- Migration tools for existing files

### Phase 4 — DeepLake OS Distribution (ongoing)

- Base on Arch, Fedora, or Debian
- Custom installer with drive configuration tiers
- Documentation, SDK, developer outreach

---

## 15. Large File Performance

The Alluvium prototype reads entire files into memory on every `read()` call. This is the known first optimization target. The solution is seek-based chunk resolution using the manifest as a seek table:

```
Manifest for movie.mkv:
  chunk 0:   hash=aa11, size=65536   → bytes 0..65535
  chunk 1:   hash=bb22, size=65536   → bytes 65536..131071
  chunk 2:   hash=cc33, size=65536   → bytes 131072..196607
  ...
  chunk 61440: hash=ff99, size=41088 → bytes 4294926336..4294967423

read(offset=131100, size=4096)
  → falls in chunk 2 (starts at 131072)
  → fetch only chunk cc33 from store
  → offset within chunk: 131100 - 131072 = 28
  → return bytes 28..4124 from that one chunk
```

One disk read. One hash verification. 64KB into memory instead of 4GB. Return the 4KB slice the app asked for.

Sequential reads benefit from an LRU cache — as long as the read position remains within a chunk, subsequent reads are served from memory. When the position crosses a chunk boundary, the next chunk is fetched. The application sees a smooth stream of bytes with no awareness of the underlying chunked storage.

The FUSE contract (`read(ino, offset, size) → bytes`) does not change — only the internal resolution strategy. No architectural changes required. This is how IPFS, Ceph, and BitTorrent handle chunked reads.

---

## 16. The Pitch

**DeepLake OS — The first operating system that understands your files.**

Content-addressed storage. Automatic AI-powered semantic tagging. Full version history on everything. A filesystem that's also a knowledge graph. Never lose a file. Never lose a version. Never break your computer again.

Open source core. Paid AI enrichment cloud. Enterprise collaboration tier.

TAM: Every knowledge worker who's ever lost a file, forgotten a version, or wished their computer could just find things.

---

## Open Design Questions

1. **Metadata layer design** — types, properties, relationships, the knowledge graph. Where WinFS's ambition meets DeepLake's decoupled architecture.
2. **Global commit DAG** — point-in-time snapshots layered on top of per-file history. "What did my project look like last Tuesday?"
3. **Branching and merging UX** — filesystem-level branches, merge conflict model, user experience design.
4. **Per-process view virtualization** — different apps seeing different projections, sandbox transparency model.
5. **AI enrichment pipeline** — enrichd architecture, plugin model, privacy modes, zone awareness.
6. **Distributed sync protocol** — specification for lake-to-lake delta transfer.
7. **Diagenesis depth policy UX** — how stratigraphic retention is expressed to users in the Timeline interface.
8. **Naming** — metadata daemon, enrichment daemon, CLI tool, sync daemon still require names from the geological vocabulary.
9. **Installer UX** — drive configuration tier presentation and guidance.

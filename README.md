# PES-VCS: Building a Version Control System from Scratch

**Student:** SAI KISHAN A
**SRN:** PES1UG24CS608  
**Repository:** PES1UG24CS608-pes-vcs  
**Platform:** Ubuntu 22.04

---
## Phase 1 — Object Storage Foundation

### Screenshot 1A — All Phase 1 tests passing  
![1A](screenshots/1.png)

All three tests pass:
- PASS: blob storage
- PASS: deduplication
- PASS: integrity check

---

<<<<<<< HEAD
### Screenshot 1B — Sharded object directory  
![1B](screenshots/2.png)
=======
## Phase 1 — Object Storage Foundation

**Concepts:** Content-addressable storage, directory sharding, atomic writes, SHA-256 hashing for integrity.

**Files implemented:** `object.c` — functions `object_write` and `object_read`.

### What was implemented

`object_write` stores any data (blob, tree, commit) in the object store by:
1. Prepending a header in the format `"<type> <size>\0"` to the raw data
2. Computing the SHA-256 hash of the full object (header + data)
3. Skipping the write if the object already exists (deduplication)
4. Creating the shard directory `.pes/objects/XX/` (first 2 hex chars)
5. Writing atomically via a temp file + `rename()` to prevent partial writes
6. `fsync()`-ing both the file and the directory to persist the rename

`object_read` retrieves and verifies objects by:
1. Building the file path from the hash using `object_path()`
2. Reading the entire file into memory
3. Recomputing the SHA-256 and comparing to the filename — returns `-1` if corrupt
4. Parsing the header to extract the type string and data size
5. Returning a heap-allocated copy of the data portion (after the `\0`)

### Screenshot 1A — All Phase 1 tests passing
Screenshot 1A — All Phase 1 tests passing  
![1A](screenshots/1.png)

![Phase 1 test_objects output](1.png)

All three tests pass:
- **PASS: blob storage** — write and read back a blob correctly
- **PASS: deduplication** — same content produces same hash, stored only once
- **PASS: integrity check** — corrupt objects are detected and rejected

### Screenshot 1B — Sharded object directory structure

![find .pes/objects -type f](2.png)

Objects are sharded under `.pes/objects/XX/` where `XX` is the first two hex characters of the SHA-256 hash. This prevents any single directory from growing too large.
>>>>>>> e769ec7 (fix: rename screenshots folder)

---

## Phase 2 — Tree Objects

### Screenshot 2A — All Phase 2 tests passing  
![2A](screenshots/3.png)

---

### Screenshot 2B — Raw tree object  
![2B](screenshots/4.png)

---

## Phase 3 — Index (Staging Area)

### Screenshot 3A — pes init → add → status  
![3A](screenshots/5.png)

---

### Screenshot 3B — index file  
![3B](screenshots/6.png)

---

## Phase 4 — Commits and History

### Screenshot 4A — commit log  
![4A](screenshots/7.png)

---

### Screenshot 4B — object store  
![4B](screenshots/8.png)

---

### Screenshot 4C — HEAD and refs  
![4C](screenshots/9.png)

---

## Final — Integration Test

![Final](screenshots/10.png)

---


## Phase 5 — Branching and Checkout (Analysis)

### Q5.1 — How would you implement `pes checkout <branch>`?

A branch in PES-VCS is just a file at `.pes/refs/heads/<branch>` containing a commit hash. Implementing `pes checkout <branch>` requires changes at three levels:

**Files that must change in `.pes/`:**
- `.pes/HEAD` — must be updated to `ref: refs/heads/<branch>` so future commits advance the new branch. If checking out a new branch, the ref file is created pointing to the current HEAD commit.

**What must happen to the working directory:**
1. Read the target branch's commit hash from `.pes/refs/heads/<branch>`
2. Read the commit object to get the root tree hash
3. Walk the target tree recursively and compare every entry to the current HEAD's tree
4. For files that exist in the target tree but differ from the current tree: overwrite the working file with the blob content from the object store
5. For files in the current tree that don't exist in the target tree: delete them from the working directory
6. Update the index to reflect the new tree's contents exactly

**What makes this operation complex:**
- **Dirty working directory detection** — if the user has modified a file that differs between branches, checkout must refuse or stash the changes to avoid data loss
- **Three-way comparison** — the checkout logic must compare HEAD tree, target tree, AND the working directory simultaneously
- **Atomic failure recovery** — if checkout is interrupted partway through (power failure, signal), the working directory can be left in a mixed state. Git handles this with a `MERGE_HEAD` mechanism; PES-VCS would need something similar
- **Subdirectory management** — directories that exist in one branch but not the other must be created or removed, including empty parent directories
- **Permissions** — executable bit (`100755` vs `100644`) must be applied via `chmod`

---

### Q5.2 — Detecting dirty working directory conflicts using only the index and object store

The index stores three pieces of metadata per file: the blob hash at the time of staging, the mtime, and the size. This is enough to detect conflicts without re-hashing every file:

**Step 1 — Fast dirty check using metadata:**  
For each file tracked in the current index, call `lstat()`. If `st_mtime` or `st_size` differs from what is recorded in the index entry, the file has likely been modified since the last `add`. This is the same fast-diff approach used in `index_status`.

**Step 2 — Confirm with hash if metadata changed:**  
If metadata differs, read the file and compute its SHA-256 blob hash. Compare to the blob hash stored in the index. If they differ, the file is genuinely dirty.

**Step 3 — Cross-branch conflict check:**  
For each dirty file identified above, look up the same path in the target branch's tree (by reading the target commit → target tree → walking tree entries). If the blob hash in the target tree differs from the blob hash in the current HEAD's tree for that path, then the file differs between branches and the user has local modifications — this is a conflict. Checkout must refuse.

If the target branch has the same blob for that file as the current branch, the local modification is safe to carry over and checkout can proceed.

This approach never requires checking out or materializing target-branch files just to detect conflicts — it only reads tree objects from the object store and compares hashes.

---

### Q5.3 — Detached HEAD and recovery

**What "detached HEAD" means:**  
Normally `.pes/HEAD` contains `ref: refs/heads/main` (an indirect reference). In detached HEAD state, `.pes/HEAD` contains a raw commit hash directly, e.g. `a1b2c3d4...`. This happens when you checkout a specific commit hash rather than a branch name.

**What happens if you commit in detached HEAD state:**  
New commits are created normally — `commit_create` calls `head_read` (which reads the raw hash from HEAD), sets it as the parent, writes the new commit, and calls `head_update` which writes the new commit hash back to HEAD directly. The commits exist in the object store and HEAD advances, but no branch ref is updated. The commit chain is only reachable via HEAD itself.

**The danger:**  
If you then run `pes checkout main` (or any branch), HEAD is updated to point to `refs/heads/main`, and the commits you made in detached HEAD state become unreachable — no branch points to them. They will eventually be deleted by garbage collection.

**How to recover:**  
Before switching away, note the commit hash shown in HEAD (e.g., `cat .pes/HEAD`). Then create a new branch pointing to it:
```bash
# While still in detached HEAD state:
cat .pes/HEAD   # note the hash, e.g. deadbeef...
echo "deadbeef..." > .pes/refs/heads/my-recovery-branch
```
Now `my-recovery-branch` points to the detached work and it won't be lost. In a real Git implementation, the `ORIG_HEAD` file and reflog exist precisely to help recover from this situation.

---

## Phase 6 — Garbage Collection (Analysis)

### Q6.1 — Algorithm to find and delete unreachable objects

**The algorithm (mark-and-sweep):**

**Mark phase — find all reachable objects:**
1. Start with all branch refs: read every file under `.pes/refs/heads/` to get a set of commit hashes
2. Use a hash set (e.g., a hash table keyed by the 32-byte `ObjectID`) to track visited hashes
3. For each starting commit hash, do a BFS/DFS walk:
   - Read the commit object → mark the commit hash as reachable
   - Add the commit's `tree` hash to the work queue
   - If the commit has a `parent`, add the parent hash to the work queue
4. For each tree hash in the work queue:
   - Read the tree object → mark the tree hash as reachable
   - For each entry: if it's a blob (`100644`/`100755`), mark the blob hash as reachable; if it's a subtree (`040000`), add its hash to the work queue

**Sweep phase — delete unreachable objects:**
1. Walk every file under `.pes/objects/XX/YYY...`
2. Reconstruct the `ObjectID` from the path (XX + YYY = full hex hash, convert to bytes)
3. If the hash is NOT in the reachable set, delete the file with `unlink()`
4. After deleting files, try `rmdir()` on each shard directory to clean up empty `XX/` dirs

**Data structure:** A hash set of `ObjectID` values (32 bytes each). A simple open-addressing hash table or a sorted array with binary search works well. For 100,000 commits and 50 branches, the set needs to hold perhaps 500,000–2,000,000 entries (commits + trees + blobs), requiring roughly 64–128 MB of memory at 32–64 bytes per entry.

**Estimate of objects to visit for 100,000 commits and 50 branches:**  
- 100,000 commit objects (one per commit)
- ~100,000 root tree objects (one per commit)
- Each commit touches some subtrees and blobs; assuming average 10 unique objects per commit delta: ~1,000,000 total objects to mark
- Total traversal: on the order of 1–2 million object reads

---

### Q6.2 — Race condition between GC and concurrent commit

**The race condition:**

Consider this interleaving between a `commit` process and a `gc` process running concurrently:

| Time | Commit process | GC process |
|------|---------------|------------|
| T1 | Calls `object_write(OBJ_BLOB, ...)` — blob `a1b2...` written to object store | |
| T2 | | Starts mark phase — reads all branch refs |
| T3 | | Traverses all reachable objects from current commits — does NOT find `a1b2...` (no commit references it yet) |
| T4 | | Sweep phase: deletes `a1b2...` as unreachable |
| T5 | Calls `tree_from_index` and `commit_create` — builds a commit that references blob `a1b2...` | |
| T6 | `object_write(OBJ_COMMIT, ...)` — commit written, HEAD updated to point to new commit | |
| T7 | | GC finishes |

**Result:** The new commit references blob `a1b2...`, but GC deleted it between T3 and T4. The repository is now corrupt — reading the new commit will find the blob missing.

**How Git's real GC avoids this:**

1. **Grace period (`--prune=2.weeks.ago`):** Git's `git gc` never deletes objects newer than 2 weeks old by default, regardless of reachability. A just-written blob will always be newer than this threshold.

2. **`gc.pid` lock file:** Git creates a lock file before starting GC. If a concurrent operation is running, it either waits or GC aborts.

3. **Loose object age check:** Git checks the file's `mtime` before deleting — if the object was written recently (within the grace period), it is preserved even if not yet referenced by any commit. This directly prevents the race above: the blob at T1 has a fresh mtime and is skipped by the sweep at T4.

4. **Pack files:** Git's pack format batches many objects into a single file with an index. GC rewrites packs atomically; partial packs are never visible to readers until fully written.

---

## Submission Checklist

| Phase | ID | Screenshot | Status |
|-------|----|-----------|--------|
| 1 | 1A | `./test_objects` — all tests passing | ✅ |
| 1 | 1B | `find .pes/objects -type f` — sharded structure | ✅ |
| 2 | 2A | `./test_tree` — all tests passing | ✅ |
| 2 | 2B | `xxd` of raw tree object | ✅ |
| 3 | 3A | `pes init → pes add → pes status` sequence | ✅ |
| 3 | 3B | `cat .pes/index` — text format index | ✅ |
| 4 | 4A | `pes log` with three commits | ✅ |
| 4 | 4B | `find .pes -type f | sort` — object growth | ✅ |
| 4 | 4C | `cat .pes/refs/heads/main` and `cat .pes/HEAD` | ✅ |
| Final | — | `make test-integration` — all tests completed | ✅ |

## Code Files Implemented

| File | Functions Implemented |
|------|-----------------------|
| `object.c` | `object_write`, `object_read` |
| `tree.c` | `tree_from_index` |
| `index.c` | `index_load`, `index_save`, `index_add` |
| `commit.c` | `commit_create` |

# Lab: NFSv4 ACLs — `nfs4_getfacl` and `nfs4_setfacl` on an NFSv4 mount

**Series:** linux-ops-mastery — RHCSA Permissions, Special Bits & ACLs
**Subjects covered:** Why NFSv4 ACL text is **not** POSIX `getfacl` format, installing `nfs4-acl-tools`, exporting a small directory tree from localhost, mounting with `vers=4`, reading ACEs with `nfs4_getfacl`, adding and removing ACEs with `nfs4_setfacl`, SELinux and `nfs_export_all_rw` awareness at high level
**Career arcs covered:** RHCSA (enterprise NAS integrations), RHCE (automation around NFS exports), SRE (multi-OS client permission mismatches), DevOps (Kubernetes ReadWriteMany volumes backed by NFS), AI/MLOps (shared read-mostly training corpora on NetApp-style exports)
**Prerequisite:** Labs 47–48 (general ACL literacy) plus comfort restarting services
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 packages · 2–3 export + mount · 4–5 nfs4_get/set · 6 teardown

---

## Objective

When the backing filesystem is a **Linux NFSv4 export**, the wire and on-disk ACL story follows **NFSv4 named attributes** — tools are `nfs4_getfacl` and `nfs4_setfacl`, not POSIX `getfacl`. Mixing the two toolchains on the same pathname is a common junior mistake: local xfs uses one grammar; NFSv4 mounted path uses another. By the end of this lab you can stand up a **localhost NFSv4** practice export on RHEL 9, mount it, display ACE strings, add a permissive ACE, remove it, and unmount.

The capstone is a text file on the NFS mount whose NFSv4 ACL you modify twice with evidence in `/tmp/nfs4-acl.log`.

> **Lab safety note:** This lab edits `/etc/exports` and starts `nfs-server`. Use a disposable VM. `no_root_squash` is **lab-only** — never copy blindly to production.

---

## Concept: NFSv4 ACL Strings Are a Different Language Than POSIX ACL Rows

```
   Local xfs/ext4 file:
      getfacl /path      ← POSIX row grammar

   NFSv4-mounted file:
      nfs4_getfacl /path ← NFSv4 ACE grammar (A::OWNER@:..., etc.)

   Same kernel VFS — different xattr namespace + tool expectations
```

> **Why this matters:** Ticket says "fix ACL on the share" — wrong tool wastes hours.

---

## 📜 Why NFSv4 ACL Tools Exist — The Story

NFSv4 standardized richer file attributes on the wire than older NFS versions exposed. Clients needed a way to **display and edit** server-side ACLs without pretending they were classic POSIX text rows. The `nfs4-acl-tools` package (names vary slightly by distribution family) supplies the user-space bridge: it speaks the NFSv4 ACE wire format administrators recognize from heterogenous environments (Linux servers, commercial NAS heads, mixed clients).

> **The point of the story:** NFSv4 ACL literacy is **integration literacy** — the skill you use when `/data` is not a local LV.

---

## 👪 The NFSv4 ACL Tool Family — Who Lives There

| Tool | Role |
|---|---|
| `nfs4_getfacl` | Dump ACE list |
| `nfs4_setfacl` | Add/remove/replace ACEs from stdin or argv |
| `nfsiostat` / `nfsstat` | (Sibling) performance — not ACL |

### Server pieces (this lab)

| Daemon | Package |
|---|---|
| `nfs-server` | `nfs-utils` |

> **The point of the family tree:** Client tools + server export + mount option `vers=4` complete the triangle.

---

## 🔬 The Anatomy of an NFSv4 ACE String — In One Diagram

```
A::OWNER@:rwatTnNcCoy
│ │   │    └─ permission letters (see nfs4_acl(5))
│ │   └─ who: OWNER@, GROUP@, EVERYONE@, or USER:name
│ └─ ACE type: A = Allow, D = Deny (when supported/configured)
└─ NFSv4 ACE entry

Typical first line from nfs4_getfacl:
# file: /mnt/nfs4demo/hello.txt
```

> **Reading rule:** Start at `man nfs4_acl` on your RHEL box — letter meanings evolve slightly; trust local man pages over blog charts.

---

## 📚 NFSv4 ACL Reference Table

| Task | Command |
|---|---|
| Show ACL | `nfs4_getfacl FILE` |
| Edit interactively | `nfs4_editfacl FILE` (if installed) |
| Set from file | `nfs4_setfacl -S acl.txt FILE` |
| Add ACE | `nfs4_setfacl -a 'ACE' FILE` |
| Remove ACE by index | `nfs4_setfacl -x INDEX FILE` |

> **Rule one of NFSv4 ACLs:** Confirm `findmnt -n -o FSTYPE /mnt/point` prints `nfs4` (or `nfs*` per util-linux version).

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Enterprise tracks expect you to recognize NFSv4 tool names. |
| **RHCE candidate** | Playbooks may shell out to `nfs4_setfacl` when module gaps exist. |
| **SRE / Platform** | Vendor NAS tickets use NFSv4 ACE vocabulary. |
| **DevOps** | RWX volumes — know the client-side ACL story. |
| **AI / MLOps** | Shared corpora mounts — NFSv4 is still default in many shops. |

---

## 🔧 The 6 Tasks

---

### Task 1 — Install client/server utilities

**Purpose:** Ensure binaries exist.

```bash
sudo -i
dnf install -y nfs-utils nfs4-acl-tools
rpm -q nfs-utils nfs4-acl-tools
```

**Human-Readable Breakdown:** `nfs-utils` supplies server components and client mount helpers; `nfs4-acl-tools` supplies `nfs4_getfacl` / `nfs4_setfacl`.

**Expected output:**

```text
nfs-utils-...
nfs4-acl-tools-...
```

**Switches**

| Token | Meaning |
|---|---|
| `dnf install -y` | non-interactive |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| no dnf | This lab targets RHEL 9 family |

---

### Task 2 — Create export directory and dummy file

**Purpose:** Data to share.

```bash
mkdir -p /srv/nfs4lab
echo 'nfs4 acls' > /srv/nfs4lab/hello.txt
chmod 644 /srv/nfs4lab/hello.txt
```

**Human-Readable Breakdown:** Keep export shallow for debugging.

**Expected output:**

```text
(silent success)
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | parents |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| permission denied | root shell |

---

### Task 3 — Configure `/etc/exports` and start `nfs-server`

**Purpose:** Export to loopback for a client mount.

```bash
cp /etc/exports /etc/exports.bak.$(date +%F-%H%M%S)
grep -q '/srv/nfs4lab' /etc/exports || echo '/srv/nfs4lab 127.0.0.1(rw,sync,no_root_squash,crossmnt)' >> /etc/exports
exportfs -rav
systemctl enable --now nfs-server
exportfs -v
```

**Human-Readable Breakdown:** **LAB ONLY** line uses `no_root_squash` so root-owned files remain root-owned over the wire for teaching. Production uses root_squash + `all_squash` patterns with anonuid mapping.

**Reading it left to right:** `exportfs -rav` re-reads `/etc/exports` and applies. `systemctl` starts kernel server threads.

**The story:** If `exportfs` errors, nothing in later tasks works — read stderr immediately.

**Expected output:**

```text
exporting 127.0.0.1:/srv/nfs4lab
```

**Switches**

| Token | Meaning |
|---|---|
| `exportfs -rav` | reexport all, verbose |
| `crossmnt` | allow NFSv4 pseudo-tree crossing (harmless on single export lab) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `exportfs: Failed to ...` | Check path exists, `/etc/exports` syntax |
| RPC errors | `systemctl status nfs-server` |

---

### Task 4 — Mount NFSv4 locally and confirm type

**Purpose:** Get an NFSv4-mounted namespace for tools.

```bash
mkdir -p /mnt/nfs4lab
mount -t nfs -o vers=4 127.0.0.1:/srv/nfs4lab /mnt/nfs4lab
findmnt -n -o TARGET,SOURCE,FSTYPE /mnt/nfs4lab
cat /mnt/nfs4lab/hello.txt
```

**Human-Readable Breakdown:** `vers=4` selects NFSv4. `findmnt` FSTYPE may show `nfs4`.

**Expected output:**

```text
/mnt/nfs4lab 127.0.0.1:/srv/nfs4lab nfs4
nfs4 acls
```

**Switches**

| Token | Meaning |
|---|---|
| `-o vers=4` | NFS version 4 |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Connection refused` | nfs-server not running |
| SELinux blocks | `setsebool -P nfs_export_all_rw 1` on enforcing hosts (lab) |

---

### Task 5 — `nfs4_getfacl` baseline, optional `nfs4_setfacl -a`, remove by index

**Purpose:** Capture NFSv4 ACE text, optionally append one Allow ACE, then remove that ACE by **numeric index** (the portable edit pattern on RHEL).

```bash
nfs4_getfacl /mnt/nfs4lab/hello.txt | tee /tmp/nfs4-acl.before

# Optional add: who-string must match your idmap + passwd view of the user.
# Try the simple local form first; if this fails, read stderr and consult `man nfs4_setfacl`.
nfs4_setfacl -a 'A::alice@localdomain:RWX' /mnt/nfs4lab/hello.txt 2>/dev/null || \
nfs4_setfacl -a 'A:alice:RWX' /mnt/nfs4lab/hello.txt

nfs4_getfacl /mnt/nfs4lab/hello.txt | tee /tmp/nfs4-acl.after

# Remove the ACE you just added: indices are 0-based in nfs4_setfacl -x.
# Replace N with the index shown in `nfs4_getfacl` for your new line.
grep -n '' /tmp/nfs4-acl.after | head
# Example only — uncomment and fix N after you inspect output:
# nfs4_setfacl -x N /mnt/nfs4lab/hello.txt
```

**Human-Readable Breakdown:** Baseline dump to `/tmp/nfs4-acl.before`. Optional `-a` appends an Allow ACE when the who-string matches what `rpc.idmapd` and NSS expect on your VM. Re-dump to `after`. Removal is **`nfs4_setfacl -x INDEX`** — students pick `INDEX` from the line order `nfs4_getfacl` prints (see `man nfs4_setfacl` on your machine for index rules).

**Reading it left to right:** `nfs4_getfacl` always works on NFSv4 mounts. `-a` is best-effort in class environments without full idmapd domain setup — a failure still teaches you to read the man page **Examples** section.

**The story:** Production sites run **sssd** + **idmapd** with predictable `user@domain` strings. A localhost lab may only accept `alice` without a domain suffix — your stderr is the lesson.

**Expected output:**

```text
# file: /mnt/nfs4lab/hello.txt
A::OWNER@:rwatTnNcCoy
A::GROUP@:rxtncy
A::EVERYONE@:rxtncy
```

**Switches**

| Token | Meaning |
|---|---|
| `-a 'ACE'` | append one Allow/Deny ACE string |
| `-x N` | remove ACE at index **N** (verify with man page + local test) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Invalid argument` on `-a` | Adjust who-string; consult `man nfs4_setfacl` examples |
| Tool says not NFS | You are not on the NFS mountpoint path |
| Unsure which index `-x` expects | Run `man nfs4_setfacl` — index semantics are documented per release |

---

### Task 6 — Capstone log + full teardown

**Purpose:** Concatenate evidence and restore system.

```bash
{ echo '=== BEFORE ==='; cat /tmp/nfs4-acl.before 2>/dev/null; echo '=== AFTER ==='; cat /tmp/nfs4-acl.after 2>/dev/null; } > /tmp/nfs4-acl.log
wc -l /tmp/nfs4-acl.log
```

**Cleanup**

```bash
umount /mnt/nfs4lab
sed -i '\#/srv/nfs4lab#d' /etc/exports
exportfs -rav
systemctl restart nfs-server
rm -rf /srv/nfs4lab /mnt/nfs4lab /tmp/nfs4-acl.before /tmp/nfs4-acl.after
exit
```

**Human-Readable Breakdown:** Unmount before editing exports. Remove export line, re-export, restart for cleanliness, delete lab paths.

**Expected output:**

```text
42 /tmp/nfs4-acl.log
```

**Switches**

| Token | Meaning |
|---|---|
| `umount` | detach NFS |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| device busy | `cd /root`; `lsof /mnt/nfs4lab` |

---

## 🔍 NFSv4 ACL Decision Guide

```
Path is on NFS mount?
  │
  ├── yes, FSTYPE nfs4
  │     └── nfs4_getfacl / nfs4_setfacl
  │
  ├── yes, but only NFSv3 mount
  │     └── remount with vers=4 if server supports
  │
  └── local xfs/ext4
        └── getfacl / setfacl (POSIX labs)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Install `nfs-utils` + `nfs4-acl-tools`
- [ ] 02 Create `/srv/nfs4lab` + hello file
- [ ] 03 `/etc/exports` + `nfs-server` + `exportfs`
- [ ] 04 Mount `vers=4` at `/mnt/nfs4lab`, `findmnt` verify
- [ ] 05 `nfs4_getfacl`, optional `nfs4_setfacl -a`, remove via `-x` index
- [ ] 06 log file, umount, unexport, cleanup paths

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Used `getfacl` on NFS file | confusing output / errors | `nfs4_getfacl` |
| Production `no_root_squash` | security incident | lab-only pattern |
| Wrong ACE who-string | Invalid argument | align idmap |
| Forgot umount before deleting export | stale mounts | umount first |
| SELinux | RPC mount failures | booleans / logs |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize tool pair: **nfs4_*** vs **getfacl**.

**RHCE candidate**
- Shell `nfs4_setfacl` only when `posix_acl` module cannot target NFS semantics.

**SRE / Platform interview**
- Mention idmap / sssd when discussing who-strings — instant senior signal.

**DevOps**
- Document NFS version in PV CSI driver (`mountOptions`).

**AI / MLOps**
- Read-only corpora: NFSv4 + ACL = gate ticket for legal review.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 47 — ACL Support Check | mount inspection instincts |
| Lab 48 — Viewing POSIX ACLs | contrast grammar |
| Lab 53 — Removing POSIX ACLs | different flags |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)

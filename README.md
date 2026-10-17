# Ghost in the Release — Cytech CTF Write-Up

| Challenge | Category | Author | Final Flag |
| :--- | :--- | :--- | :--- |
| **Ghost in the Release** | Git Forensics / Cryptography / OSINT | C1ph3r | `Cryptix{v4ult_unl0ck3d_st4g3_tw0_n0t3s_0s1nt_g1st_w1ns}` |

---

## 📌 Challenge Overview

The challenge provides a compressed archive containing a Git repository: `payguard-vault.tar.gz`. While the repository's `main` branch appears completely clean and harmless, suspicious activity, hidden references, orphaned git commits, encrypted payload vaults, and an external OSINT trail hide the final flag.

**Flag Format:**
```text
Cryptix{<Fragment_A>_<Fragment_B>_<Fragment_C>}
```

---

## 🔍 Investigation & Forensic Walkthrough

### 1. Extracting Archive & Investigating Hidden Git Refs

Extracting the archive and exploring the Git repository structure:

```bash
tar -xzf payguard-vault.tar.gz
cd payguard-vault
git log --oneline
```

The main branch log appears clean. However, enumerating all Git references (`refs/`):

```bash
git for-each-ref
```

Uncovered three hidden references:
* `refs/pull/23/head` (Commit `d010abf`)
* `refs/notes/ir` (Commit `a7d6f92`)
* `refs/stash` (Commit `49c5f4c`)

---

### 2. Inspecting PR #23 & Git Notes

Examining commit `d010abf` (`refs/pull/23/head`):
```bash
git show d010abf
```
Found encrypted files and vault configuration:
* `vault/config.enc`
* `vault/stage2.enc`
* `config/vault.params`:
  ```ini
  CIPHER=aes-256-cbc
  ITER=200000
  MD=sha256
  ```

Viewing the Git note in `refs/notes/ir`:
```bash
git notes --ref=ir show
```
**Output:**
> *"pw is not in CI. it lives where branches never look."*

---

### 3. Finding Dangling / Unreachable Git Objects

The note hints at dangling or unreachable objects outside regular branch pointers:

```bash
git fsck --unreachable --dangling --no-reflogs
```

This revealed two critical unreachable objects:

1. **Dangling tag `v2.3.1` (Commit `3573063`):**
   * Contained an injected `package.json` postinstall hook: `node scripts/telemetry.js`.
   * `scripts/telemetry.js` decoded to an exfiltration endpoint: `https://cdn.qalam-metrics.example/u/7f3a`.
2. **Dangling commit `7779aec`:**
   * Commit message: `"ci: add env for release pipeline"`.
   * Contained `.env.ci`:
     ```text
     VAULT_PASS=Qal4m-R3l3as3!2026
     ```

---

### 4. Decrypting Stage 1

Using the discovered password `Qal4m-R3l3as3!2026` and parameters from `config/vault.params` (`aes-256-cbc`, PBKDF2, 200,000 iterations, SHA-256):

```bash
openssl enc -d -a -aes-256-cbc -pbkdf2 -iter 200000 -md sha256 \
  -pass pass:"Qal4m-R3l3as3!2026" -in vault/config.enc
```

**Output:**
```ini
OPERATOR=ph4nt0m-byte
FRAG_A=v4ult_unl0ck3d
STAGE2_FILE=vault/stage2.enc
STAGE2_PASS=Ph4nt0m!St4ge2-Kn0ck
```

* **Fragment 1:** `v4ult_unl0ck3d`
* **Operator Alias:** `ph4nt0m-byte`
* **Stage 2 Password:** `Ph4nt0m!St4ge2-Kn0ck`

---

### 5. Decrypting Stage 2

Decrypting `vault/stage2.enc` with `Ph4nt0m!St4ge2-Kn0ck`:

```bash
openssl enc -d -a -aes-256-cbc -pbkdf2 -iter 200000 -md sha256 \
  -pass pass:"Ph4nt0m!St4ge2-Kn0ck" -in vault/stage2.enc
```

**Output:**
```ini
FRAG_B=st4g3_tw0_n0t3s
NEXT=operator keeps working notes where GitHub shows public snippets
```

* **Fragment 2:** `st4g3_tw0_n0t3s`
* **Pivot Clue:** Check GitHub public snippets (Gists) for the operator `ph4nt0m-byte`.

---

### 6. OSINT Pivoting on GitHub Gists

Searching GitHub Gists for user **`ph4nt0m-byte`**:
* Found Public Gist: [https://gist.github.com/ph4nt0m-byte/570ad1896f103507b245575fdf422964](https://gist.github.com/ph4nt0m-byte/570ad1896f103507b245575fdf422964)

File content of `gistfile1.txt`:
```text
frag3: 0s1nt_g1st_w1ns
```

* **Fragment 3:** `0s1nt_g1st_w1ns`

---

## 🚩 Final Flag Construction

Combine all three fragments:
1. `v4ult_unl0ck3d`
2. `st4g3_tw0_n0t3s`
3. `0s1nt_g1st_w1ns`

```text
Cryptix{v4ult_unl0ck3d_st4g3_tw0_n0t3s_0s1nt_g1st_w1ns}
```

---

## 💡 Summary Attack Flow

```text
payguard-vault.tar.gz
└── git for-each-ref (Found pull/23, notes/ir, stash)
    └── git fsck (Uncovered orphaned commit 7779aec with .env.ci)
        └── VAULT_PASS=Qal4m-R3l3as3!2026
            └── Decrypt vault/config.enc
                ├── FRAG_A: v4ult_unl0ck3d
                ├── Operator: ph4nt0m-byte
                └── STAGE2_PASS: Ph4nt0m!St4ge2-Kn0ck
                    └── Decrypt vault/stage2.enc
                        ├── FRAG_B: st4g3_tw0_n0t3s
                        └── Gist Clue
                            └── OSINT: gist.github.com/ph4nt0m-byte
                                └── FRAG_C: 0s1nt_g1st_w1ns
                                    └── Cryptix{v4ult_unl0ck3d_st4g3_tw0_n0t3s_0s1nt_g1st_w1ns}
```

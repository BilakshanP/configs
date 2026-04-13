# Configs

## Bash

1. `.bashrc`

```sh
# User specific aliases and functions
if [ -d ~/.bashrc.d ]; then
    for rc in ~/.bashrc.d/*; do
        if [ -f "$rc" ]; then
            . "$rc"
        fi
    done
fi
unset rc
```

2. `.bashrc.d`

```sh
# aliases.sh
alias ..='cd ..'

alias c='clear'
alias q='exit'
alias e='exit'

alias py='python3'
```

```sh
# functions.sh
cd() {
    builtin cd "$@" && ls
}
```

## SSH

1. Key Gen

```sh
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id user@server # aoutomatic or manually append it to ~/.ssh/authorized_keys
```

2. `~/.ssh/config`

```ssh-config
Host nickname
    HostName example.com
    User username
    IdentityFile ~/.ssh/id_ed25519
```

**Note:** Approved keys are saved in `~/.ssh/authorized_keys`, with following format: `ssh-rsa pub-key identifier`.

3. Port forwarding

```sh
ssh -vfNR Y:localhost:X hostname  # Run on A, A:X -> M:Y
ssh -vfNL Z:localhost:Y hostname  # Run on B, M:Y -> B:Z
```

- `-v`: verbose, `-f`: background, `-N`: no-shell

- `-R`: it tells *Remote* to open the port `Y` and listen
- `-L`: it tells *Local* to open the port `Z` and listen

**Note**: You can replace `localhost` with: `127.0.0.1`, `0.0.0.0` or `hostname`.

## WSL

### Host (Windows) - `~/.wslconfig`

```toml
[wsl2]
firewall=false
autoProxy=false
networkingMode=mirrored
```

### Guest (WSL) - `/etc/wsl.conf`

```toml
# Refer to https://learn.microsoft.com/en-us/windows/wsl/wsl-config#wslconf
# for the full set of configuration options.
[boot]
systemd=false               # this will override resolv.conf

# [user]
# default=sketch

[network]
hostname=fedora
# generateHosts=false       # '/etc/hosts'
# generateResolvConf=false  # '/etc/resolv.conf'

[interop]
appendWindowsPath=false
```

**Note:** You might have to modify `/etc/host.conf` and `/etc/hosts`. Usually commenting out unwanted hostname should suffice.

#### Examples

1. `/etc/hosts`

```
127.0.0.1       localhost
127.0.0.1       fedora

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

2. `/etc/resolv.conf`

```
nameserver 8.8.8.8
# search .
```

**Note:** If `systemd` is enabled, it will overwrite this. It could be symlinked, the you would have to remove it first.

## Rust

### Better Releases

```toml
[profile.release]
lto = "fat"
strip = true
opt-level = 3
panic = "abort"
codegen-units = 1
```

### Quick Debug Builds

1. Install dependencies

```sh
sudo dnf install mold clang
cargo install sccache
rustup toolchain install nightly
rustup override set nightly # in project directory
rustup component add rustc-codegen-cranelift-preview --toolchain nightly
```

2. `.cargo/config.toml` (project directory)

```toml
[build]
rustc-wrapper = "sccache"

[unstable]
codegen-backend = true

[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
```

3. `Cargo.toml` (project-directory)

```toml
[profile.dev]
opt-level = 0
debug = 0
incremental = true
codegen-units = 256
lto = "off"
panic = "unwind"
split-debuginfo = "unpacked"
codegen-backend = "cranelift"

[profile.dev.build-override]
opt-level = 3
```

4. Verify

```sh
rustc --version       # should say nightly
mold --version
cargo build           # dev build, should be fast
sccache --show-stats  # confirm cache hits after build
```

## GPG

### Setup

1. Install pinentry

```sh
sudo dnf install pinentry-curses
```

2. `~/.gnupg/gpg-agent.conf`

```sh
pinentry-program /usr/bin/pinentry-curses
```

3. `~/.gnupg/dirmngr.conf`

```sh
keyserver hkps://keys.openpgp.org
```

4. Restart agents

```sh
gpgconf --kill gpg-agent
gpgconf --kill dirmngr
```

### Key Generation

```sh
gpg --full-generate-key
# Choose: ECC (sign and encrypt), Curve 25519, expiry as needed
```

GPG automatically creates two keys:

| Key | Type | Usage |
|---|---|---|
| Primary (`sec`) | ed25519 | **[S]** Sign + **[C]** Certify - your identity |
| Subkey (`ssb`) | cv25519 | **[E]** Encrypt only |

**Note:** If the subkey is compromised, revoke and replace it - your identity and web of trust stay intact. If the primary key is compromised, everything is lost.

### Backup

```sh
# Private key (contains both primary + subkey)
gpg --armor --export-secret-keys your@email.com > private_key.asc

# Revocation certificate (already generated at)
~/.gnupg/openpgp-revocs.d/<fingerprint>.rev
```

**Note:** Store both offline on an encrypted USB. The private key is passphrase-protected, but treat it as highly sensitive. No need to back up the public key - it is always derivable from the private key.

### Export
```sh
# Public key (to share with others)
gpg --armor --export your@email.com > public_key.asc

# Private key (for backup/migration)
gpg --armor --export-secret-keys your@email.com > private_key.asc

# Subkeys only (safer for day-to-day use on other machines)
gpg --armor --export-secret-subkeys your@email.com > subkeys.asc

# All public keys
gpg --armor --export > all_public_keys.asc
```

If you don't know your email, list keys first:

```sh
gpg --list-keys          # public keys
gpg --list-secret-keys   # private keys
```

Then export by key ID or fingerprint:

```sh
gpg --armor --export <fingerprint> > public_key.asc
```

### Publish

```sh
gpg --send-keys <fingerprint>
```

Then verify your email at https://keys.openpgp.org/upload so your key is searchable by name.

### Restore on Another Machine

```sh
gpg --import private_key.asc          # also derives and imports public key
gpg --list-keys your@email.com        # verify public key is present
gpg --list-secret-keys your@email.com # verify private key is present
gpg --edit-key your@email.com
# then: trust → 5 (ultimate) → save
```

### Update Password

```sh
gpg --edit-key your@email.com
```

Inside the GPG prompt:

```
passwd
save
```

### Update Expiry

Both the primary key and subkey have independent expiry dates - both must be updated.

```sh
gpg --edit-key your@email.com
```

Inside the GPG prompt:

```
expire      # updates primary key
274         # or 1y, 6m, etc.
y
key 1       # selects subkey (* appears next to ssb)
expire      # updates subkey
274
y
save
```

Then push updated expiry to keyserver:

```sh
gpg --send-keys <fingerprint>
```

### Inspect Encrypted Files

```sh
# Verify signature only
gpg --verify file.txt.asc

# Decrypt + auto-verify signature
gpg --decrypt file.txt.asc

# Inspect file metadata without decrypting (recipient, algorithm, etc.)
gpg --list-packets file.txt.asc
```

**Note:** To verify a signature, the recipient only needs the signer's public key - no secrets involved.

### Trust Model

The keyserver is just a convenience directory - not a trust authority. Security is in your hands:

```sh
# Always verify a contact's fingerprint out-of-band (phone, in person)
gpg --fingerprint their@email.com

# Set trust after verifying
gpg --edit-key their@email.com
# then: trust → 4 (full) or 5 (ultimate, only for your own keys) → save
```

### Daily Usage

```sh
# Get someone's key
gpg --search-keys their@email.com

# Encrypt (to them)
gpg --encrypt --armor --recipient their@email.com file.txt

# Encrypt + sign (to them)
gpg --encrypt --sign --armor --recipient their@email.com file.txt

# Decrypt
gpg --decrypt file.txt.asc

# Sign only
gpg --clearsign document.txt         # signature embedded
gpg --detach-sign document.txt       # separate .sig file

# Verify signature
gpg --verify document.txt.sig document.txt

# Refresh all keys (picks up revocations and expiry updates)
gpg --refresh-keys

# Revoke your key
gpg --import <fingerprint>.rev
gpg --send-keys <fingerprint>
```

### Delete Keys
 
```sh
gpg --delete-keys <fingerprint>                    # public key only
gpg --delete-secret-keys <fingerprint>             # private key only
gpg --delete-secret-and-public-keys <fingerprint>  # both at once
```

### One-off Verification
 
#### Method 1: Temporary Keyring (simple)
 
```sh
TMPGPG=$(mktemp -d)
GNUPGHOME=$TMPGPG gpg --import public_key.asc
GNUPGHOME=$TMPGPG gpg --verify file.sig file
rm -rf $TMPGPG
```
 
Completely isolated from `~/.gnupg` — no key touches your real keyring.
 
#### Method 2: `--no-default-keyring` (broken with keyboxd)
 
```sh
gpg --no-default-keyring --keyring ./tmp-keyring.gpg --import key.asc
gpg --no-default-keyring --keyring ./tmp-keyring.gpg --verify file.sig file
rm tmp-keyring.gpg
```
 
**Warning:** This is silently broken if `use-keyboxd` is set in `~/.gnupg/common.conf` (the default for new GnuPG 2.4 installations). The `--keyring` option is ignored without error, and the key gets imported into your real keyring anyway. This is a [known upstream issue](https://dev.gnupg.org/T7265). Use Method 1 instead.

### Password Based Encryption

```
gpg --symmetric --armor file.txt    # encrypt
gpg --decrypt file.txt.asc          # decrypt
```

### Armoring

`--armor` flag converts it from binary (.gpg) to plaintext (.asc)

1. `.gpg` to `.asc` (binary to text)

```sh
gpg --enarmor < file.gpg > file.asc
```

2. `.asc` to `.gpg` (text to binary)

```sh
gpg --dearmor < file.asc > file.gpg
```

### Directory Encryption

1. Single Step

```sh
# Tar + encrypt in one go
tar czf - folder/ | gpg --symmetric --armor > folder.tar.gz.asc

# Decrypt + extract in one go
gpg --decrypt folder.tar.gz.asc | tar xzf -
```

2. Multi Step

```sh
# Encrypt
tar czf folder.tar.gz folder/
gpg --symmetric --armor folder.tar.gz

# Decrypt
gpg --decrypt folder.tar.gz.asc > folder.tar.gz
tar xzf folder.tar.gz
```

### YubiKey (Recommended)

YubiKey 5 series supports OpenPGP - private key operations happen inside the device, the key never leaves it.

```sh
# Check if GPG sees the YubiKey
gpg --card-status

# Reset OpenPGP applet (wipes all keys and PINs)
ykman openpgp reset
```

- Default User PIN: `123456` (for decrypt/sign operations)
- Default Admin PIN: `12345678` (for settings)
- Change both immediately after setup

**Recommended setup:** Keep the primary key completely offline (cold storage), load only subkeys onto the YubiKey. Even if the YubiKey is stolen, your core identity is safe.

## Netcat (nc)

### File Transfer

```sh
# Receiver (run first)
nc -lvp 9001 > output_file

# Sender
nc -N <receiver-ip> 9001 < input_file
```

- `-l`: listen mode
- `-v`: verbose
- `-p`: port
- `-N`: close connection after EOF (so you don't have to Ctrl+D)

**Note:** nc transfers in plaintext. Always encrypt sensitive files with GPG first before sending over nc.

### Chat

```sh
# Side A
nc -lvp 9001

# Side B
nc <ip> 9001
```

Type and press Enter to send. Ctrl+C to quit.

### Port Scanning

```sh
nc -zv <ip> 80          # single port
nc -zv <ip> 20-100      # port range
```

- `-z`: scan only, don't send data

### Check if Port is Open

```sh
nc -zv <ip> <port>
# Connection succeeded = open
# Connection refused   = closed
```

### Netcat as a Proxy (Port Forwarding)

```sh
nc -lvp 9001 | nc <destination-ip> 9002
```

## Diff & Patches

### Comparing Files and Folders

1. Basic Diff

```sh
# Single file
diff file1.txt file2.txt

# Recursive folder comparison
diff -r folder1/ folder2/

# Summary only (which files differ, not the content)
diff -r --brief folder1/ folder2/
```

2. Unified Format (Recommended)

```sh
diff -u file1.txt file2.txt        # single file
diff -ur folder1/ folder2/         # recursive folders
```

**Why unified format?**
- Shows context around changes (lines before/after)
- Standard format for patches, git, and pull requests
- Much more readable than default diff output

**Example output:**
```diff
--- folder1/script.sh
+++ folder2/script.sh
@@ -10,7 +10,7 @@
 echo "Starting..."
 
 # Configuration
-DB_HOST="localhost"
+DB_HOST="production.example.com"
 DB_PORT=5432
```

**Key symbols:**
- `---` / `+++`: file headers (original / modified)
- `@@ -10,7 +10,7 @@`: line numbers and counts
- ` ` (space): unchanged context lines
- `-`: removed lines
- `+`: added lines

3. Other Useful Options

```sh
diff -w folder1/ folder2/      # ignore whitespace
diff -B folder1/ folder2/      # ignore blank lines
diff -i folder1/ folder2/      # ignore case
diff -u --color folder1/ folder2/  # colored output
```

### Creating Patches

1. From Files or Folders

```sh
# Single file
diff -u old_file.txt new_file.txt > changes.patch

# Folder
diff -ur old_folder/ new_folder/ > changes.patch
```

2. From Git

```sh
# Unstaged changes
git diff > changes.patch

# Staged changes
git diff --cached > changes.patch

# Between commits
git diff abc123 def456 > changes.patch

# Specific file
git diff file.txt > file.patch
```

### Applying Patches

1. Test Before Applying (Recommended)

```sh
# Dry-run to see what would change
patch --dry-run < changes.patch
```

2. Apply the Patch

```sh
# Simple case
patch < changes.patch

# If patch has directory paths (e.g., `a/` and `b/` prefixes)
patch -p1 < changes.patch
```

**`-p` flag:** strips directory levels
- `-p0`: use full path as-is
- `-p1`: strip first level (common for git patches)
- `-p2`: strip two levels

3. Reverse a Patch (Undo)

```sh
patch -R < changes.patch
```

### Using Patches with Git

1. Apply Without Committing

```sh
git apply changes.patch
```

2. Apply as a Commit

```sh
git am changes.patch
```

Creates a commit with metadata from the patch header.

3. Convert Git Diff to Patch

```sh
git format-patch -1 HEAD        # last commit as patch
git format-patch -3 HEAD        # last 3 commits as patches
```

### Workflow Example

```sh
# Developer A: Create changes
diff -ur myapp.py.old myapp.py > fix.patch

# Developer B: Receive fix.patch, then:

# 1. Verify the patch
patch --dry-run < fix.patch

# 2. Apply it
patch < fix.patch

# 3. Verify changes
cat myapp.py

# 4. If something went wrong, undo
patch -R < fix.patch
```

**Note:** Patches only work if the base file is similar. If the target file has drifted significantly, the patch may fail or apply incorrectly.

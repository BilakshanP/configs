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

### Publish

```sh
gpg --send-keys <fingerprint>
```

Then verify your email at https://keys.openpgp.org/upload so your key is searchable by name.

### Restore on Another Machine

```sh
gpg --import private_key.asc         # also derives and imports public key
gpg --list-keys your@email.com       # verify public key is present
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

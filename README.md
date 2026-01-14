# sshproxy_bitwarden

[![License: BSD 3-Clause](https://img.shields.io/badge/License-BSD%203--Clause-blue.svg)](LICENSE)

sshproxy is a small Bash script that fetches short-lived SSH keys from the NERSC
sshproxy service. It also supports PuTTY key output and optional ssh-agent
loading.

This version uses Bitwarden CLI ("bw") to supply both the account password and
the OTP/TOTP code.

Official documentation: https://docs.nersc.gov/connect/mfa/#sshproxy

## How it works

1. Builds the request based on CLI flags (scope, output path, proxy, etc.).
2. Uses Bitwarden to retrieve:
   - the account password
   - the TOTP code
3. Sends a POST request to the sshproxy server, authenticating with
   "<password><otp>".
4. Writes a private key, public key, and certificate into your `~/.ssh` (or
   chosen) directory.
5. Optionally loads the private key into `ssh-agent` until the cert expires.

## Requirements

- Bash
- `curl`, `ssh-keygen`, `mktemp`
- Bitwarden CLI: `bw`
- A Bitwarden item that contains:
  - the password for the sshproxy account
  - a TOTP secret for the same account

## Bitwarden configuration

The script reads Bitwarden via the CLI and supports either:

- an existing `BW_SESSION`, or
- interactive unlock if the vault is locked

The item name or ID is provided via `BW_ITEM` (default: `nersc`).

Examples:

- Use default item name:
  - `BW_ITEM` not set (uses `nersc`)
- Use a different item name:
  - `BW_ITEM="nersc sshproxy" ./sshproxy_bitwarden.sh`
- Use a specific item ID:
  - `BW_ITEM="<item-id>" ./sshproxy_bitwarden.sh`

### BW_SESSION behavior

The script runs `bw status --raw` to determine lock state:

- If unlocked: prints `Bitwarden already unlocked.` and proceeds.
- If locked: prints `Bitwarden vault is locked; unlocking...` and runs
  `bw unlock --raw`.
- On successful unlock: prints `Bitwarden unlock successful.`
- On unlock failure: exits with an error message.

If you want to pre-seed the session non-interactively:

```bash
export BW_SESSION="$(bw unlock --raw)"
./sshproxy_bitwarden.sh
```

## Usage

```bash
./sshproxy_bitwarden.sh [-u <user>] [-o <filename>] [-s <scope>] [-c <account>] [-p] [-a] \
              [-x <proxy-url>] [-U <server URL>] [-v] [-h]
```

Flags:

- `-u <user>`: NERSC username (default: `$USER`)
- `-o <filename>`: output private key path (default: `~/.ssh/nersc`)
- `-s <scope>`: scope name (default: `default`)
- `-c <account>`: collaboration account (sets scope and target user)
- `-p`: output PuTTY `ppk` format
- `-a`: add key to `ssh-agent` until cert expiration
- `-x <URL>`: use a SOCKS proxy for the sshproxy server
- `-U <URL>`: override sshproxy server URL
- `-v`: print version and exit
- `-h`: show help

## Output files

The script writes the following files next to the target key path:

- private key: `<idfile>`
- public key: `<idfile>.pub`
- certificate: `<idfile>-cert.pub`
- PuTTY key: `<idfile>.ppk` (when using `-p`)

Temporary files are created in the same directory as the final keys and cleaned
up on exit.

## Examples

Fetch a standard key using the default Bitwarden item:

```bash
./sshproxy_bitwarden.sh
```

Fetch a key with a custom scope:

```bash
./sshproxy_bitwarden.sh -s myscope
```

Fetch a PuTTY key and add it to ssh-agent:

```bash
./sshproxy_bitwarden.sh -p -a
```

Use a custom Bitwarden item:

```bash
BW_ITEM="nersc sshproxy" ./sshproxy_bitwarden.sh
```

## Troubleshooting

- `Bitwarden CLI (bw) is required but was not found in PATH`:
  Install the Bitwarden CLI and ensure it is in your PATH.

- `Failed to unlock Bitwarden; check master password`:
  The unlock failed. Try running `bw unlock --raw` manually and exporting
  `BW_SESSION`.

- Authentication failed:
  Your password or OTP might be wrong, or the Bitwarden item does not contain
  the correct password/TOTP.

## Security notes

- The script does not store passwords or OTPs on disk.
- The OTP is combined with the password only for the request to the sshproxy
  server.
- Keys are written with restrictive permissions and should be protected.

## License

See `LICENSE`.

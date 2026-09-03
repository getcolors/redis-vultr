# redis-vultr

Desired state for one [Redis](https://redis.io) 7.2 server on one Vultr
instance in Amsterdam, using the
[`redis`](https://github.com/getcolors/redis) Package Skill: one Docker
Compose service with `noeviction` and an append-only file, published on
loopback and the VPC address only, reached over an SSH tunnel, with RDB
backup sets in Cloudflare R2 and a rehearsed restore.

This repository holds `colors.yml`, the installed launcher, `.envrc`, and
`devenv.nix`. Everything else is generated (`.colors/`) or secret
(`.envrc.private`).

## Use

```sh
direnv allow                 # once, after the toolchain is installed
./green build                # render .colors/redis-vultr/ — no credentials needed
./green create --dry-run     # walk the workflow, skip every side effect
./green create               # converge for real
./green describe             # the host's last monitor result, over SSH
./green rehearse             # fresh set, restore it into a scratch instance, read back
```

## Connecting

```sh
ssh -L 6379:127.0.0.1:6379 redis-vultr
REDISCLI_AUTH=$(ssh redis-vultr cat /etc/redis/secrets/password) redis-cli -p 6379
ssh redis-vultr redis-status
```

The password is generated on the server and exists nowhere else.

## Credentials

All in the gitignored `.envrc.private`; the header of `colors.yml` lists
them. The state pair reaches no host. The backup pair — the `langfuse-backup`
bucket's scoped token, mapped by `.envrc` onto the names the package reads —
is the one pair that reaches the host.

## Recovery

| Failure | Recovers from | RPO |
|---|---|---|
| a Redis restart | the append-only file (proven on every converge) | ≤ 1 s |
| the host | the newest completed set under `redis-vultr/redis/` in `langfuse-backup` | 6 h |

`./green rehearse` proves the path and writes
`redis-vultr/.colors-recovery-verified` in the backup bucket.

## Deleting

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

Removes the machine, the firewall group, the VPC, the SSH config block and
the machine keypair. Removes nothing in R2.

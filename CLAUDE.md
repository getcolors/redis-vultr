# CLAUDE.md

Guidance for agents working in this deployment. Read
`~/code/getcolors/CLAUDE.md` first.

## What this is

Desired state only. No source code. `colors.yml` is the single file to edit;
everything else is either generated (`.colors/`), secret (`.envrc.private`), or
an installed copy of the [`redis`](https://github.com/getcolors/redis) Package
Skill launcher.

## Things specific to this deployment

- **One machine, one service.** `redis-vultr` in Amsterdam (`vc2-1c-2gb`,
  Ubuntu 24.04) in its own VPC (`10.60.0.0/24`), running Redis 7.2 published
  on loopback and the VPC address only. The firewall opens 22 alone; `ssh
  redis-vultr` reaches it through the block the package wrote.
- **The client path is a tunnel**: `ssh -L 6379:127.0.0.1:6379 redis-vultr`,
  then `redis-cli -p 6379` with the password read over SSH from
  `/etc/redis/secrets/password`. There is no DNS record and no public port.
- **Backups reuse the `langfuse-backup` bucket and its token.** Sets live
  under `redis-vultr/redis/<stamp>/`; `.envrc` maps the bucket's existing
  `COLORS_PAR_LANGFUSE_BACKUP_R2_*` pair onto the
  `COLORS_PAR_REDIS_BACKUP_R2_*` names the package reads.
- **Keygen mode**: `~/.ssh/redis-vultr` is generated and owned by the package.
  Adding `vultr-ssh-keys` here would switch the deployment to opt-out mode.

## The launcher is a copy

The root `./green` is a **copy** of `.agents/skills/package-redis-green/green`,
not a symlink. `npx skills update -p` rewrites the payload and leaves the root
file alone, so the project would keep running the old pin while the lockfile
claimed the new one. After every update:

```sh
npx skills update -p
cp .agents/skills/package-redis-green/green green
```

## Before any converge

```sh
./green build                # renders offline
./green create --dry-run     # walks the DAG, skips every side effect
```

## Afterwards

```sh
./green describe             # the host's last monitor result
./green rehearse             # fresh set, restore into a scratch instance, read back
ssh redis-vultr redis-status # monitor, completed sets, recovery marker, container
```

## Never

- Read or print `.envrc.private`.
- Edit, read as source, or commit `.colors/`.
- Export `COLORS_PAR_PROFILE`; the package refuses to run when it is set.
- Weaken `compute-prevent-destroy` in committed desired state.
- Run `create`, `rehearse` or `delete` against this deployment without
  explicit authorization.
- Touch objects under `redis-vultr/` in the backup bucket by hand.

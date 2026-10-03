# Discourse migration — status 2026-10-03

Work paused after Phase 0. The full plan with checkboxes is `DISCOURSE.md`
in [`vzekc/cluster-infra`](https://github.com/vzekc/cluster-infra/blob/main/DISCOURSE.md);
the manifests live in `clusters/production/apps/discourse/`.

## Where things stand

- Test install at discourse.k8s.classic-computing.de: DB, Redis, PVCs
  up and empty; `discourse-tls` issued via DNS-01 for both the `.k8s.`
  and the bare hostname.
- Flux image automation commits `stable-*` tag bumps to main as
  fluxcdbot.
- web and sidekiq pods sit in `CreateContainerConfigError` because the
  `discourse-secrets` block in `sealed-secrets.yaml` is still an empty
  placeholder.
- `discourse-woltlab-sync` CronJob is `suspend: true`; its image is
  published once woltlab-to-discourse#1 is merged.
- Production still runs on the old Docker VM.

## How to pick up

1. Rotate the SMTP password for `discourse` on `mail.classic-computing.de`.
2. Seal `discourse-secrets` with `cluster-infra/scripts/seal.sh`
   (command in `DISCOURSE.md`, Phase 1), paste the `encryptedData` into
   the first block of `clusters/production/apps/discourse/sealed-secrets.yaml`,
   push to main.
3. Watch `kubectl -n prod get pod -l app=discourse -w`, then continue
   with Phase 2 in `DISCOURSE.md`.

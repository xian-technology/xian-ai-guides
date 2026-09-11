# Agent instructions: launch the first Xian testnet node

Give this entire file to the agent responsible for the coordinator's node.
The companion `testnet-join-agent.md` is for participants. These are operator
instructions, not an announcement of a live network.

## Objective and defaults

Set up a fresh, operator-managed testnet with one initial validator, a separate
treasury wallet, and a public handoff that lets other people sync their own
nodes. Use the ordinary `local` genesis contract bundle with freshly generated
keys. Its name describes a reusable bundle; it does not restrict the resulting
chain to one machine. Do not join the checked-in `testnet` preset or reuse its
fixture keys/genesis.

Use the signed, digest-pinned **integrated Docker node image**, prepared with
`xian-cli` and deployed to Linux with `xian-deploy`. The integrated image runs
CometBFT and ABCI together. This is the maintained released-artifact path and
avoids independently building different runtimes on participant servers.
The split image is useful for separate service management; source builds are
for development or a deliberately coordinated unreleased test.

Enable BDS/PostgreSQL indexing, read-only GraphQL with GraphiQL, and the node
dashboard by default. Keep paid metered transactions, serial execution,
periodic empty blocks with a five-second interval, no pruning, and no state
sync. Actual block cadence also depends on consensus timings. Prometheus/Grafana
sidecars remain disabled; `monitoring_profile=none` does not disable the node
dashboard. Keep RPC, GraphQL, dashboard, and metrics private. Preserve blocks
so early participants can replay from genesis.

The pinned Ansible deployment supports BDS and the dashboard, but does not
deploy PostGraphile. The explicit GraphQL companion procedure below is part
of setup, not an optional follow-up. There is no `--enable-graphql` CLI flag.

Use manual validator admission initially. Anyone with the handoff can run a
syncing full node; becoming a validator requires registration and governance.
Do not generate additional genesis validators for people who have not joined.

## Inputs and scope

Inspect the available environment first. Obtain only missing deployment inputs:

- a fresh network name and chain ID; use a distinct operator-managed namespace
  and a new generation suffix after any genesis reset
- target Linux host, SSH user/access, sudo availability, public P2P hostname/IP,
  and permitted firewall changes
- private control-machine working directory and backup destination
- coordinator contact and where the public handoff will be shared

Use an always-on Linux host with stable addressing and SSD storage. A practical
starting budget for light testing is 4 vCPU, 8 GiB RAM, and 100 GiB SSD; this is
an operator estimate, not a throughput guarantee. Account for PostgreSQL,
GraphQL, and dashboard memory and indexed-history disk growth. Check disk,
BDS spool backlog, and load; increase capacity before these limits are reached.
Allow inbound TCP 26656 and the required administrative access. Keep host RPC
26657 on loopback; use an SSH tunnel for administration and treasury transfers.
A public RPC service can be added deliberately with its own access/rate policy.
Verify exposure from another machine, including Docker-published ports.
Use SSH tunnels for dashboard port 8080 and GraphQL/GraphiQL port 5000.
PostgreSQL port 5432 stays on the internal Docker network without a host port.

When asked to execute this guide, complete preparation, deployment, and checks
within the owner's authorized scope. Do not buy infrastructure or send a public
announcement without authorization. Never overwrite an existing node home,
reset a running chain, or upload an old validator signing state as routine setup.
Keep current-code defaults local in the source repositories. This network's
identity and endpoints belong in its explicit operator manifest and handoff.

## 1. Freeze the runtime and prepare tooling

Authoritative repositories are under `https://github.com/xian-technology/`:
`xian-stack`, `xian-cli`, `xian-configs`, `xian-abci`, `xian-contracting`,
`xian-py`, and `xian-deploy`. Read each relevant `AGENTS.md` and `README.md`.
In an existing sibling workspace, start with `xian-meta/docs/WORKSPACE.md`.
Use fresh clones or isolated worktrees for pins; do not reset someone's checkout.

The inspected baseline on 2026-09-12 is stack **v0.3.3** and CLI package
**0.2.1**, using the refreshed CLI source/lock commit below.
The published v0.3.3 image index references are:

```text
ghcr.io/xian-technology/xian-node@sha256:8acda26890d55d7aedcdbec9478943adacd82cbc81d1c0ca5abf1fde78a78d5f
ghcr.io/xian-technology/xian-node-split@sha256:62ecf64958285bd63761860e54f8aa3ad5132fe75bb5ab0a637c9c8d4b25100b
```

Download `release-manifest.json`, `image-release.json`, and their Sigstore
bundles from the [v0.3.3 release](https://github.com/xian-technology/xian-stack/releases/tag/v0.3.3).
Verify signatures using `xian-stack/docs/RELEASES.md`; verify the image digest
as well. Do not silently substitute `latest` or select versions independently
on each host. If choosing another coordinated release, revalidate this guide
and record the complete replacement pin set before generating genesis.

For v0.3.3, the release manifest pins these genesis/runtime inputs:

| Repository | Commit |
| --- | --- |
| xian-abci | `9699bab13384a7a9aa37aa2ac22fe5ae4b4040da` |
| xian-configs | `6c35fc9c50a29c7a25bc145c32ee9803c21105df` |
| xian-contracting | `d250e988019556fe872c56732b55dc3ca04f10d8` |
| xian-py | `293931ceaac36d6b84990c9690089ef384b612f6` |

Compare these with the verified downloaded manifest. Use these controller pins
in the public `operator-tooling.json` sidecar:

| Repository | Commit | Purpose |
| --- | --- | --- |
| xian-stack | `6ac0259f9d8fe4a8fb7730f35bb0052b45e33969` | v0.3.3 release |
| xian-cli | `62db44a1b24bb0c3a7987265548e380f951e4c1e` | CLI 0.2.1 with refreshed dependency lock |
| xian-deploy | `a60a1b0523cb04c3ac6635f67ef055774d2bcf7e` | inspected deployment playbooks |

The pinned deploy revision runs its configuration container with a Python
entrypoint override and handles null BDS enqueue-error fields in health reports.
Both fixes are required for the verified deployment flow.

The CLI **v0.2.1 tag predates the dependency-lock refresh**. Do not combine that
tag's old frozen lock with the v0.3.3 component checkouts. Pin the CLI commit
above; its command surface is unchanged. CLI and deploy are not component
entries in the stack release manifest, so retain their separate pins. Record
Python version and all resolved preparation dependencies too.

The matching package versions are `xian-tech-abci==0.9.3`,
`xian-tech-contracting==1.1.3`, `xian-tech-vm-core==0.1.2`,
`xian-tech-py==0.5.2`, and `xian-tech-compiler-core==0.2.0`.
The Contracting/VM update fixes deterministic, metered storage scans; it affects
contract execution even with parallel execution disabled. Start every node on
this same runtime. Do not assume mixed v0.3.2/v0.3.3 operation is safe.

If a network was already started using the earlier guide, do not rerun creation
or replace its genesis. Treat adoption of this execution change as a coordinated
upgrade requiring history/replay compatibility validation and an agreed switch
plan. An image update alone does not repair previously divergent history.

On the control machine, use Python 3.14 and `uv`. Prepare the pinned sibling
workspace, then run `uv sync --frozen --group dev` in `xian-cli`. Use that
environment's `xian` executable for all following commands. Genesis creation
needs the native VM/compiler packages; verify their installed versions and
native imports in that exact environment. After changing native source pins,
rebuild/reinstall the native extension if needed; editable package metadata
alone does not prove a previously built binary contains the fixes. Run the
existing `xian-contracting/tests/integration/test_vm_storage_scans.py` regression
with the preparation interpreter, explicitly passing `-m optional_native`
(the default pytest selection excludes these cases). From the pinned
`xian-contracting` checkout, set `CLI_PYTHON` to the absolute Python executable
in the prepared CLI environment and run:

```bash
"$CLI_PYTHON" -m pytest -q -m optional_native \
  tests/integration/test_vm_storage_scans.py
```

Require all 12 cases to pass for this baseline; skipped/deselected cases are
not validation. Do not silently fall back to a legacy execution engine.
A published CLI installation is also usable
if its resolved ABCI, compiler, VM, Contracting, and SDK versions have been
checked against the selected release. Pinning just the CLI package is not a
complete runtime pin. The SDK version matters on the **controller** too: the
node image cannot update the CLI's client-side nonce and receipt handling.

Install compatible Ansible tooling on the control machine following
`xian-deploy` requirements, and run its `make validate`. The target host only
needs the deployment prerequisites and released Docker runtime, not sibling
source checkouts. The controller may be the same Linux machine if using a
proper local Ansible inventory; otherwise use SSH.

## 2. Create private keys and the network exactly once

Set these shell variables to real absolute paths/accepted values before use:
`RUN_DIR`, `STACK_DIR`, `CONFIGS_DIR`, `NETWORK`, `CHAIN_ID`, `INTEGRATED_IMAGE`,
and `SPLIT_IMAGE`. `RUN_DIR` is a fresh private directory outside any public
repository. Image variables must be the verified references above or the
accepted replacement release. `XIAN_CONFIGS_DIR` must reference the pinned
configs checkout, including during genesis generation.

```bash
umask 077
mkdir -p "$RUN_DIR/keys" "$RUN_DIR/homes" "$RUN_DIR/release"
chmod 700 "$RUN_DIR" "$RUN_DIR/keys"
export XIAN_CONFIGS_DIR="$CONFIGS_DIR"
xian client wallet generate \
  --private-key-out "$RUN_DIR/keys/treasury.key" \
  > "$RUN_DIR/treasury-public.json"

xian network create "$NETWORK" \
  --base-dir "$RUN_DIR" --chain-id "$CHAIN_ID" \
  --genesis-bundle local --validator-selection-mode manual \
  --bootstrap-node bootstrap-1 --generate-validator-key \
  --founder-private-key-file "$RUN_DIR/keys/treasury.key" \
  --node-image-mode registry \
  --node-integrated-image "$INTEGRATED_IMAGE" \
  --node-split-image "$SPLIT_IMAGE" \
  --block-policy-mode periodic --block-policy-interval 5s \
  --tx-fee-mode paid_metered --no-parallel-execution-enabled \
  --enable-bds --enable-dashboard --no-enable-monitoring \
  --dashboard-host 127.0.0.1 --dashboard-port 8080 \
  --no-enable-pruning \
  --stack-dir "$STACK_DIR" --configs-dir "$CONFIGS_DIR" \
  --home "$RUN_DIR/homes/bootstrap-1" --init-node
```

No all-in-one development template is needed. Inspect the resulting profile;
set `operator_profile` to `shared_network`, `monitoring_profile` to `none`,
and `advanced.cometbft.allow_cors` to `false`. Add the verified release manifest
as `node_release_manifest` in both the network manifest and bootstrap profile.
Validate both with the pinned CLI model readers. These are profile/manifest
edits, not edits to the genesis state.

Verify `services.bds.enabled=true`, `services.dashboard.enabled=true`, and
`services.monitoring.enabled=false`. Keep BDS connection fields such as host,
user, password, DSN, RPC URL, and spool directory empty/default in the profile:
`xian-deploy` owns those bindings through inventory/vault. Do not copy a
development template's database credentials into a shared profile.

The explicit founder option matters: omitting it makes the first validator key
the founder. Keep the treasury key on the controller/in the owner's secret
store; only the node's own consensus/P2P keys belong on its server.
`validator_key_info.json` contains private material despite its name. Do not
print it, include it in a handoff, or ask anyone to paste it into chat.

Verify generated `genesis.json` before starting:

- `chain_id` is the accepted chain ID; there is exactly one CometBFT validator
- the same validator account is the active genesis member in contract state
- in `abci_genesis.genesis`, `currency.balances:<treasury public key>` is
  **11,111,111.10 XIAN** (JSON fixed-decimal encoding may show `11111111.1`)
- the treasury account differs from the validator account and P2P node ID
- the fixed fixture account was replaced as currency/foundation founder

This keeps the standard supply/allocation behavior: other allocations remain
in `dao` and `team_lock`. It does not give the treasury the entire supply, and
does not deploy extra contracts to spend the `team_lock` allocation. The
treasury's directly spendable balance is sufficient for this testnet. Do not
edit serialized genesis balances by hand: genesis includes state hashes and a
signature. A different allocation requires rebuilding before launch.

The generator/packager and node-home writer can format equivalent JSON
differently. Compare the generated and installed genesis as parsed JSON; they
must be identical. Use the bootstrap home's `config/genesis.json` as the
canonical launch **bytes**, copy those exact bytes to the network's
`genesis.json`, and calculate the launch SHA-256 from that file. Do this before
distribution. It is a serialization normalization, not a state change.

## 3. Configure and deploy the bootstrap node

In the prepared home's `config/config.toml`, set `[p2p].external_address` to
the actual publicly reachable `host:26656`; keep `seed_mode = false` and
`pex = true`. This node validates blocks and will also be a persistent peer.
Keep its own persistent-peer list empty until another peer exists. Do not put
its own ID in its peer list. Disable wildcard CORS in the effective config.
Use a TOML-aware edit and recheck the effective config after deployment.

Create a private archive of the **contents** of the prepared home:

```bash
tar -czf "$RUN_DIR/bootstrap-1-home.tar.gz" \
  -C "$RUN_DIR/homes/bootstrap-1" .
chmod 600 "$RUN_DIR/bootstrap-1-home.tar.gz"
```

Create a private inventory, replacing the host/user and controller paths.
Copy `inventories/example/group_vars/all/main.yml` into the private inventory's
`group_vars/all/main.yml` to supply the full deployment variable set. Merge the
`vars` values below into that copied file as well: Ansible group-vars files can
override inline group values. In particular, replace its public RPC binding
and `xian_node_home_replace: true` defaults.

```yaml
all:
  children:
    xian_nodes:
      hosts:
        bootstrap.example.org:
          ansible_user: deploy
          ansible_become: true
          xian_node_profile: /absolute/private/run/nodes/bootstrap-1.json
          xian_node_home_archive: /absolute/private/run/bootstrap-1-home.tar.gz
      vars:
        xian_project_name: xian
        xian_deploy_root: /srv/xian-testnet
        xian_deploy_topology: integrated
        xian_node_home_replace: false
        xian_rpc_bind_host: 127.0.0.1
        xian_rpc_port: 26657
        xian_p2p_bind_host: 0.0.0.0
        xian_p2p_port: 26656
        xian_dashboard_bind_host: 127.0.0.1
        xian_dashboard_port: 8080
        xian_bds_user: xian
        xian_bds_password: "{{ vault_xian_bds_password }}"
        xian_postgres_image: "postgres:17.10@sha256:REPLACE_WITH_VERIFIED_DIGEST"
```

Run from the pinned `xian-deploy` checkout; `INVENTORY` is the absolute inventory
path. Use the repo's Ansible configuration and example variable structure,
retaining the loopback RPC and nonreplacement overrides above. Confirm the
remote destination is unused before uploading; `replace: false` alone does
not prevent extracting an archive over existing files.

Before deployment, resolve and record the PostgreSQL image's real immutable
digest in the inventory and public tooling record. Replace the example marker;
`shared_network` validation rejects unpinned PostgreSQL images. Generate a
strong, unique database password per node and supply `vault_xian_bds_password`
through the private Ansible Vault/secret-store configuration (including its
unlock mechanism). Never put the password in the public bundle. Retain the
persistent PostgreSQL data and BDS recovery spool under `xian_deploy_root`.

```bash
ansible-playbook -i "$INVENTORY" playbooks/bootstrap.yml
ansible-playbook -i "$INVENTORY" playbooks/push-home.yml
ansible-playbook -i "$INVENTORY" playbooks/deploy.yml
ansible-playbook -i "$INVENTORY" playbooks/health.yml
```

Do not pass `xian_genesis_bundle`, genesis regeneration, or validator-key
replacement options to deployment. The prepared home already has the correct
genesis and keys. Never start this validator home on the controller while it
also runs remotely. After initial upload, use status/restart/upgrade workflows,
not `push-home`, for routine operation.

The inspected deploy configurator resets `external_address` to empty. After
`deploy.yml`, set `[p2p].external_address` in the **remote** home's config to the
accepted address using a private idempotent post-deploy task, then run
`playbooks/restart.yml` with the same inventory. Repeat that task after future
deploy/upgrade reconciliation; the restart playbook itself does not regenerate
config. Verify the resulting setting and run `health.yml` again. Do not claim
that editing the prepared archive alone persists this field.

Inspect the rendered Compose config and running image digest. Confirm restart
policy, persistent bind mount, P2P reachability, RPC binding, and the unchanged
genesis hash. Remote
checks use Ansible and the target's RPC; a local `xian node status` does not
inspect the remote Docker daemon.

### Required GraphQL companion and indexed-service checks

Run this after the main Ansible deployment has healthy PostgreSQL and BDS.
The pinned `xian-deploy` commit does not include GraphQL in its Compose file or
health report. Do not mark GraphQL complete solely because `health.yml` passes.
Keep the companion in a separate private directory/project so subsequent
Ansible rendering does not erase it.

1. Use the selected stack commit's `docker/postgraphile.Dockerfile`, the entire
   `docker/postgraphile/` directory (including its npm lock and Xian presets),
   and `docker/postgres/init-postgraphile-role.sh`. The coordinator builds the
   GraphQL image once per supported architecture on a build/controller machine.
   Use a copied Dockerfile with its `FROM node:24-alpine` resolved to a recorded
   immutable base digest; do not edit the pinned source checkout. Build using
   that stack checkout as context and retain `npm ci --omit=dev`. The released
   node images do not themselves contain the PostGraphile server.
2. Record the stack commit, base digest, copied Dockerfile hash, and resulting
   image identity in the public tooling record. Deliver an image archive with
   SHA-256 and expected image ID per architecture, or an accepted registry
   digest. Participants verify and load/pull this artifact; they do not rebuild
   from a floating Node image. This is a query-service image, not a replacement
   consensus image. Do not claim that the custom artifact has the official
   node-release signature; authenticate it through the coordinator's handoff.
3. On each server, create a mode-0700 `GRAPHQL_DIR`, for example
   `/srv/xian-testnet/graphql`, with a `secrets/` subdirectory. Copy the pinned
   role-init script there. Use a separate random hex password for a local
   `xian_graphql` read-only database role. Generate mode-0600
   `secrets/graphql-init.env` containing `PGHOST=postgres`, `PGPORT=5432`,
   `PGDATABASE=xian`, `PGUSER=xian`, `PGPASSWORD=<this node's BDS password>`,
   `XIAN_BDS_USER=xian`, `XIAN_POSTGRAPHILE_USER=xian_graphql`,
   `XIAN_POSTGRAPHILE_PASSWORD=<read-only password>`, and
   `XIAN_POSTGRAPHILE_STATEMENT_TIMEOUT_MS=10000`. Populate secrets locally from
   the private store without printing them. If inventory changes the database
   owner/name, use those actual values consistently.
4. Set `POSTGRES_IMAGE` to the same verified digest as the main deployment and
   run the script against its database, after PostgreSQL is ready:

```bash
docker run --rm --network xian_xian-db \
  --env-file "$GRAPHQL_DIR/secrets/graphql-init.env" \
  --mount "type=bind,src=$GRAPHQL_DIR/init-postgraphile-role.sh,dst=/init-role.sh,readonly" \
  "$POSTGRES_IMAGE" /bin/bash /init-role.sh
```

The script grants SELECT access to existing and future BDS tables, with
restricted role attributes and query timeouts. Retain/recreate its private
credential file only for administration; do not mount it into GraphQL.

5. Create a separate mode-0600 `secrets/graphql.env` with only
   `POSTGRAPHILE_CONNECTION=postgres://xian_graphql:<read-only password>@postgres:5432/xian`.
   Random hex passwords avoid URI/Compose escaping problems. Verify SQL writes
   are denied for this role. GraphQL must never use the BDS owner credential.
6. Save this `compose.yml` in `GRAPHQL_DIR`, and place the verified image
   digest/reference (or loaded `sha256:<image ID>`) in `.env` as `GRAPHQL_IMAGE`.
   Preload/pull that image explicitly; the template will not fetch a substitute.

```yaml
services:
  postgraphile:
    image: ${GRAPHQL_IMAGE:?set the verified GraphQL image}
    pull_policy: never
    restart: unless-stopped
    init: true
    stop_grace_period: 30s
    mem_limit: 1g
    env_file:
      - ./secrets/graphql.env
    environment:
      GRAPHILE_ENV: production
      POSTGRAPHILE_HOST: 0.0.0.0
      POSTGRAPHILE_PORT: "5000"
      POSTGRAPHILE_SCHEMA: public
      POSTGRAPHILE_DISABLE_DEFAULT_MUTATIONS: "true"
      POSTGRAPHILE_SIMPLE_COLLECTIONS: omit
      POSTGRAPHILE_BODY_SIZE_LIMIT_BYTES: "1048576"
      POSTGRAPHILE_SCHEMA_WAIT_TIMEOUT_SECONDS: "300"
    command: ["/usr/src/app/start-postgraphile.sh"]
    ports:
      - "127.0.0.1:5000:5000"
    networks: [db, app]
networks:
  db:
    external: true
    name: xian_xian-db
  app:
    external: true
    name: xian_xian-net
```

The network names assume `xian_project_name: xian` in the main inventory.
Verify them with Docker and substitute the actual names if the project name
was changed. Keep the main PostgreSQL service as the only database; the
companion must not start a second database or node.

```bash
docker compose --project-directory "$GRAPHQL_DIR" \
  -p xian-graphql -f "$GRAPHQL_DIR/compose.yml" config -q
docker compose --project-directory "$GRAPHQL_DIR" \
  -p xian-graphql -f "$GRAPHQL_DIR/compose.yml" up -d
```

7. Check BDS through target-loopback dashboard
   `http://127.0.0.1:8080/api/abci_query/bds_status`: require a healthy database
   and worker, increasing `indexed.indexed_height`, and a draining backlog.
   Wait until indexing reaches the smoke transaction's block before querying
   it through GraphQL; consensus finality and indexed visibility are separate.
8. Open the dashboard at `http://127.0.0.1:8080/`, and inspect its chain/BDS
   status. Query `http://127.0.0.1:5000/graphql` with
   `{"query":"{ allBlocks(first: 1, orderBy: HEIGHT_DESC) { nodes { height blockHash } } }"}`.
   Require actual indexed rows matching the node, not merely HTTP 200. Verify
   GraphiQL at `http://127.0.0.1:5000/graphiql` and inspect the schema to confirm
   no default mutations are exposed. These are server-loopback URLs; use SSH
   forwarding from the controller/browser, with alternate local ports if needed.
9. Record separate companion status/restart commands and logs. Main Ansible
   health checks do not test GraphQL. For a full shutdown, stop the companion
   before `xian-deploy` removes its networks; bring it back after the main
   deployment is healthy. Restart/reboot checks must cover all three services
   and preserve PostgreSQL data. Back up the database consistently and retain
   the BDS recovery spool; a node-home-only backup does not include indexed data.

## 4. Verify the chain and treasury

Read target-loopback RPC using SSH or a tunnel. For example, bind controller
port 27657 to target `127.0.0.1:26657` with SSH, and set
`RPC_URL=http://127.0.0.1:27657`. Verify `/status` reports the correct network,
increasing heights over multiple observations, and `catching_up=false`.
Read `node_info.id` there: this is the **P2P node ID**, not the validator
address or the treasury account. Form `BOOTSTRAP_PEER=<node ID>@<host>:26656`.

Extract `TREASURY_ADDRESS` from `treasury-public.json`, then:

```bash
xian client query balance --node-url "$RPC_URL" "$TREASURY_ADDRESS"
xian client call --node-url "$RPC_URL" validators get_active_validators
xian client call --node-url "$RPC_URL" validators get_policy_config
```

Generate a disposable recipient with `xian client wallet generate
--private-key-out ...`, save its public address as `RECIPIENT`, and verify a
committed transfer and recipient balance:

```bash
xian client tx transfer --node-url "$RPC_URL" --chain-id "$CHAIN_ID" \
  --private-key-file "$RUN_DIR/keys/treasury.key" --mode commit \
  "$RECIPIENT" 10
xian client query balance --node-url "$RPC_URL" "$RECIPIENT"
```

Retain the receipt/hash and check execution success, not just mempool
acceptance or inclusion in a block. SDK 0.5.2 recovers receipts using actual
block execution results; unavailable results must remain unresolved, never be
reported as success. If submission times out, use `xian client query tx
--node-url "$RPC_URL" <tx-hash>` and reconcile the receipt and account nonce
before retrying. A `NonceReservationError` in SDK automation means an earlier
automatic submission is unresolved. Do not bypass it with a guessed explicit
nonce or a fresh process. Serialize treasury submissions across CLI processes
and agents: SDK coordination is per client, not a shared wallet lock across
independent processes. Never regenerate keys/genesis as a timeout remedy.

Fund the bootstrap validator account with a modest treasury transfer for
future governance transaction fees: it is distinct from the treasury and does
not receive the treasury allocation. Keep funds sufficient for fees on every
active governance signer.

## 5. Prepare the public participant handoff

Add the bootstrap persistent peer to the network manifest's
`p2p.persistent_peers`. Keep a real seed service in `p2p.seeds` only if one is
actually operated. Participants must also carry the bootstrap entry in their
node profile: do not assume network-level persistent peers propagate through
every preparation/deployment path.

Package into a fresh output directory:

```bash
xian network package-operator-bundle "$NETWORK" \
  --base-dir "$RUN_DIR" \
  --network-manifest "$RUN_DIR/networks/$NETWORK/manifest.json" \
  --configs-dir "$CONFIGS_DIR" \
  --bootstrap-seed "$BOOTSTRAP_PEER" \
  --output "$RUN_DIR/public-handoff"
```

Here the packager's `bootstrap-seed` field is a bootstrap contact, not a
request to enable CometBFT seed mode. In the public manifest, remove the
ordinary validator from `p2p.seeds` and retain it in `persistent_peers`.
Use the companion agent guide's explicit join flow. The stock generated
participant scripts enable parallel execution and simulation and use a seed
argument; remove those two scripts from this handoff, and update its README
and `operator-bundle.json` file list accordingly. Do not distribute scripts
whose defaults differ from the chosen network posture.

Add `testnet-join-agent.md`, verified release assets/signature bundles,
`operator-tooling.json`, and `NETWORK-INFO.md`. Include in the latter:

- network name, chain ID, coordinator contact, and reset/upgrade policy
- exact genesis SHA-256 and manifest SHA-256
- both image digests, component/tooling pins, and supported host architectures
- PostgreSQL digest and verified GraphQL image/archive provenance per architecture
- bootstrap persistent peer, block/fee/execution policy, retention posture
- treasury **public** address and the process for requesting test XIAN
- optional validator admission process and current registration bond
- RPC access instructions; use participants' local RPC if there is no public RPC
- private GraphQL/GraphiQL and dashboard access, indexed-service checks, and
  lifecycle/backup instructions for the GraphQL companion

The standard registration bond is 100,000 XIAN; fund an approved validator
candidate with **100,100 XIAN** initially for that bond plus fee headroom,
checking live fees before sending. Ordinary users/full nodes need no bond and
can receive a smaller test balance.

Copy the bootstrap home's exact launch `config/genesis.json` into the handoff
after packaging, which may change JSON formatting again. First require parsed
JSON equality, then verify the handoff and installed genesis are byte-identical.
After all public files are final, create `SHA256SUMS` over an explicit public
file allowlist, archive the handoff, and record its SHA-256 externally.
Checksums alone are not authentication: deliver the expected archive/genesis
hash through the coordinator's trusted channel or a verified signature.
Inspect the archive listing and scan locally for secret values **without
printing them**. Never include `keys/`, `homes/`, private inventories, node-home
archives, databases, signing state, or logs. Archive only this clean directory.
Include the non-secret GraphQL companion Compose template, pinned role-init
script, and image manifest/download checksums in the allowlist; credentials
are generated separately on every participant node.

## 6. Rehearse joining, then admit validators if requested

Use the participant guide on a second independently reachable host before
calling the handoff ready. Confirm persistent peering, replay from genesis,
same block hash/app hash at the same height, matching transfer results, and a
successful restart. A single-node check cannot validate Internet peering.
Test XIAN goes to account addresses; a server/P2P node ID is not a recipient.

Include a storage-scan convergence check in the launch rehearsal. The pinned
stack's `scripts/localnet_storage_scan_checks.py` contains the tested source
and assertions for native, foreign, and dynamic hash scans with deleted entries.
Use that probe in the rehearsal and compare its persisted ordered results and
common-height block/app hashes on both hosts. For seed 0 the probe expects
`[[1,2,3,4,5],[1,2,3,4,5],[1,2,3,4,5]]`. Record probe contract names and receipt
hashes in the handoff's rehearsal evidence. The release gate phase is
`11-storage-scan-determinism`; retain its evidence from the accepted release
as well. A successful balance transfer alone does not exercise this fix.

For manual admission, first wait until the candidate is fully synced and
registered. Use the bootstrap validator's account to call
`validators.propose_vote(type_of_vote="add_member", arg=<candidate account>)`.
The proposer already votes yes; with one active validator this can finalize
immediately. Later proposals require the current active validators' threshold.
Use the same trusted local signing-file approach described in the participant
guide; the treasury is not itself an active governance signer.

Check both contract membership and CometBFT's effective validator set after
activation. Add candidates one at a time and verify their signatures. With
equal power, two validators require both online, and three require all three;
four allow one offline while retaining more than two-thirds voting power.
Do not activate an unavailable node. The one-node bootstrap has no redundancy.

Report the final chain ID, genesis hash, public handoff path/hash, treasury
public address and balance, node/peer identities, deployed digest, successful
transfer receipt, remote health/restart evidence, and the second-host join
result. Include BDS indexed height/lag, successful GraphQL query and GraphiQL
checks, dashboard URL/health, and the separate query-service image identity.
Report missing external checks plainly. Keep encrypted backups of
keys and a consistent stopped-node home/signing state; never restore stale
signing state into an already-used validator or run duplicate signers.

## Source anchors and validation boundary

This guide was checked against `xian-cli` network/node/client commands,
`xian-abci/src/xian/genesis_builder.py` and `node_setup.py`,
`xian-configs/contracts/currency.s.py` and `validators.s.py`, and
`xian-deploy` inventory/profile/runtime roles. The v0.3.3 release assets were
retrieved to check the recorded digests. Source references are navigation aids;
the verified release pins govern the actual launch.

Local preparation was rehearsed with disposable keys: genesis generation,
distinct treasury allocation, bundle packaging, and participant initialization
with an identical genesis and explicit persistent peer. For this revision,
the 12 native storage-scan cases and 19 SDK submission-recovery/wire cases
passed in the preparation environment, and the documentation site built.
The indexed default profiles were initialized locally and the GraphQL Compose
templates passed configuration validation. No live PostgreSQL/GraphQL/dashboard
deployment is implied by those preparation checks.
This document alone
does not establish a deployed network, verified image signatures, or a
successful two-host Internet rehearsal; the executing agent must perform them.

# Agent instructions: join an operator-managed Xian testnet

Give this entire file, the coordinator's public operator bundle, and its
independently received expected hash to the participant's agent. It is
self-contained for joining; the coordinator's private setup files are not
needed. The full-node flow does not require becoming a validator.

The default participant node includes BDS/PostgreSQL indexing, read-only
GraphQL/GraphiQL, and the dashboard, so participants can inspect the chain.
Parallel transaction execution and Prometheus/Grafana sidecars remain disabled.
GraphQL is an explicit companion service because the pinned Ansible deployment
does not deploy it automatically when BDS is enabled.

## Objective and required inputs

Deploy this participant's own Linux node using the coordinator's exact genesis
and released Docker image, sync from the bootstrap peer, and verify a test
transfer. Generate this participant's private keys locally. Do not generate a
new network, genesis, or allocation, and do not use the checked-in `testnet`
preset merely because it has that name.

Obtain missing inputs before dependent work:

- public bundle location plus a trusted expected archive/genesis hash
- `NETWORK-INFO.md`: network name, chain ID, bootstrap persistent peer,
  coordinator contact, release and tooling pins
- participant Linux host, SSH user/access, sudo, permitted firewall changes,
  public P2P address if available, and a unique local node name
- private controller working directory, backup location, and whether the owner
  wants only a full node or also validator admission

An always-on host with SSD is appropriate. For light testing, 4 vCPU, 8 GiB RAM,
and 100 GiB SSD is a starting estimate; monitor actual load and storage growth.
Budget for PostgreSQL, GraphQL, dashboard memory, and the BDS recovery spool.
The released image supports Linux amd64 and arm64; use the coordinator's
verified image index digest for native platform selection.

Continue through authorized setup and checks. Do not purchase hosting or send
messages to the coordinator without authorization; prepare the public message
for the participant to send when needed. Never overwrite another node home or
reset an existing chain to resolve a setup error.

## 1. Verify the handoff and prepare compatible tools

Inspect an archive listing before extracting into a fresh directory. Reject
absolute/traversal paths, unexpected links, node homes, or private key files.
Verify the archive against the hash obtained from the coordinator through a
trusted channel, then verify the internal `SHA256SUMS` and exact genesis hash.
A checksum downloaded beside an untrusted bundle is not proof of its source.

Confirm:

- manifest network name and genesis `chain_id` match `NETWORK-INFO.md`
- manifest genesis is the colocated, materialized `genesis.json`
- `node_image_mode` is `registry`; image references include `@sha256:`
- the release assets and `operator-tooling.json` agree with the manifest
- bootstrap contact is `<P2P node ID>@<host>:26656`; it is not a validator key
- no snapshot/state-sync trust inputs are assumed when replaying from genesis
- pinned PostgreSQL and GraphQL image provenance plus the non-secret GraphQL
  companion template and role-init script; never the coordinator's credentials

Verify release asset/image signatures using the selected stack release's
`docs/RELEASES.md`. Do not replace a pinned image with a tag or update the
release independently. The inspected baseline on 2026-09-12 is stack v0.3.3,
CLI package 0.2.1, ABCI 0.9.3, Contracting 1.1.3, VM core 0.1.2, SDK 0.5.2,
and compiler core 0.2.0. The controller CLI pin is
`62db44a1b24bb0c3a7987265548e380f951e4c1e` (refreshed lock), not the older
v0.2.1 tag. The inspected deploy pin is
`23aab505ed5594d4c07e3e414d07989812ead356`.

The coordinator's accepted handoff remains authoritative. If it is still based
on v0.3.2, resolve the release mismatch with the coordinator rather than silently
joining with v0.3.3. The storage-scan fix changes execution behavior even in
serial mode; do not assume mixed versions or replay of an existing network
are compatible. An upgrade needs a coordinated, validated plan. Never replace
an existing network's genesis to make it match this guide.

On the control machine use Python 3.14 and `uv`. Repositories are under
`https://github.com/xian-technology/`. Prepare pinned sibling checkouts of
`xian-cli`, `xian-stack`, `xian-configs`, `xian-abci`, `xian-contracting`, and
`xian-py`, plus the pinned `xian-deploy`. Read their `AGENTS.md`/`README.md`.
Use component refs from the release manifest and CLI/deploy refs from
`operator-tooling.json`; avoid changing existing checkouts. Run
`uv sync --frozen --group dev` in the pinned `xian-cli` checkout and use its
environment's `xian` executable. Install Ansible per `xian-deploy` and run
`make validate` there. The remote host runs released images and needs no Xian
source tree. An isolated published CLI environment is also acceptable if its
resolved dependencies match the handoff; do not assume a CLI pin pins them all.
Verify installed versions and native imports in the actual CLI environment.
The server's image does not update the controller's SDK: require SDK 0.5.2 for
this baseline's nonce/receipt fixes. If native packages were built before the
source update, rebuild/reinstall and run the existing Contracting storage-scan
regression with the CLI environment's interpreter: from `xian-contracting`,
run `python -m pytest -q -m optional_native tests/integration/test_vm_storage_scans.py`.
Select that interpreter explicitly; the default pytest marker selection excludes
these tests. Require 12 passed cases for this baseline. Metadata alone is
insufficient to verify a native binary.

## 2. Prepare a unique node home

Set `RUN_DIR` to a new absolute private path, `BUNDLE_DIR` to the verified
public bundle, `STACK_DIR`/`CONFIGS_DIR` to pinned checkouts, `NETWORK` and
`CHAIN_ID` to the handoff values, and `NODE_NAME` to a unique local name.

```bash
umask 077
mkdir -p "$RUN_DIR/keys" "$RUN_DIR/homes"
chmod 700 "$RUN_DIR" "$RUN_DIR/keys"
mkdir -p "$RUN_DIR/networks/$NETWORK"
cp -R "$BUNDLE_DIR/." "$RUN_DIR/networks/$NETWORK/"
BUNDLE_DIR="$RUN_DIR/networks/$NETWORK"
export XIAN_CONFIGS_DIR="$CONFIGS_DIR"

xian network join "$NODE_NAME" \
  --base-dir "$RUN_DIR" --network "$NETWORK" \
  --network-manifest "$BUNDLE_DIR/manifest.json" \
  --generate-validator-key --node-image-mode registry \
  --enable-bds --enable-dashboard --no-enable-monitoring \
  --dashboard-host 127.0.0.1 --dashboard-port 8080 \
  --no-enable-pruning --no-parallel-execution-enabled \
  --tx-fee-mode paid_metered \
  --stack-dir "$STACK_DIR" --configs-dir "$CONFIGS_DIR" \
  --home "$RUN_DIR/homes/$NODE_NAME"
```

This explicit flow assumes the coordinator retained the indexed paid/serial
posture. If the handoff specifies different execution or fee settings, reconcile
them before initializing; participants must not invent chain policy overrides.
Do not run the stock generated participant scripts blindly: their defaults
enable parallel execution and use seed discovery.

Before initialization, edit `nodes/<NODE_NAME>.json` with a JSON-aware tool:

- put the coordinator's bootstrap entry in `p2p.persistent_peers`
- retain only actual seed services in `p2p.seeds`; an ordinary bootstrap
  validator need not be listed as a seed
- set `operator_profile=shared_network`, `monitoring_profile=none`, and
  `advanced.cometbft.allow_cors=false`
- verify the two digest-pinned image references, block policy, fee policy,
  serial execution, disabled state sync, BDS and dashboard enabled, and
  Prometheus/Grafana disabled

Keep deployment-owned BDS host/user/password/DSN/RPC/spool fields empty/default
in the profile; credentials and bindings belong in private inventory/vault.
If the participant explicitly requests a smaller full node, use
`--no-enable-bds --no-enable-dashboard` and skip GraphQL/database setup. That
opt-out changes local query services, not the shared genesis or execution policy.

Do this explicitly even if the manifest contains persistent peers: the CLI's
node initialization and Ansible deployment consume profile peer settings.
Validate the profile with the pinned CLI model reader, then initialize:

```bash
xian node init "$NODE_NAME" --base-dir "$RUN_DIR" \
  --network "$BUNDLE_DIR/manifest.json" \
  --configs-dir "$CONFIGS_DIR"
```

The profile stores a network name, not the explicit manifest path passed to
`network join`. The copy under `RUN_DIR/networks/NETWORK` makes later name-based
commands resolve the accepted network. Keep this directory and all relative
manifest assets together; do not let commands fall back to a canonical preset.
Verify the installed `config/genesis.json` matches the coordinator's hash
exactly.
Never copy the coordinator's `priv_validator_key.json`, `node_key.json`, or
`data/priv_validator_state.json`. Your home must have a different validator
key and P2P node ID. `validator_key_info.json` contains secrets: do not print
or share its full contents.

In the home's `config/config.toml`, verify `[p2p].persistent_peers`,
`seed_mode=false`, `pex=true`, and P2P listen address on port 26656. Set
`external_address` to this participant's own reachable `host:26656` if it has
one; do not advertise the bootstrap node's address as your own. Behind NAT,
configure forwarding for inbound peering. A full node may sync using outbound
peers without a publicly reachable address; report this limitation. Keep
`addr_book_strict=true` on the Internet. Change it only for a deliberate
private-address network, not to conceal a broken public peer address.

## 3. Deploy the released integrated image

On a remote Linux host, use `xian-deploy` from the controller. Archive only
your prepared home's contents into a private archive:

```bash
tar -czf "$RUN_DIR/node-home.tar.gz" -C "$RUN_DIR/homes/$NODE_NAME" .
chmod 600 "$RUN_DIR/node-home.tar.gz"
```

Create a private inventory with real host/user and absolute controller paths.
Copy `inventories/example/group_vars/all/main.yml` to the private inventory's
`group_vars/all/main.yml` for the complete deployment variable set. Merge the
`vars` values below into that copied file too; the group-vars file can override
inline group values. Replace its public RPC binding and home-replacement
defaults explicitly.

```yaml
all:
  children:
    xian_nodes:
      hosts:
        participant.example.org:
          ansible_user: deploy
          ansible_become: true
          xian_node_profile: /absolute/private/run/nodes/participant-1.json
          xian_node_home_archive: /absolute/private/run/node-home.tar.gz
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

Use the pinned repo's `ansible.cfg` and example variable structure, keeping
these RPC/nonreplacement overrides. Confirm the remote home is unused before
upload: archive extraction can overwrite files even with replacement disabled.
Replace the PostgreSQL digest marker with the coordinator's verified image
pin; `shared_network` validation requires a digest. Generate a unique strong
`vault_xian_bds_password` for this host and configure private Vault/secret-store
access for Ansible. Do not reuse a coordinator password or put secrets in the
profile/public bundle. Preserve PostgreSQL data and the BDS recovery spool.
Run from the `xian-deploy` checkout with `INVENTORY` set to the inventory path:

```bash
ansible-playbook -i "$INVENTORY" playbooks/bootstrap.yml
ansible-playbook -i "$INVENTORY" playbooks/push-home.yml
ansible-playbook -i "$INVENTORY" playbooks/deploy.yml
ansible-playbook -i "$INVENTORY" playbooks/health.yml
```

Do not set deployment variables that regenerate genesis or replace validator
keys. Never start the prepared home simultaneously on controller and server.
For a same-machine Linux controller use an explicit local Ansible inventory.

Allow administrative access and inbound P2P TCP 26656 as authorized. Keep RPC
26657, dashboard 8080, GraphQL/GraphiQL 5000, and application/Comet metrics private.
PostgreSQL 5432 must have no host-published port. Access the query UI and
dashboard through SSH tunnels.
Check actual Docker-published bindings and reachability from another machine.
Confirm the runtime uses the handoff's digest, persistent home mount, restart
policy, and the unchanged genesis. Deployment reconciliation resets
`[p2p].external_address` to empty in the inspected release. If this participant
advertises a public address, apply it in the remote home's config with a private
idempotent post-deploy task, run `playbooks/restart.yml` with the same inventory,
and recheck health. Repeat after future deploy/upgrade reconciliation; restart
alone does not regenerate the config. Verify persistent peers as well.
After initial installation,
use status/restart/upgrade workflows; do not upload the initial home again.

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

## 4. Verify syncing and receive test XIAN

Use target-loopback RPC through SSH, or tunnel controller port 27657 to target
`127.0.0.1:26657`; set `RPC_URL=http://127.0.0.1:27657` in that case.
Check `/status`, `/net_info`, and `/block?height=<agreed height>`:

- the network is the handoff chain ID
- at least one persistent connection exists to the expected peer
- heights advance and `catching_up` eventually becomes `false`
- your node and the coordinator agree on block hash and header app hash at
  the **same height**, not two independently queried latest heights

If the coordinator provides the storage-scan rehearsal probe and receipt,
also compare its persisted ordered result on your node. The v0.3.3 reference is
`xian-stack/scripts/localnet_storage_scan_checks.py`; for seed 0 its reader
stores `[[1,2,3,4,5],[1,2,3,4,5],[1,2,3,4,5]]`. Compare at an agreed state/height
and record the receipt and common-height hashes. Do not substitute a successful
transfer for this execution check.

Use Ansible remote health/status too. A controller-local `xian node status`
does not check the server's Docker daemon. If replay fails because the peer
pruned old blocks, request an archival peer or a separately authenticated
snapshot/state-sync handoff. Do not enable state sync with invented trust
height/hash or skip missing blocks.

Generate a separate wallet for ordinary test transfers:

```bash
xian client wallet generate \
  --private-key-out "$RUN_DIR/keys/test-wallet.key" \
  > "$RUN_DIR/test-wallet-public.json"
```

Prepare a coordinator message containing only the wallet public address,
network/chain ID, P2P node ID, and sync status. Request a modest test balance.
No coins are needed merely to sync a full node. Never send private key files.
Once funded, query the wallet and perform an agreed small return/test transfer:

```bash
xian client query balance --node-url "$RPC_URL" "$WALLET_ADDRESS"
xian client tx transfer --node-url "$RPC_URL" --chain-id "$CHAIN_ID" \
  --private-key-file "$RUN_DIR/keys/test-wallet.key" --mode commit \
  "$AGREED_RECIPIENT" 1
```

Set `WALLET_ADDRESS` from the generated public JSON and `AGREED_RECIPIENT` from
the coordinator. Keep fee headroom. Check committed execution, receipt hash,
and recipient balance on both nodes. Resolve a submission timeout by querying
the transaction before retrying. SDK 0.5.2 requires actual block execution
results to recover a receipt; missing results do not mean success. If SDK
automation raises `NonceReservationError`, reconcile the earlier transaction
hash/account nonce before further sends; do not override a nonce or restart a
client merely to bypass it. Keep one submitting process/agent per wallet at a
time because independent CLI processes do not share a nonce reservation lock.
Restart the server runtime and verify that it
keeps its identity, resumes syncing, and retains the same chain history.

At this point a full-node-only request is complete. Report that the node is
synced and **not an active validator**; do not imply otherwise.

## 5. Optional validator admission

Perform this only when validator participation is requested and coordinated.
First confirm the node is synced, reachable, and will remain online.

In the inspected contract, `validators.register` uses `ctx.caller` as the
validator identity; it has **no separate `consensus_key` parameter**. Therefore
registration must be signed by the account whose public key matches this
node's consensus Ed25519 public key. A separate ordinary wallet may receive
rewards but cannot register an unrelated node key by passing it as an argument.
Do not import the consensus key into a browser wallet or send it to the
coordinator. Use only a trusted local CLI/signing process under the operator's
control for validator administration. Keep the separate test wallet separate.

Extract only `validator_public_key_hex` for the public admission request.
The standard local bundle has a 100,000 XIAN registration bond. Check the live
`validators.registration_fee` state via the supported RPC/SDK state query
and call `validators.get_policy_config()` before funding/approval; do not
assume policy remains unchanged. Request bond plus fee headroom at this
validator account (100,100 XIAN for the unmodified baseline).

For the baseline CLI, locally create a mode-0600 signing file containing only
the generated metadata's `validator_private_key_hex`, using a secure file
writer without logging it. Set `VALIDATOR_SIGNING_FILE` to that private path.
Do not pass a full JSON key file to `--private-key-file`, which expects seed
hex. The following example assumes the verified 100,000 XIAN bond:

```bash
xian client tx send --node-url "$RPC_URL" --chain-id "$CHAIN_ID" \
  --private-key-file "$VALIDATOR_SIGNING_FILE" --mode commit \
  currency approve --kwargs-json '{"amount":100000,"to":"validators"}'

xian client tx send --node-url "$RPC_URL" --chain-id "$CHAIN_ID" \
  --private-key-file "$VALIDATOR_SIGNING_FILE" --mode commit \
  validators register --kwargs-json '{}'
```

Wait for each successful committed receipt before the next transaction.
Registration escrows the bond; it does not activate the validator in manual
mode. Query `validators.get_validator(account=<validator public key>)` and
confirm pending status. Prepare the public admission request for the coordinator.
The active validators propose/vote `add_member`; participants cannot self-approve.
If the accepted network uses a different selection mode, follow its live
eligibility/bond/rebalance policy rather than this manual step.

After approval, confirm your key appears in both
`validators.get_active_validators()` and CometBFT `/validators` (paginate when
needed), that power is nonzero, and that recent block commits contain your
validator signature. Allow for validator-update activation delay. Merely
starting the node or seeing a registration receipt is not sufficient.

The early equal-power network needs all validators online at sizes two and
three; at four, one may be offline while more than two-thirds power remains.
Coordinate activation and downtime. Never run the same validator key in two
homes/processes or restore stale signing state. Back up keys and a consistent
stopped-node state privately. For an active validator's departure, follow
`announce_leave()`/`leave()` and verify removal before shutting down.

## Completion report

Return a short report with chain ID and genesis hash, image digest, node name,
P2P/validator public identities, peer count, sync height, common-height hash
comparison, health/restart result, wallet public address, and test receipt.
Include BDS indexed height/lag, a successful indexed GraphQL query, GraphiQL
and dashboard access/health, and the query-service image identity. For an
explicit service opt-out, identify which local query features are unavailable.
State either `synced full node` or `active validator` with the corresponding
evidence. Identify any pending funding, admission, or external reachability
check explicitly. Include local operational paths and restart/status commands
for the owner, but no secrets. This guide is not evidence of a live deployment;
the executing agent must gather those results.

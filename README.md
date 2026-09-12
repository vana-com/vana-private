# Vana PoS Network Validator Setup

This guide will help you set up a validator node for the Vana Proof-of-Stake (PoS) network using Docker.

## Prerequisites

- Docker: [Install Docker](https://docs.docker.com/get-docker/)
- Docker Compose: [Install Docker Compose](https://docs.docker.com/compose/install/)
- OpenSSL: Install via your package manager:
  ```bash
  sudo apt-get install openssl  # Debian-based systems
  brew install openssl          # macOS
  ```
- Hardware: Ensure your system meets the [minimum hardware requirements](https://docs.vana.org/vana/core-concepts/roles/propagators#hardware-requirements) for running a Vana Propagator (Validator)

## Quick Start

1. Clone the repository:
   ```bash
   git clone https://github.com/vana-com/vana.git
   cd vana
   ```

2. Prepare the developer tools and local pre-push scan:
   ```bash
   .github/scripts/setup-developer.sh
   ```

3. Configure your environment:
   ```bash
   # For Moksha testnet
   cp .env.moksha.example .env
   # OR for Mainnet
   cp .env.mainnet.example .env

   # Edit .env with your preferred text editor
   ```

4. Start your node:
   ```bash
   docker compose --profile init --profile node up -d
   ```

5. Verify your node is running:
   ```bash
   # View logs for key services
   docker compose logs -f geth    # Execution client
   docker compose logs -f beacon  # Consensus client
   ```

> **💡 Tip**: Check out the [Fast Syncing](#fast-syncing) section below to significantly speed up your initial node sync!
>
> **Note**: If services fail to start, check the configuration validation logs:
> ```bash
> docker compose logs check-config-node
> ```

## Validator Setup

Once your node is fully synced, follow these steps to set up and run a validator.

> **⚠️ IMPORTANT**: Running a validator on the Vana network requires whitelisting. Please join our [Discord](https://discord.com/app/invite-with-guild-onboarding/withvana) to request validator permissions before proceeding with setup.

1. Configure validator settings in `.env`:
   ```bash
   # Configure validator settings
   WITHDRAWAL_ADDRESS=<your_withdrawal_address>
   FEE_RECIPIENT_ADDRESS=<your_suggested_fee_recipient>
   DEPOSIT_RPC_URL=<your_rpc_url>
   DEPOSIT_CONTRACT_ADDRESS=<contract_address>
   ```

2. Set up validator keys:

   If you have existing keys:
   - Place keystore files in `./secrets`
   - Create `wallet_password.txt` and `account_password.txt` in `./secrets`
   ```bash
   # Import existing keys
   docker compose run --rm validator-import
   ```

   If you need new keys:
   ```bash
   # Generate new keys
   docker compose run --rm validator-keygen

   # Import generated keys
   docker compose run --rm validator-import
   ```

   If you need to generate new keys for more than one validator, 
   set `NUM_VALIDATORS` to the number of new validators in `.env`.

3. Submit deposits (if not done already):
   ```bash
   # Add deposit private key
   echo "your_private_key" > ./secrets/deposit_private_key.txt

   # Submit deposits
   docker compose run --rm submit-deposits
   ```

4. Configure validator statistics reporting:
   ```bash
   # Set stats configuration in .env
   STATS_SERVER_URL=http://stats.vana.org
   INSTANCE_NAME="Your Validator Name"
   VALIDATOR_PUBLIC_KEY=0x...  # Your validator's public key
   ```

   Get your validator's public key using either method:
   ```bash
   # Method 1: From deposit data
   cat ./secrets/deposit_data-*.json | jq -r '.[0].pubkey'

   # Method 2: List validator accounts
   docker compose --profile init run --rm validator accounts list --wallet-dir=/vana/wallet
   ```

   View your node's statistics at [stats.vana.org](https://stats.vana.org). Your node will appear under the name specified in `INSTANCE_NAME`.

5. Start the validator:
   ```bash
   docker compose --profile validator up -d
   ```

   > **Note**: If the validator fails to start, check the configuration validation logs:
   > ```bash
   > docker compose logs check-config-validator
   > ```

### Validator Voluntary Exit

To voluntarily exit your validator, ensure your beacon node is fully synced and run:

```bash
docker compose --profile init run --rm validator-exit
```

This service requires `account_password.txt` and `wallet_password.txt` in the `secrets` folder.

> **Important**: Exiting your validator is permanent and cannot be reversed. This only signals your intent to exit - it does not withdraw funds. For withdrawal functionality, see the [withdrawal documentation](https://docs.prylabs.network/docs/wallet/exiting-a-validator).

## Validator Approval Process

Before proceeding with setup, you must get your validator whitelisted:

1. Join the [Vana Discord](https://discord.com/app/invite-with-guild-onboarding/withvana) and request validator permissions

2. Generate your validator keys following the [Validator Setup](#optional-validator-setup) section

3. Submit your validator's public key for whitelisting through Discord

4. Wait for confirmation before proceeding with deposits and starting your validator
## Fast Syncing

There are two recommended methods to speed up your initial node sync:

### 1. Checkpoint Sync

Checkpoint sync is the recommended way to quickly sync your node. You will need:

1. A trusted beacon node that serves checkpoint syncing (checkpoint sync URL)
2. A specific block root and epoch number that you wish to sync to (weak subjectivity checkpoint)

Configure them in your `.env` file:

```bash
# Use appropriate URL for your network
TRUSTED_BEACON_NODE_URL=http://archive.vana.org:3500

# Replace with actual checkpoint from a trusted source
WEAK_SUBJECTIVITY_CHECKPOINT=0x0000...0000:0  # block root:epoch number
```

Then uncomment these lines in `docker-compose.yml` under the `beacon` service:

```yaml
- --weak-subjectivity-checkpoint=${WEAK_SUBJECTIVITY_CHECKPOINT}
- --checkpoint-sync-url=${TRUSTED_BEACON_NODE_URL}
- --genesis-beacon-api-url=${TRUSTED_BEACON_NODE_URL}
```

### 2. Syncing from a Public Snapshot

For a fast initial sync, you can also restore a recent publicly available backup. See the [Backup and Restore](#backup-and-restore) section for detailed instructions on downloading and restoring snapshots.

## Configuration

### Environment Variables

Edit the `.env` file to configure your node. Key variables include:

- `NETWORK`: Choose between `moksha` (testnet) or `mainnet`
- `CHAIN_ID`: Network chain ID
- `EXTERNAL_IP`: Your node's external IP address
- `BEACON_IMAGE`: The beacon node image — Vana's build of Prysm (see below)
- Various port configurations for different services

Ensure all required variables are set correctly before proceeding.

### Deposit contract switch (mainnet)

Mainnet moves to a new deposit contract at **block 10,229,958**:

| | |
|---|---|
| Current contract (from the switch block onward) | `0xB98aafa6684aef18AEf518772F01F5aE0DA5eA4C` |
| Retired contract (holds the history before it) | `0x17BbE91c315Bf14f38F6D35052a827cadfFe184e` |
| Switch block | `10229958` |

The new contract was deployed already seeded with the retired contract's final state, so the deposit
index and merkle tree continue across the boundary. Nothing is rebuilt and no node resyncs: below
the switch block a node scans the retired contract, at and above it the current one. A node syncing
from genesis still reconstructs the same deposit history.

**This requires Vana's build of Prysm**, `ghcr.io/vana-com/prysm-beacon-chain`, which is the default
for the `beacon` service and is pinned by `BEACON_IMAGE` in `.env`. Upstream Prysm does not
understand the two new chain-config keys and will not run against this config.

Only the beacon node is forked. The validator and prysmctl stay on upstream Prysm, now `v7.1.8` via
`PRYSM_VERSION` — the same release the Vana build is based on. The deposit contract switch is
entirely beacon-node work, and the upstream validator reads this config file fine; it reports the
two new keys as unknown fields in the same non-fatal log line it already emits for `BLOB_SCHEDULE`,
and applies everything else.

Everything needed is already in `networks/mainnet/config.yml` and `docker-compose.yml`, so upgrading
is an image change:

```bash
docker compose pull beacon validator
docker compose up -d beacon validator
```

> **Note**: `PRYSM_VERSION` moves from `v5.1.0` to `v7.1.8`, so the validator crosses two major
> versions. Back up your validator database first — see
> [Backup and Restore](#backup-and-restore) — since the slashing protection history lives there
> and downgrading is not supported.

Two things to know when upgrading a node that has already been running:

- Your database records the deposit contract address it last ran with, and the node refuses to
  start when that no longer matches the config (`database contract is 0x17bbe91c... but tried to
  run with 0xb98aafa6...`). The `beacon` service passes `--clear-deposit-contract` to handle this.
  It clears only that record and writes it again from the config on the same start — all other data
  is kept, so this is **not** a resync.
- If your node has already scanned past the switch block, it rewinds its deposit log scan once and
  logs `Rewinding deposit log scan`. This is expected, and happens only once.

To confirm the switch is active, look for this line as the scan crosses the boundary:

```
Deposit log scan crossing the deposit contract switch block
  retiredContract=0x17BbE9... currentContract=0xB98aaf... switchBlock=10229958
```

Deposits submitted after the switch go to the current contract, so `DEPOSIT_CONTRACT_ADDRESS` in
`.env` must be `0xB98aafa6684aef18AEf518772F01F5aE0DA5eA4C`. The retired contract no longer accepts
deposits.

## Verifying Your Setup

After starting your services, you can check the logs to ensure everything is running correctly:

1. View logs for all services:
   ```bash
   docker compose logs
   ```

2. View logs for specific key services:
   ```bash
   docker compose --profile=init --profile=node logs -f geth
   docker compose --profile=init --profile=node logs -f beacon
   docker compose --profile=init --profile=node logs -f validator
   ```

3. To follow logs in real-time and filter for specific patterns:
   ```bash
   docker compose --profile=init --profile=node logs -f geth 2>&1 | grep 'Looking for peers'
   docker compose --profile=init --profile=node logs -f beacon 2>&1 | grep 'Synced new block'
   docker compose --profile=init --profile=node logs -f validator 2>&1 | grep 'Submitted new'
   ```

When reviewing logs, look for:

- Geth (execution layer): Messages about peer connections and syncing progress
- Beacon Chain: Indications of connection to the network and slot processing
- Validator: Messages about duties being performed and contributions submitted

If you see error messages or unexpected behavior in the logs, refer to the troubleshooting section or seek support.

## Troubleshooting

If you encounter issues:

1. Ensure all configuration files are present and correctly formatted.
2. Check individual service logs for specific error messages.
3. Verify that your `.env` file contains all necessary variables.
4. Run the configuration check:
   ```bash
   docker compose run --rm check-config
   ```
5. For connection issues, check your firewall settings and ensure the necessary ports are open.
6. If services fail to start, try restarting them individually:
   ```bash
   docker compose restart <service_name>
   ```

## Security Considerations

- Securely store your validator keys and never share them.
- Regularly update your node software to the latest version.
- Monitor your validator's performance and status regularly.

For additional help or to report issues, please open an issue in the GitHub repository or contact the Vana support team.

## Advanced Usage

The `docker-compose.yml` file provides several additional capabilities for managing your Vana PoS validator node. Here are some useful commands and their purposes:

### Profiles

Different profiles are available for various operations:

- `init`: Initialize clients, generate secrets
- `node`: Run the main node services
- `validator`: Run validator-specific services
- `manual`: For manual operations like key generation
- `delete`: Delete data, e.g. to reset the chain so you can re-sync
- `public`: Expose APIs securely via Caddy reverse proxy (ports 80/443)

You can combine profiles as needed. Whenever a service depends on another service, you must include the dependent profile.

 For example, to start the node, you must include the `init` and `node` profiles:
```bash
docker compose --profile init --profile node up -d
```

For example, to run the node with public API access:
```bash
docker compose --profile init --profile node --profile public up -d
```

Or to start/stop just the API gateway:
```bash
docker compose --profile init --profile node --profile public up -d caddy
docker compose --profile init --profile node --profile public down caddy
```

### Key Management

Generate validator keys (interactive process):
```bash
docker compose --profile init run --rm validator-keygen
```

Import validator keys:
```bash
docker compose run --rm validator-import
```

### Deleting Data

To delete all data/ (does not remove generated secrets/):

```bash
docker compose --profile delete run --rm delete-all
```

To delete execution or consensus layer data:
```bash
docker compose --profile delete run --rm delete-geth
docker compose --profile delete run --rm delete-beacon
```

### Configuration Check

Run a configuration check:

```bash
docker compose --profile=init --profile=node run --rm check-config
```

### Individual Services

You can start, stop, or restart individual services:

```bash
docker compose --profile=init --profile=node up -d geth
docker compose --profile=init --profile=node stop beacon
docker compose --profile=init --profile=node restart validator
```

### Viewing Logs

View logs for specific services:

```bash
docker compose --profile=init --profile=node logs geth
docker compose --profile=init --profile=node logs beacon
docker compose --profile=init --profile=node logs validator
```

Add `-f` to follow the logs in real-time:

```bash
docker compose --profile=init --profile=node logs -f geth
```

Use grep to filter for specific events:

```bash
docker compose --profile=init --profile=node logs -f geth 2>&1 | grep 'Looking for peers'
docker compose --profile=init --profile=node logs -f beacon 2>&1 | grep 'Synced new block'
docker compose --profile=init --profile=node logs -f validator 2>&1 | grep 'Submitted new'
```

### Environment Variables

Remember that many settings are controlled via environment variables in the `.env` file. You can modify these to adjust your node's configuration.

For more detailed information on Docker Compose commands and options, refer to the [official Docker Compose documentation](https://docs.docker.com/compose/reference/).

## Submitting Deposits

After generating validator keys and before starting your validator, you need to submit deposits for each validator. This process stakes your ETH and registers your validator(s) with the network.

1. Ensure you have the following environment variables set in your `.env` file:
   - `DEPOSIT_RPC_URL`: The RPC URL for the network on which you're submitting deposits
   - `DEPOSIT_CONTRACT_ADDRESS`: The address of the deposit contract. On mainnet this is the
     current contract, `0xB98aafa6684aef18AEf518772F01F5aE0DA5eA4C` — see
     [Deposit contract switch](#deposit-contract-switch-mainnet). It must match
     `DEPOSIT_CONTRACT_ADDRESS` in `networks/mainnet/config.yml`.

2. Run the deposit submission process with the private key of the account funding the deposits:
   ```bash
   DEPOSIT_PRIVATE_KEY="your_private_key_here" docker compose run --rm submit-deposits
   ```
   Replace `your_private_key_here` with the actual private key.

   This command will iterate through all generated validator keys and submit the required deposits.

3. Wait for the transactions to be confirmed on the network before proceeding to start your validator.

For more detailed information on Docker Compose commands and options, refer to the [official Docker Compose documentation](https://docs.docker.com/compose/reference/).

## Using the API Gateway

The validator node exposes its APIs through a Caddy reverse proxy for secure HTTPS access. By default, it uses `localhost` but you can configure a custom domain in your `.env` file. The provided Caddyfile configuration is a basic starting point and may need additional security headers and hardening for production use.

### Domain Setup

If using a custom domain:
1. Point your domain's DNS to your server's IP address
2. Ensure ports 80 and 443 are open on your firewall
3. Set your domain and email (for Let's Encrypt) in the `.env` file

### API Access Control

The API gateway implements the following access controls:

- Public endpoints:
  - Execution layer: All JSON-RPC endpoints (POST /)
  - Consensus layer: Limited set of beacon endpoints including genesis, headers, validator info, and node status
- Private endpoints (localhost and trusted IPs only):
  - All other consensus layer endpoints under /eth/*
  - Configure trusted IPs via RPC_TRUSTED_IP_RANGES in .env

### Local Testing

For local testing, you can access the APIs using curl with the `-k` flag to skip certificate verification:

```bash
# Query beacon node identity (public endpoint)
curl -k -X GET 'https://localhost/eth/v1/node/identity' -H 'accept: application/json'

# Query execution node info (public endpoint)
curl -k -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"admin_nodeInfo","params":[],"id":1}' \
  https://localhost
```

### Installing Local CA Certificate

You can also install Caddy's root CA certificate on your host machine:

Linux:
```bash
docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    /usr/local/share/ca-certificates/root.crt \
  && sudo update-ca-certificates
```

macOS:
```bash
docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    /tmp/root.crt \
  && sudo security add-trusted-cert -d -r trustRoot \
    -k /Library/Keychains/System.keychain /tmp/root.crt
```

Windows:
```bash
docker compose cp \
    caddy:/data/caddy/pki/authorities/local/root.crt \
    %TEMP%/root.crt \
  && certutil -addstore -f "ROOT" %TEMP%/root.crt
```

Note: Many modern browsers maintain their own certificate trust stores. You may need to manually import the root.crt file in your browser's security settings.

### Troubleshooting SSL

If you encounter SSL-related issues:

1. Check Caddy logs:
   ```bash
   docker compose logs caddy
   ```
2. Verify your domain points to your server's IP address
3. Confirm ports 80 and 443 aren't used by other services
4. Check your firewall allows traffic on ports 80 and 443

## Backup and Restore

The setup includes services for backing up and restoring your node data. You can also use [provided snapshots](https://console.cloud.google.com/storage/browser/vana-snapshots;tab=objects?prefix=&forceOnObjectsSortingFiltering=false) for faster sync.

### Using Provided Snapshots

To quickly sync your node using provided snapshots:

1. Download and verify snapshots (`curl` or `wget` can be used but `aria2c` is highly recommended):
   ```bash
   # Download snapshot files (replace DATE with actual date in the format YYYYMMDD, e.g., 20250528)
   aria2c -c -s 16 -x 16 https://storage.googleapis.com/vana-snapshots/DATE/beacon-chaindata-DATE.tar{,.md5}
   aria2c -c -s 16 -x 16 https://storage.googleapis.com/vana-snapshots/DATE/geth-chaindata-DATE.tar{,.md5}

   # Verify checksums
   md5sum -c *.md5
   ```

2. Extract snapshots to the data directory:
   ```bash
   tar -xvf geth-chaindata-DATE.tar
   tar -xvf beacon-chaindata-DATE.tar
   ```

3. Start your node:
   ```bash
   docker compose --profile init --profile node up -d
   ```

> **Note**: make sure you trust the snapshot provider and verify the checksums before restoring!

### Manual Backups

To create and restore manual backups of your node data:

#### Geth (Execution Client) Backup

To perform a backup of your Geth data, ensure that the geth service is stopped, then run:

```bash
docker compose --profile backup run --rm geth-backup
```

This will create a timestamped backup file in the ./backups directory.

#### Beacon Chain Backup

To perform a backup of your Beacon Chain data, ensure that the beacon service is stopped, then run:

```bash
docker compose --profile backup run --rm beacon-backup
```

This creates a timestamped copy of the Beacon Chain database in the ./backups directory.

#### Validator Backup

The validator backup can be triggered while the validator service is running:

```bash
docker compose --profile backup run --rm validator-backup
```

This sends a request to the validator service to create a backup, which will be stored in the ./backups directory.

### Restore

Before performing any restore operations, ensure that the respective services are stopped.

#### Geth (Execution Client) Restore

To restore Geth data:

```bash
docker compose --profile restore run --rm geth-restore
```

You'll be prompted to select a backup file to restore from.

#### Beacon Chain Restore

To restore Beacon Chain data:

```bash
docker compose --profile restore run --rm beacon-restore
```

You'll be prompted to select a backup file to restore from.

#### Validator Restore

To restore Validator data:

```bash
docker compose --profile restore run --rm validator-restore
```

You'll be prompted to select a backup file to restore from.

### Important Notes

- Remember your password and separately backup your keystore(s)!
- Performing backups while services are running risks corrupting the backup, with the exception of the validator backup.
- After restoring data, you may need to resync your node to catch up with the latest state of the network.

### CORS Configuration

The API gateway includes CORS (Cross-Origin Resource Sharing) headers to control which domains can access the API. By default, it allows all origins (`*`) but this can be restricted:

1. Set allowed origins in your `.env` file:
```bash
# Allow specific origins (comma-separated)
CORS_ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# Or allow all origins (default)
CORS_ALLOWED_ORIGINS=*
```

2. When setting specific origins:
   - Credentials will be allowed (`Access-Control-Allow-Credentials: true`)
   - Preflight requests are automatically handled
   - Methods are limited to GET, POST, OPTIONS
   - Only Content-Type header is allowed
   - Preflight responses are cached for 24 hours

> **Security Note**: Using `*` for CORS_ALLOWED_ORIGINS is acceptable for public RPC nodes but not recommended for nodes handling sensitive operations. Always restrict origins in production environments.

# Version Management

This repository follows [Conventional Commits](https://www.conventionalcommits.org/). Each commit message must be structured as:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Types that affect versioning:
- `fix:` for bug fixes (PATCH)
- `feat:` for new features (MINOR)
- Any commit with `!` after type/scope or with `BREAKING CHANGE:` footer (MAJOR)

Other valid types include: `build:`, `chore:`, `ci:`, `docs:`, `perf:`, `refactor:`, `style:`, `test:`

Example:
```
feat(api)!: change default ports for all services

The default ports for geth, beacon, and validator services have been updated
to avoid conflicts with common system services.

BREAKING CHANGE: Users must update their firewall rules and client configurations

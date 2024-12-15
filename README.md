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
   git clone https://github.com/vana-com/vana-private.git
   cd vana
   ```

2. Configure your environment:
   ```bash
   # For Moksha testnet
   cp .env.moksha.example .env
   # OR for Mainnet
   cp .env.mainnet.example .env

   # Edit .env with your preferred text editor
   ```

3. Start your node:
   ```bash
   docker compose --profile init --profile node up -d
   ```

4. Verify your node is running:
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
   DEPOSIT_RPC_URL=<your_rpc_url>
   DEPOSIT_CONTRACT_ADDRESS=<contract_address>
   ```

2. Set up validator keys:

   If you have existing keys:
   - Place keystore files in `./secrets`
   - Create `wallet_password.txt` and `account_password.txt` in `./secrets`
   ```bash
   # Import existing keys
   docker compose --profile manual run --rm validator-import
   ```

   If you need new keys:
   ```bash
   # Generate new keys
   docker compose --profile manual run --rm validator-keygen

   # Import generated keys
   docker compose --profile manual run --rm validator-import
   ```

3. Submit deposits (if not done already):
   ```bash
   # Add deposit private key
   echo "your_private_key" > ./secrets/deposit_private_key.txt

   # Submit deposits
   docker compose --profile manual run --rm submit-deposits
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
docker compose --profile init --profile manual run --rm validator-exit
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
- Various port configurations for different services

Ensure all required variables are set correctly before proceeding.

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
docker compose --profile init --profile manual run --rm validator-keygen
```

Import validator keys:
```bash
docker compose run --profile manual --rm validator-import
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
   - `DEPOSIT_CONTRACT_ADDRESS`: The address of the deposit contract

2. Create a file named `deposit_private_key.txt` in the `./secrets` directory containing the private key of the account funding the deposits:
   ```bash
   echo "your_private_key_here" > ./secrets/deposit_private_key.txt
   ```
   Replace `your_private_key_here` with the actual private key.

3. Run the deposit submission process:
   ```bash
   docker compose --profile manual run --rm submit-deposits
   ```

   This command will iterate through all generated validator keys and submit the required deposits.

4. Wait for the transactions to be confirmed on the network before proceeding to start your validator.

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

1. Download and verify snapshots:
   ```bash
   # Download snapshot files (replace DATE with actual date, e.g., 20241104)
   wget https://storage.googleapis.com/vana-snapshots/DATE/geth-chaindata-DATE.tar.zst{,.md5}
   wget https://storage.googleapis.com/vana-snapshots/DATE/beacon-chaindata-DATE.tar.zst{,.md5}

   # Verify checksums
   md5sum -c *.md5
   ```

2. Extract snapshots to the backups directory:
   ```bash
   zstd -d geth-chaindata-DATE.tar.zst -o "backups/geth_backup_DATE.dat"
   zstd -d beacon-chaindata-DATE.tar.zst -o "backups/beaconchain_DATE.db"
   ```

3. Restore the snapshots:
   ```bash
   # For Geth (requires interactive terminal)
   docker compose run -it --rm geth-restore

   # For Beacon chain (requires interactive terminal)
   docker compose run -it --rm beacon-restore
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

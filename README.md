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

2. Configure your environment:
   ```bash
   cp .env.example .env
   # Edit .env with your preferred text editor
   ```

3. Generate validator keys (interactive process):
   ```bash
   docker compose --profile manual run --rm validator-keygen
   ```

4. Submit deposits for your validator:
   ```bash
   docker compose --profile init --profile manual run --rm submit-deposits
   ```

   Before running this command, ensure you have correctly set the `DEPOSIT_*` environment variables in your `.env` file.

   This process stakes 35k VANA for each set of validator keys in secrets/ and registers them with the network.

5. Initialize and start all services:
   ```bash
   docker compose --profile init --profile node up -d
   ```

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
   docker logs -f geth-node      # Execution layer
   docker logs -f beacon-node    # Consensus layer
   docker logs -f validator-node # Validator client
   ```

3. To follow logs in real-time and filter for specific patterns:
   ```bash
   docker logs -f geth-node 2>&1 | grep 'Looking for peers'
   docker logs -f beacon-node 2>&1 | grep 'Synced new block'
   docker logs -f validator-node 2>&1 | grep 'Submitted new'
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

- `init`: Initialize the environment
- `node`: Run the main node services
- `validator`: Run validator-specific services
- `manual`: For manual operations like key generation
- `clean`: Clean up data

You can combine profiles as needed. For example:

```bash
docker compose --profile init --profile node up -d
```

### Key Management

Generate validator keys (interactive process):
```bash
docker compose --profile manual run --rm validator-keygen
```

Import validator keys:
```bash
docker compose run --rm validator-import
```

### Cleaning Up

To clean all data:
```bash
docker compose --profile clean run --rm clean-all
```

To clean specific components:
```bash
docker compose --profile clean run --rm clean-geth
docker compose --profile clean run --rm clean-prysm
```

### Configuration Check

Run a configuration check:

```bash
docker compose run --rm check-config
```

### Individual Services

You can start, stop, or restart individual services:

```bash
docker compose up -d geth
docker compose stop beacon
docker compose restart validator
```

### Viewing Logs

View logs for specific services:

```bash
docker compose logs geth
docker compose logs beacon
docker compose logs validator
```

Add `-f` to follow the logs in real-time:

```bash
docker compose logs -f geth
```

### Environment Variables

Remember that many settings are controlled via environment variables in the `.env` file. You can modify these to adjust your node's configuration.

For more detailed information on Docker Compose commands and options, refer to the [official Docker Compose documentation](https://docs.docker.com/compose/reference/).

## Submitting Deposits

After generating validator keys and before starting your validator, you need to submit deposits for each validator. This process stakes your ETH and registers your validator(s) with the network.

1. Ensure you have the following environment variables set in your `.env` file:
   - `DEPOSIT_RPC_URL`: The RPC URL for the network on which you're submitting deposits
   - `DEPOSIT_CONTRACT_ADDRESS`: The address of the deposit contract
   - `DEPOSIT_PRIVATE_KEY`: The private key of the account funding the deposits

2. Run the deposit submission process:
   ```bash
   docker compose --profile deposit run --rm submit-deposits
   ```

   This command will iterate through all generated validator keys and submit the required deposits.

3. Wait for the transactions to be confirmed on the network before proceeding to start your validator.

For more detailed information on Docker Compose commands and options, refer to the [official Docker Compose documentation](https://docs.docker.com/compose/reference/).
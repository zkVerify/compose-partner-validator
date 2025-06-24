# Compose zkVerify validator project for partners

This repository contains all the necessary resources for deploying a zkVerify validator node for external partners. 

## Project overview

There is only one supported **node type** available for deployment:  `validator`

All scripts in this repository prompt for selection of the **node type** and the **network** to deploy.

---

## Requirements

* docker
* docker compose
* jq
* gnu-sed for Darwin distribution

---

## Instructions

⚠️ **Please review the `OPTIONAL` steps before manually starting the project after running the `./scripts/init.sh` script.**

Run the [init.sh](./scripts/init.sh) script and follow the instructions to prepare the deployment for the first time.

This script will generate all necessary deployment files under the [deployments](deployments) directory and provide the command to start the project. **However, it will not start the project automatically.**

```shell
./scripts/init.sh
```

### Optional: ZKV Node Data Snapshots

To reduce the time required for a node's startup, **daily snapshots of chain data** are available for:
- Testnet: https://bootstraps.zkverify.io/
- Mainnet: **<PLACEHOLDER_FOR_MAINNET_URL>** <!-- TODO: Replace with mainnet snapshot URL -->

Snapshots are available in two forms:

- **Node snapshot**
- **Archive node snapshot**

Each snapshot is a **.tar.gz** archive containing the **db** directory, intended to replace the **db** directory generated during the initial node run.

### Optional: ZKV Node Secrets Injection

During the initial deployment, if prompted, the script will generate and store **ZKV_NODE_KEY** and **ZKV_SECRET_PHRASE** values in the `.env` file.

Alternatively, these secrets can be injected at runtime using a custom container entrypoint script to avoid keeping them in plaintext on disk.

Use the following steps to implement this approach:

1. Delete values of **ZKV_NODE_KEY** and **ZKV_SECRET_PHRASE** under the `deployment/validator-node/${NETWORK}/.env`
    ```bazaar
    ZKV_NODE_KEY=""
    ZKV_SECRET_PHRASE=""
    ```
2. Create **entrypoint_secrets.sh** file under `deployment/validator-node/${NETWORK}/` directory. For example:
    ```
    #!/usr/bin/env sh
    set -eu
    
    # TODO: Implement logic to inject secrets into the environment
   
    # Run the application entrypoint
    echo "=== 🚀 Starting the application entrypoint now..."
    exec /app/entrypoint.sh "$@"
    ```
3. Modify `deployment/validator-node/${NETWORK}/docker-compose.yml` file to mount and execute **custom entrypoint** script
    ```
    volumes:
      - "node-data:/data:rw"
      - "./entrypoint_secrets.sh:/app/entrypoint_secrets.sh:rw"
    entrypoint: [ "/app/entrypoint_secrets.sh" ]
    ```
4. Start compose project using the command provided in the end of [init.sh](./scripts/init.sh) script execution.

### Update

To update the project to a new version (e.g., when a new release is available):

1. Pull the latest changes from the repository.
2. Run the [update.sh](./scripts/update.sh) script.

⚠️ If the script prompts to update values in the `.env` file, it is **recommended** to accept all changes, unless there is a specific reason not to.

```shell
./scripts/update.sh
```

### Destroy

Run the [destroy.sh](./scripts/destroy.sh) script to destroy the node stack and all the associated resources. The script will prompt for confirmation before removing any resources.

```shell
./scripts/destroy.sh
```

### Start

Run the [start.sh](./scripts/start.sh) script to start the node stack.

```shell
./scripts/start.sh
```

### Stop

Run the [stop.sh](./scripts/stop.sh) script to just stop the node stack.

```shell
./scripts/stop.sh
```

---

## Contributing Guidelines

Please refer to the [CONTRIBUTING.md](CONTRIBUTING.md) file for information on how to contribute to this project.

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

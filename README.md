# Compose zkVerify validator project for partners

This repository contains all the necessary resources for deploying a zkVerify validator node for external partners. 

In order to help launch a node on your validation machine, we provide a set of scripts that automate the configuration of the Docker Compose project comprising a zkVerify node, and a nginx reverse proxy to let the zkVerify team access its metrics interface.

We also provide instructions on the on-chain activities needed to become an active validator in the Proof of Stake consensus of zkVerify, and to claim the rewards for validating afterwards.

## Project overview

The project includes the following scripts ([full description](#management-scripts)):
* *init.sh*: creates a new configuration for the Docker Compose project for a validator deployment
* *start.sh*: starts the Docker Compose deployment
* *stop.sh*: stops the Docker Compose deployment
* *update.sh*: updates the existing configuration (if any) to align with the latest templates included with this repository
* *destroy.sh*: stops any running Docker Compose deployment and removes any existing configuration

All the scripts in this repository prompt for selection of the **node type** and the **network** to deploy.

There is only one supported **node type** available for deployment:  `validator`

The **network** can be chosen to be *testnet* (i.e. [the Volta testnet](https://polkadot.js.org/apps/?rpc=wss://zkverify-volta-rpc.zkverify.io)), or *mainnet* (i.e. [the Mainnet](https://polkadot.js.org/apps/?rpc=wss://zkverify-rpc.zkverify.io)).

---

## Requirements

* docker
* docker compose (v2 or newer)
* jq
* gnu-sed for Darwin distribution

---

## Instructions
In order to become an active validator on zkVerify, and be compensated for such activities, you need to (1) [set up your node](#1-setting-up-the-node), and (2) [state your intent on chain in the Proof of Stake consensus](#2-join-proof-of-stake).

### 1. Setting up the node

⚠️ **Please review the `OPTIONAL` steps before manually starting the project after running the `./scripts/init.sh` script.**

Run the [init.sh](./scripts/init.sh) script and follow the instructions to prepare the deployment for the first time.

This script will generate all necessary deployment files under the [deployments](deployments) directory and provide the command to start the project. **However, it will not start the project automatically.**

```shell
./scripts/init.sh
```

#### Optional: ZKV Node Data Snapshots

To reduce the time required for a node's startup, **daily snapshots of chain data** are available for:
- Mainnet: https://bootstraps.zkverify.io/
- Testnet: https://bootstraps.zkverify.io/volta

Snapshots are available in two forms:

- **Node snapshot**
- **Archive node snapshot**

Each snapshot is a **.tar.gz** archive containing the **db** directory, intended to replace the **db** directory generated during the initial node run.

To use a snapshot:

1. Stop the running node:
   ```shell
   ./scripts/stop.sh
   ```
2. Navigate to the node's data directory. This may require `sudo` permissions. For an RPC node, the path is:
   ```
   cd /var/lib/docker/volumes/zkverify-rpc_node-data/_data/node/chains/zkv_mainnet
   ```
3. Note the owner and permissions of the existing `db` directory, then delete it.
4. Extract the downloaded snapshot and move its `db` directory into the current directory.
5. Ensure the new `db` directory has the same permissions as the original db directory.
6. Return to the project directory and start the node:
   ```shell
   ./scripts/start.sh
   ```
7. Verify the snapshot is working by checking the logs for `Highest known block at`, which should be close to the current chain height.
8. Watch the logs until you can see the block height increasing.

#### Optional: ZKV Node Secrets Injection

During the initial deployment, if prompted, the script will generate and store **ZKV_NODE_KEY** and **ZKV_SECRET_PHRASE** values in the `.env` file.

Alternatively, these secrets can be injected at runtime using a custom container entrypoint script to avoid keeping them in plaintext on disk.

Use the following steps to implement this approach:

1. Delete values of **ZKV_NODE_KEY** and **ZKV_SECRET_PHRASE** under the `deployments/validator-node/${NETWORK}/.env`
    ```bazaar
    ZKV_NODE_KEY=""
    ZKV_SECRET_PHRASE=""
    ```
2. Create **entrypoint_secrets.sh** file under `deployments/validator-node/${NETWORK}/` directory. For example:
    ```
    #!/usr/bin/env sh
    set -eu
    
    # TODO: Implement logic to inject secrets into the environment
    # You need ALL the following keys to perform all the validation activities: "babe", "gran", "para", "audi", "asgn".
   
    # Run the application entrypoint
    echo "=== 🚀 Starting the application entrypoint now..."
    exec /app/entrypoint.sh "$@"
    ```
3. Modify `deployments/validator-node/${NETWORK}/docker-compose.yml` file to mount and execute **custom entrypoint** script
    ```
    volumes:
      - "node-data:/data:rw"
      - "./entrypoint_secrets.sh:/app/entrypoint_secrets.sh:rw"
    entrypoint: ["/app/entrypoint_secrets.sh"]
    ```
4. Start the compose project using the command provided in the end of [init.sh](./scripts/init.sh) script execution.

#### Optional: Public Address

The **ZKV_CONF_PUBLIC_ADDR** variable sets the node's `--public-addr` parameter and is optional. Declining is valid and safe, and is the right choice if you are unsure.

[init.sh](./scripts/init.sh) asks for this during first-time setup. Answer **yes** only if this node runs on a machine with a public IP address or hostname that other nodes can connect to. A wrong address is worse than none.

Choose an address type from the menu:

- ipv4 address
- hostname

Enter the address on its own, without the `/ip4/` or `/dns/` prefix and without the `/tcp/<port>` suffix; the script adds those. The port always comes from **NODE_NET_P2P_PORT**. In the `.env` file the result looks like `ZKV_CONF_PUBLIC_ADDR="/ip4/<your-public-ipv4>/tcp/<NODE_NET_P2P_PORT>"`. Choose `done` to finish, or add another address first.

The script rejects addresses that can never work as a public address, including private ranges, loopback, link-local, multicast, carrier-grade NAT, documentation addresses, and local or reserved hostnames.

If you decline, the script comments the variable out (`#ZKV_CONF_PUBLIC_ADDR=""`) so the node starts without the parameter. Set a value and remove the leading `#` at any time.

⚠️ Other validators may not be able to reach this node, and a future node version will require a public address.

[update.sh](./scripts/update.sh) asks for this variable once, on the first run where it has not yet been set or declined. If you quit at the prompt, the next run asks again.

### 2. Join Proof of Stake

In this section you can learn how to register a new validator on the blockchain. The operations described below must be performed just once, **after the node started in the previous section has synchronized with the existing history of the selected zkVerify chain** (testnet or mainnet).  They consist of the submission of some extrinsics (transactions, in Substrate terminology) resulting in your node being able to author new blocks and consequently earn new tokens through staking mechanism.

#### Create a New Validator Account

First, you need a zkVerify account that you will use to execute all the operations needed for becoming a validator. Notice, this will be the Validator account whose address will show up in the list of validators for zkVerify. This account needs a sufficient balance to be able to bond enough tokens and execute management operations on chain, and for such reason **we highly encourage to use a secure wallet extension (e.g. Subwallet) for storing the secrets of the account**. At the same time, we advice against storing this account's secrets directly in PolkadotJS.

> **_NOTE_** Since you are going to submit extrinsics which change the blockchain state, you need sufficient funds in the account. If necessary, reach out to the zkVerify team and provide your account address to get your token allocation.

#### Collect the Session Keys

Session keys are the keys that will sign all the validation operations behind the scenes. You will never use these keys directly in your wallet to sign transactions, but your node will use them to sign operational messages that are at the base of the security of the network. For this reason, **we strongly advise that these public keys do not belong to the same keypair as the one used for the Validator account above.**

We will require the public keys associated with the secrets injected in the node in the [node setup section](#1-setting-up-the-node) to share them with the network.   You need *five* public keys: Babe, Grandpa, Para Validator, Para Assignment, and Authority Discovery.

If you let the ``init.sh`` script create the secret phrase for you, you will find your public keys at the bottom of the output env file (e.g. deployments/validator-node/mainnet/.env).

For the rest of this section, we will assume the following keys:

```
Babe: 0xc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914
Grandpa: 0x0dbccabf681188116e642c1dbc9332a2bbec7fbef1792196879a3cba6c52464b
ParaValidator: 0xc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914
ParaAssignment: 0xc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914
AuthorityDiscovery: 0xc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914
```

> **_NOTE_** If you do not take the [optional step for key injection above](#optional:-zkv-node-secrets0injection), you must use the very same public key for all the types above but Grandpa. This is expected, given that Grandpa uses a signature scheme that is different from the others.

#### Announce Session Keys Onchain

Once you have these five keys, you need to visit [PolkadotJS](https://polkadot.js.org/apps/?rpc=wss://zkverify-rpc.zkverify.io#/extrinsics) and call the ``setKeys`` extrinsic under ``sessions`` module. Make sure that you sign and submit the extrinsic from your Validator account through your wallet extension.

In order to fill the `keys` field you need to concatenate the `Babe` key, followed by the `Grandpa` key (without the `0x` at the beginning), followed by the `ParaValidator` key (without the `0x` at the beginning), followed by the `ParaAssignment` key (without the `0x` at the beginning), followed by the `AuthorityDiscovery` key (without the `0x` at the beginning). **The order of the keys is fixed and must be preserved**. Considering the keys here above, the value for the `keys` field will be the following:

```bash
0xc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf99140dbccabf681188116e642c1dbc9332a2bbec7fbef1792196879a3cba6c52464bc0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914c0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914c0c07abce7879c09231fcbd07165cfaabc4a634636850578a914b08b87cf9914
```

The `proof` field can be set to ``0x``. Now sign and submit the extrinsic. In few seconds you should receive a green pop-up message on the top-right corner confirming the extrinsic has been succesfully submitted.

![setkeys](doc/polkadotjs_setkeys_s1.png?raw=true "PolkadotJS setKeys")

#### Staking VFY

Next step would be to stake VFY for your registered validator node. 

##### Initial Bonding

Call the ``bond`` extrinsic under the ``staking`` module. You need to fill in two fields, that we will describe in this section: (1) the *value* field with the amount of VFY you would like to stake, and (2) the *payee* field that selects where to receive the staking rewards.

![bond](doc/polkadotjs_staking_bond.png?raw=true "PolkadotJS bond")

###### Value

When determining the amount of tokes to bond, your take would be to stake more than the lowest in the list of validators scheduled to be active in the next era.

In order to get a picture of the current state, and get an estimate of the amount of tokens that you have to stake to be elected as a valdator for the next era, you can check the active and candidate validator sets in the current era in the same PolkadotJS. Navigate to ``Network`` > ``Staking``, and choose the ``Targets`` tab. You will get a complete list of validators, one per row. Each row has a ``total stake`` field, which represents the sum of the own stake of the validator itself, plus all the nominations that the validator received.

Also, as of now you need to stake a minimum amount of 100000 VFY to become a validator.

You can check the `MinimumValidatorBond` anytime by going to `Developer -> Chain State -> staking state query -> minValidatorBond` and clicking the ``+`` button. Notice that you need to remove 18 trailing digits to compute the amount in VFY.

###### Payee

Next, you must choose the account type in payee option; this options determines where the reward for being a validator will be credited. The default will be *Staked*, which means that the reward, when claimed, will automatically increase your bond amount. It will still be possible to unlock such funds at a later time, but it will require manual intervention and will incur an unlocing delay (see the [Removing the Bond section](#optional:-updating-the-bond)).

If instead you wish to receive tokens that you can use without restrictions immediately, we suggest to select either *Stash* or *Account*. *Stash* credit your rewards on the same account that owns the bond, while the *Account* type lets you fill in any account that you own in the field that appears below (as shown in the picture).

We discourage using the deprecated *Controller* option.

##### Optional: Updating the Bond

Once you have some tokens bonded, using again the ``bond`` extrinsic results in the error ``AlreadyBonded``.

In order to bond more tokens from the free balance, you must use the ``bondExtra`` extrinsic from the ``staking`` module. If instead you wish to update the payee for your rewards, you should use the ``updatePayee`` extrinsic.

##### Optional: Removing the Bond

In order to remove a bond, the ``unbond`` extrinsic should be used. After that, once the unlocking period has passed, it is possible to use the ``withdraw_unbonded`` extrinsic to move the unbonded tokens back to the free balance.

#### Start validating

Once we have staked VFY, we are ready to start validating on the zkVerify network. We need to execute an extrinsic called as ``validate(prefs)`` under ``staking`` module which takes an input of how much commission your validator would take from the nominations. (If you are not sure, you can put ``100000000``, corresponding to ``10%``, as the default value). Also set the ``blocked`` field to ``no`` and sign and submit the transaction.

Wait for a green pop-up message confirming successful submission. As an additional double check you can navigate to the section `Network` then to the subsection `Staking`, click on `Waiting` tab and verify that your validator is within the list.

That's it! You just need to wait for the completion of the current era and the next one (since an era lasts for 6 hours, in the worst case this would be 12 hours).  After that, provided that your stake grants you a slot in tha active validator set, your node will start authoring new blocks. You can check this by navigating to the section `Network` then to the subsection `Explorer` for a summarized view of the list of recently authored blocks, or to the section `Network` then to the subsection `Staking` for an advanced console specifically designed for staking.

![bond](doc/polkadotjs_staking_validate.png?raw=true "PolkadotJS validate")

#### Collect the Rewards

In order to claim the new tokens you and your nominators deserve for securing the blockchain, you need to submit a dedicated exintrinsic. Navigate to the section `Developer` then to the subsection `Extrinsics` and select `staking`, `payoutStakers`. Choose your validator account as `validatorStash: AccountId32` and insert target era in the textbox `era: u32 (EraIndex)`.  Finally click on `Submit Transaction` button:

![claim](./doc/polkadotjs_claim.png?raw=true "PolkadotJS Claim")

the era index being retrievable from section `Developer` then to the subsection `Chain state`, state `staking`, `erasRewardPoints`, then filtering with respect to your validator account. Notice that an era lasts for 6 hours, so you will have to claim the rewards for up to 4 eras per day. **Rewards must be claimed within 30 eras (i.e. approximately 1 week) since the end of an era, otherwise the reward for such era is lost.**

You can also check what eras have already been claimed from section `Developer` then to the subsection `Chain state`, state `staking`, `claimedRewards`, then filtering by era number and your validator account. An empty result indicates that no claim was performed for such era for such validator, whereas any result which includes a `0` (plus potentially some other digits in case of more than 64 nominators) indicates that the reward was already claimed for such era and such validator.

![claimed](./doc/polkadotjs_claimedrewards.png?raw=true "PolkadotJS Claimed Rewards")

---

## Management Scripts

### Update

```shell
./scripts/update.sh
```

To update the project to a new version (e.g., when a new release is available):

1. Pull the latest changes from the repository.
2. Run the [update.sh](./scripts/update.sh) script.

⚠️ If the script prompts to update values in the `.env` file, it is **recommended** to accept all changes, unless there is a specific reason not to.

If **ZKV_CONF_PUBLIC_ADDR** has not yet been set or declined, the script asks for it once. See [Optional: Public Address](#optional-public-address).

### Destroy

```shell
./scripts/destroy.sh
```

Run the [destroy.sh](./scripts/destroy.sh) script to destroy the node stack and all the associated resources. The script will prompt for confirmation before removing any resources.

### Start

```shell
./scripts/start.sh
```

Run the [start.sh](./scripts/start.sh) script to start the node stack.

### Stop

```shell
./scripts/stop.sh
```

Run the [stop.sh](./scripts/stop.sh) script to just stop the node stack.

---

## Contributing Guidelines

Please refer to the [CONTRIBUTING.md](CONTRIBUTING.md) file for information on how to contribute to this project.

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

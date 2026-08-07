## 1.0.5
* node: zkVerify version for mainnet pinned to `2.0.0`
* node: zkVerify version for testnet updated to `2.0.0`

## 1.0.4
* node: zkVerify version for testnet pinned to `2.0.0-rc1`
* node: added optional **ZKV_CONF_PUBLIC_ADDR** variable
* automation: `init.sh` and `update.sh` scripts prompt to set **ZKV_CONF_PUBLIC_ADDR**
* automation: `update.sh` script syncs **NODE_VERSION** from the template
* automation: `docker compose` version check accepts v2 and newer
* automation: interactive menus list one option per line

## 1.0.3
* node: added **ZKV_CONF_STATE_PRUNING=4096** state pruning configuration

## 1.0.2
* node: added **ZKV_CONF_BLOCKS_PRUNING=14400** blocks pruning configuration

## 1.0.1
* node: added **ZKV_CONF_NO_PRIVATE_IP** and **ZKV_CONF_NO_MDNS** as failsafe mechanism to prevent network abuse from the node

## 1.0.0
* general: Release to public

## 0.2.1
* general: support for Testnet is back

## 0.2.0
* general: support for Mainnet added + small fixes

## 0.1.1
* general: README.md file typo fix and COMPOSE_PROJECT_NAME value adjusted

## 0.1.0
* general: zkVerify validator node compose project for external partners

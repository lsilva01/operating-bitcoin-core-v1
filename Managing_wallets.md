# Managing Wallets

## 1. Via Command-Line

### 1.1 Creating Wallet

Since version 0.21, Bitcoin Core no longer has a default wallet.
Wallets can be created with the RPC command `createwallet` and more information about this command can be found running `bitcoin-cli help createwallet`.

The following command, for example, creates a descriptor wallet:

>$ bitcoin-cli -named createwallet wallet_name="wallet01" descriptors=true

If the node is running on testnet network, `-testnet` parameter should be added.

>$ bitcoin-cli -testnet -named createwallet wallet_name="wallet01" descriptors=true

The `descriptors` parameter can be omitted if the intention is to create a legacy wallet.

For now, the default is the legacy wallet, but that should change in the near future.

By default, wallets will be create in the `~/.bitcoin/wallets/wallet_name` folder. If running on testnet, it will be created in `~/.bitcoin/testnet3/wallets/wallet_name`.

### 1.2 Encryping Wallet

### 1.3 Listing Wallets

To open or backup a wallet, the user needs to know the its name. The command `listwallets` 

### 1.4 Backing Up Wallet

Wallets can be safely copied for another destination. This backup file should be stored off-line.

The command is `bitcoin-cli backupwallet "destination"`. The destination parameter must include the name of the file. Otherwise, the command will return an error message like "Error: Wallet backup failed!" if the wallet is of descriptor type. If it is of legacy type, it will be copied with the name `wallet.dat`.

>./src/bitcoin-cli -testnet -rpcwallet="legacy-walley-01" backupwallet /home/node01/Backups/test.dat

The `-rpcwallet` is the name of the wallet that will be copied.

### 1.5 Dumping a wallet

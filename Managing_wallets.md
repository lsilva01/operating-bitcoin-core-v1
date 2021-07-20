# Managing Wallets

## 1. Backing Up and Restoring Wallet

### 1.1 Creating Wallet

Since version 0.21, Bitcoin Core no longer has a default wallet.
Wallets can be created with the RPC command `createwallet` and more information about this command can be found running `bitcoin-cli help createwallet`.

The following command, for example, creates a descriptor wallet:

`$ bitcoin-cli -named createwallet wallet_name="wallet01" descriptors=true`

If the node is running on testnet network, `-testnet` parameter should be added. Or `-signet` if signet.

`$ bitcoin-cli -testnet -named createwallet wallet_name="wallet01" descriptors=true`

The `descriptors` parameter can be omitted if the intention is to create a legacy wallet.

For now, the default type is the legacy wallet, but that should change in the near future.

By default, wallets are created in the `~/.bitcoin/wallets/wallet_name` folder. If the node is running on testnet, the wallets are created in `~/.bitcoin/testnet3/wallets/wallet_name` or if signet, ``~/.bitcoin/signet/wallets/wallet_name`.

### 1.2 Encrypting Wallet

The `wallet.dat` file is not encrypted by default and is, therefore, vulnerable if an attacker gains access to the device where the wallet or the backups are stored.

The wallet must be encrypted with the following command:

`$ bitcoin-cli -rpcwallet="wallet-01" encryptwallet "passphrase"`

The `encryptwallet` command is used only when the wallet is not encrypted yet. Otherwise, the `walletpassphrasechange` command should be used.

`$ bitcoin-cli -rpcwallet="wallet-01" walletpassphrasechange "oldpassphrase" "newpassphrase"`

The `-rpcwallet` is the name of the wallet that will be encrypted.

The term "encrypt the wallet" used here is not very accurate. This command only encrypts only the private key. All other wallet information, such as transactions, is still visible.

Note that if the passphrase is lost, all the coins in the wallet will also be lost forever.

### 1.3 Backing Up Wallet

Wallets can be safely copied for another destination. This backup file should be stored offline, such as on a USB drive, another computer, or an external hard drive.

If the wallet and backup are lost for any reason, the bitcoins related to this wallet will become permanently inaccessible.

The command is `bitcoin-cli backupwallet "destination"`. The destination parameter must include the name of the file. Otherwise, the command will return an error message like "Error: Wallet backup failed!" if the wallet is of descriptor type. If it is of legacy type, it will be copied with the default name `wallet.dat`.

`$ bitcoin-cli -rpcwallet="wallet-01" backupwallet /home/node01/Backups/backup01.dat`

### 1.4 Backup Frequency

The Bitcoin Core wallet was originally a collection of unrelated private keys with their associated addresses. If a non-HD wallet generated a key/address, gave that address out and then restored a backup from before that key's generation, then any funds sent to that address would be lost definitively.

However, [version 0.13](https://github.com/bitcoin/bitcoin/blob/master/doc/release-notes/release-notes-0.13.0.md) introduced HD wallets. Restoring old backups can no longer definitively lose funds as long as the addresses used were from the wallet's HD seed (since all private keys can be rederived from the seed).

So, theoretically, a single backup is enough. But it is recommended to make regular backups (1 time a day or a week) or when there is a relevant amount of new transactions in the wallet.

### 1.5 Restoring Wallet From a Backup

An empty wallet must be created to restore a wallet, . And then, the backup file rewrites the `wallet.dat` of this new wallet.

The user must unload the wallet and load it again after replacing the file.

```
$ bitcoin-cli createwallet "restored_wallet"
$ bitcoin-cli unloadwallet "restored_wallet"
$ cp ~/Backups/backup01.dat ~/.bitcoin/wallets/restored_wallet/wallet.dat
$ bitcoin-cli loadwallet "restored_wallet"
```

After these steps, the wallet balance can be checked.

`$ bitcoin-cli -rpcwallet="restored_wallet" getbalance`

### 1.6 `-rescan` and `-reindex`

The `-rescan` argument rescans the blockchain for missing wallet transactions on startup.

`$ bitcoind -rescan`

This is only necessary if there are missing transactions after restoring the wallet.

A pruned node, however, is incompatible with the `-rescan` option, since it does not have the data to check for relevant transactions.

In that case, it is necessary to restart the node with the `-reindex` command-line option to start over with the initial sync. The wallet will scan for relevant transactions during the synchronization and rediscover the funds and transaction history of the wallet.

`$ bitcoind -reindex`

### 1.7 Dumping Wallet

Alternatively, the `dumpwallet` command can be used to backup the wallet. It dumps all wallet keys (including the master private key) in a human-readable format to a file.

Note that this file is not encrypted and the keys are exposed. An attacker in possession of the file could recreate the wallet and gain access to the keys.

The file is generated on the server-side and so the user must have access to server folders.

`$ bitcoin-cli -rpcwallet="wallet-01" dumpwallet /home/node01/Backups/dump01.txt`

### 1.8 Importing Wallet From a Dump File

The command `importwallet` imports keys from a wallet dump file.

The first step is to create a new wallet.

`$ bitcoin-cli createwallet "from_dump_file"`

Then this command can be called.

`$ bitcoin-cli -rpcwallet="from_dump_file" importwallet /home/node01/Backups/dump01.txt`

After that, `getwalletinfo` can be used to check if the wallet has been fully restored.

`$ bitcoin-cli -rpcwallet="from_dump_file" getwalletinfo`

# 1. Cold Wallet using Bitcoin Core

Since Bitcoin 0.7, it is possible to keep your private keys entirely offline without third party software.

In this tutorial, there are two Bitcoin Core hosts, one offline that runs the cold wallet and the other is a regular online node.

This tutorial is, in part, similar to the multisig one. But it emphasizes more the use of an offline machine and the use of an singlesig wallet. For the beginning reader, it is good to know the two processes and and the similarities.

## 1.1 Create the Wallet

First, create the wallet on the offline machine. If the purpose is cold storage, it is important that this machine does not connect to any network after that.

```bash
./src/bitcoin-cli -signet -named createwallet wallet_name="cold_wallet" descriptors=true
```

This tutorial uses a descriptor wallet. Other options, such as encryption, can be added.

## 1.2 Export the descriptors

The `listdescriptors` RPC is used to export the xpubs. By default, the xprivs are not listed.

To create a watch-only wallet in the connected node, only the xpubs are necessary.

The RPC result is write in a file, so it can be copied to another machine.

```bash
./src/bitcoin-cli -signet -rpcwallet="cold_wallet" listdescriptors > ~/desc_cold_wallet.txt
```

## 1.3 Create the Watch-Only Wallet

Copy the descriptor file to on-line machine and store the descriptor in a bash variable.

```bash
watch_only_desc=$(cat ~/Downloads/desc_cold_wallet.txt | jq '.descriptors')

./src/bitcoin-cli -signet -named createwallet wallet_name="watch_only_wallet" disable_private_keys=true blank=true descriptors=true

./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" importdescriptors "$watch_only_desc"

./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" getwalletinfo

./src/bitcoin-cli -signet -named createwallet wallet_name="recipient_wallet" descriptors=true
```

The `watch_only_wallet` was created with no keys, no HD seed and private keys disabled.

This can be confirmed with `getwalletinfo` RPC.

The `recipient_wallet` will only be used to receive coins from `watch_only_wallet`.

## 1.4 Fund Wallet

```bash
receiving_address=$(./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" getnewaddress)

./contrib/signet/getcoins.py -a $receiving_address
```

Generate a new address on the online node and use it to receive coins. From now on, the cold wallet will only be used for signing.

The wallet can receive signet coins generating a new address and passing it as parameters to `getcoins.py` script.

If the script throws an error such as `Captcha required (reload page)`, the url in script can be access directy.
At time of writing, the url is [`https://signetfaucet.com`](https://signetfaucet.com).

Coins received by the wallet can only be spent after 1 confirmation. It is necessary to wait for the time for a new block to be mined to continue.

The `getbalances` RPC may be used to check the balance. Coins with `trusted` status can be spent.

```bash
./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" getbalances
```

## 1.4 Spend Coins

The watch-only wallet knows its UTXOs, since it is connected to blockchain, so it must be used to spend them.

Since the watch-only wallet does not have private keys, a Partially Signed Bitcoin Transaction (PSBT) should be created first.

It is also possible to createad raw transaction instead, but PSBT is more flexible, it can be combined with other transactions and it is also used for multisig.

```bash
balance=$(./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" getbalance)

amount=$(echo "$balance * 0.8" | bc -l | sed -e 's/^\./0./' -e 's/^-\./-0./')

destination_addr=$(./src/bitcoin-cli -signet -rpcwallet="recipient_wallet" getnewaddress)

./src/bitcoin-cli -signet -named -rpcwallet="watch_only_wallet" walletcreatefundedpsbt outputs="{\"$destination_addr\": $amount}" | jq -r '.psbt' > ~/part01.psbt
```

The last command writes the PSBT in a file, so it can be copied to the off-line machine.

Optionally, the PSBT can be decoded to a JSON format using `decodepsbt` RPC or analyzed using `analyzepsbt`, which provides information about the current status of a PSBT and its inputs, eg missing signatures.

```bash
./src/bitcoin-cli -signet decodepsbt $(< ~/part01.psbt)

./src/bitcoin-cli -signet analyzepsbt $(< ~/part01.psbt)
```

## 1.5 Sign the PSBT

The step below must be executed using the cold wallet, in the off-line machine.

```bash
./src/bitcoin-cli -signet -rpcwallet="cold_wallet" walletprocesspsbt "$(< ~/part01.psbt)" | jq -r '.psbt' > ~/signed.psbt
```

Just as before, the signed PSBT is saved in a file so that it can be copied to the online node, which will broadcast the transaction to the connected peers.

## 1.5 Finalize and Broadcast the PSBT

Before execute the commands below, copy th file to online machine.

```bash
finalized_psbt_hex=$(./src/bitcoin-cli -signet finalizepsbt $(< ~/signed.psbt) | jq -r '.hex')

./src/bitcoin-cli -signet sendrawtransaction $finalized_psbt_hex
```

The `finalizepsbt` RPC is used to produce a network serialized transaction which can be broadcast with `sendrawtransaction`.

It checks that all inputs have complete scriptSigs and scriptWitnesses and, if so, encodes them into network serialized transactions.


### 1.6 Alternative Workflow (Raw Transaction)

The recommended way to create transaction today is using PSBT and using the PSBT-based RPCs, as seen in the previous section.

But the user can still uses the raw transaction RPCs. This workflow shows this.

```bash
balance=$(./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" getbalance)

amount=$(echo "$balance * 0.8" | bc -l | sed -e 's/^\./0./' -e 's/^-\./-0./')

destination_addr=$(./src/bitcoin-cli -signet -rpcwallet="recipient_wallet" getnewaddress)

raw_transaction=$(./src/bitcoin-cli -signet createrawtransaction "[]" "[{\"$destination_addr\": $amount}]")

./src/bitcoin-cli -signet -rpcwallet="watch_only_wallet" fundrawtransaction "$raw_transaction" | jq -r '.hex' > ~/funded_tx.txt
```

The first step `createrawtransaction` returns a hex raw partial transaction with just one output and no inputs. This must be done on the online node as it has access to the blockchain and knows the UTXO Set.

The inputs can be added manually passing them instead of `'[]'` in the first argument, but the wallet can do it automatically using `fundrawtransaction`. This will return another unsigned raw transaction, but now one that includes inputs (sufficient to cover the amount to be sent) and change (for whatever is left of the inputs spent after subtracting the amount, plus a reasonable fee based on network estimation, and estimation of how big the transaction will be after signing).

The inputs that was added, as well the transaction structure itself, can be verified with `decoderawtransaction` RPC.

```bash
./src/bitcoin-cli -signet decoderawtransaction $(< ~/funded_tx.txt)
```

Then the `funded_tx.txt` file must be copied to the offline machine, where the cold wallet is installed. The hex transaction can this file signed with `signrawtransactionwithwallet` RPC.

```bash
./src/bitcoin-cli -signet -rpcwallet="cold_wallet" signrawtransactionwithwallet $(< ~/funded_tx.txt)  | jq -r '.hex' > ~/signed_tx.txt
```

Finally, the `signed_tx.txt` can be copied to online node adn the transaction can be broadcasted.

```bash
./src/bitcoin-cli -signet sendrawtransaction $(< ~/signed_tx.txt)
```

### Source:

1. [fundrawtransaction - what is it?](https://bitcoin.stackexchange.com/a/106204)

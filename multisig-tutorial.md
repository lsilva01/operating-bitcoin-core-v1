# Multisign

Currently, it is possible to create a multig wallet using Bitcoin Core only.

Although there is already a brief explanation and a functional test about the multisig implemented in [PR 22067](https://github.com/bitcoin/bitcoin/pull/22067), this tutorial proposes to use the signet (instead of regtest), bringing the reader closer to a real environment and explaining some functions in more detail.

Before starting this tutorial, start the bitcoin node on the signet network.

```bash
./src/bitcoind -signet
```

### 1.1 Basic Multisig Workflow

1. For a 2-of-3 multisig, create 3 descriptor wallets. It is important that they are of the descriptor type in order to retrieve the wallet descriptors. These wallets contain HD seed and private keys, which will be used to sign the PSBTs and derive the xpub.

```bash
for ((n=1;n<=3;n++))
do
 ./src/bitcoin-cli -signet -named createwallet wallet_name="participant_${n}" descriptors=true
done
```

2. Extract the xpub of each wallet. To do this, the `listdescriptors` RPC is used. By default, Bitcoin Core single-sig wallets are created using path `m/44'/1'/0'` for PKH, `m/84'/1'/0'` for WPKH and `m/49'/1'/0'` for P2WPKH-nested-in-P2SH based accounts. Each of them uses the chain 0 for external addresses and chain 1 for internal ones, as shown in the example below.

```
wpkh([1004658e/84'/1'/0']tpubDCBEcmVKbfC9KfdydyLbJ2gfNL88grZu1XcWSW9ytTM6fitvaRmVyr8Ddf7SjZ2ZfMx9RicjYAXhuh3fmLiVLPodPEqnQQURUfrBKiiVZc8/0/*)#g8l47ngv

wpkh([1004658e/84'/1'/0']tpubDCBEcmVKbfC9KfdydyLbJ2gfNL88grZu1XcWSW9ytTM6fitvaRmVyr8Ddf7SjZ2ZfMx9RicjYAXhuh3fmLiVLPodPEqnQQURUfrBKiiVZc8/1/*)#en65rxc5
```

The suffix (after #) is the checksum. Descriptors can optionally be suffixed with a checksum to protect against typos or copy-paste errors.
All RPCs in Bitcoin Core will include the checksum in their output.

```bash
declare -A xpubs

for ((n=1;n<=3;n++))
do
 xpubs["internal_xpub_${n}"]=$(./src/bitcoin-cli -signet -rpcwallet="participant_${n}" listdescriptors | jq '.descriptors | [.[] | select(.desc | startswith("wpkh") and contains("/1/*"))][0] | .desc' | grep -Po '(?<=\().*(?=\))')

 xpubs["external_xpub_${n}"]=$(./src/bitcoin-cli -signet -rpcwallet="participant_${n}" listdescriptors | jq '.descriptors | [.[] | select(.desc | startswith("wpkh") and contains("/0/*") )][0] | .desc' | grep -Po '(?<=\().*(?=\))')
done
```

`jq` is used to extract the xpub from the `wpkh` descriptor.

The following command can be used to verify if the xpub was generated correctly.

```bash
for x in "${!xpubs[@]}"; do printf "[%s]=%s\n" "$x" "${xpubs[$x]}" ; done
```

Note that this step extracts the `m/84'/1'/0'` account and does not conform to [BIP 45](https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki) or [BIP 87](https://github.com/bitcoin/bips/blob/master/bip-0087.mediawiki).
At the time of writing, there is no way to extract a specific path from wallets in Bitcoin Core. For this, an external signer/xpub can be used.

[PR #22341](https://github.com/bitcoin/bitcoin/pull/22341), which is still under development, introduces a new wallet RPC `getxpub`. It takes a BIP32 path as an argument and returns the xpub, along with the master key fingerprint.

3. Define the external and internal multisig descriptors, add the checksum and then, join both in a JSON array.

```bash
external_desc="wsh(sortedmulti(2,${xpubs["external_xpub_1"]},${xpubs["external_xpub_2"]},${xpubs["external_xpub_3"]}))"
internal_desc="wsh(sortedmulti(2,${xpubs["internal_xpub_1"]},${xpubs["internal_xpub_2"]},${xpubs["internal_xpub_3"]}))"

external_desc_sum=$(./src/bitcoin-cli -signet getdescriptorinfo $external_desc | jq '.descriptor')
internal_desc_sum=$(./src/bitcoin-cli -signet getdescriptorinfo $internal_desc | jq '.descriptor')

multisig_ext_desc="{\"desc\": $external_desc_sum, \"active\": true, \"internal\": false, \"timestamp\": \"now\"}"
multisig_int_desc="{\"desc\": $internal_desc_sum, \"active\": true, \"internal\": true, \"timestamp\": \"now\"}"

multisig_desc="[$multisig_ext_desc, $multisig_int_desc]"
```

`external_desc` and `internal_desc` specify the output type (`wsh`, in this case) and the xpubs involved. They also use BIP 67 (`sortedmulti`), so the wallet can be recreated without worrying about the order of xpubs. Conceptually, descriptors describe a list of scriptPubKey (along with information for spending from it) [[source](https://github.com/bitcoin/bitcoin/issues/21199#issuecomment-780772418)].

Note that at least two descriptors are usually used, one for internal derivation paths and external ones. There are discussions about eliminating this redundancy, as can been seen in the issue [#17190](https://github.com/bitcoin/bitcoin/issues/17190).

After creating the descriptors, it is necessary to add the checksum, which is required by the `importdescriptors` RPC.

The checksum for a descriptor without one can be computed using the `getdescriptorinfo` RPC. The response has the `descriptor` field, which is the descriptor with checksum added.

There are other fields that can be added to th descriptors:

* `active`: Set the descriptor to be the active one for the corresponding output type (`wsh`, in this case).
* `internal`: Whether matching outputs should be treated as not incoming payments (e.g. change).
* `timestamp`: Time from which to start rescanning the blockchain for the descriptor, in UNIX epoch time.

Documentation for these and other parameters can be found by typing `./src/bitcoin-cli help importdescriptors`.

`multisig_desc` concatenates external and internal descriptors in a JSON array and will be used to create the multisig wallet.

4. Create the multisig wallet

```bash
./src/bitcoin-cli -signet -named createwallet wallet_name="multisig_wallet_01" disable_private_keys=true blank=true descriptors=true

./src/bitcoin-cli  -signet -rpcwallet="multisig_wallet_01" importdescriptors "$multisig_desc"

./src/bitcoin-cli  -signet -rpcwallet="multisig_wallet_01" getwalletinfo
```

To create the multisig wallet, first create an empty one (no keys, HD seed and private keys disabled).

Then import the descriptors created in the previous step using the `importdescriptors` RPC.

After that, `getwalletinfo` can be used to check if the wallet was created successfully.

5. Fund the wallet

```bash
receiving_address=$(./src/bitcoin-cli -signet -rpcwallet="multisig_wallet_01" getnewaddress)

./contrib/signet/getcoins.py -a $receiving_address
```

The wallet can receive signet coins generating a new address and passing it as parameters to `getcoins.py` script.

If the script throws an error such as `Captcha required (reload page)`, the url in script can be access directy.
At time of writing, the url is [`https://signetfaucet.com`](https://signetfaucet.com).

Coins received by the wallet can only be spent after 1 confirmation. It is necessary to wait for the time for a new block to be mined to continue.

6. Create a PSBT

```bash
balance=$(./src/bitcoin-cli -signet -rpcwallet="multisig_wallet_01" getbalance)

amount=$(echo "$balance * 0.8" | bc -l | sed -e 's/^\./0./' -e 's/^-\./-0./')

destination_addr=$(./src/bitcoin-cli -signet -rpcwallet="participant_1" getnewaddress)

funded_psbt=$(./src/bitcoin-cli -signet -named -rpcwallet="multisig_wallet_01" walletcreatefundedpsbt outputs="{\"$destination_addr\": $amount}" options='{"feeRate": 0.00010}' | jq -r '.psbt')
```

Multisig wallets cannot create and sign transactions directly, like it happens in the singlesig ones because it requires the signatures of the co-signers.

Instead a Partially Signed Bitcoin Transaction (PSBT) is created. PSBTs are a data format that allows wallets and other tools to exchange information about a Bitcoin transaction and the signatures necessary to complete it. [[source](https://bitcoinops.org/en/topics/psbt/)

Te current PSBT version (v0) is defined in [BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki).

For simplicity, the destination address is taken from the `participant_1` wallet in the code above, but it can be any valid bitcoin address.

The `walletcreatefundedpsbt` RPC is used to create and fund a transaction in the PSBT format. It is the first step in creating the PSBT.

The `send` RPC can also return a PSBT if more signatures are needed to sign the transaction.

7. Decode or Analyze the PSBT

```bash
./src/bitcoin-cli -signet decodepsbt $funded_psbt

./src/bitcoin-cli -signet analyzepsbt $funded_psbt
```

Optionally, the PSBT can be decoded to a JSON format using `decodepsbt` RPC.

The `analyzepsbt` RPC analyzes and provides information about the current status of a PSBT and its inputs, eg missing signatures.

8. Update the PSBT

```bash
psbt_1=$(./src/bitcoin-cli -signet -rpcwallet="participant_1" walletprocesspsbt $funded_psbt | jq '.psbt')

psbt_2=$(./src/bitcoin-cli -signet -rpcwallet="participant_2" walletprocesspsbt $funded_psbt | jq '.psbt')
```

In the code above, two PSBTs are created. One signed by `participant_1` wallet and other, by the `participant_2` wallet.

The `walletprocesspsbt` is used by the wallet to sign a PSBT.

9. Combine the PSBT

```bash
combined_psbt=$(./src/bitcoin-cli -signet combinepsbt "[$psbt_1, $psbt_2]")
```

The PSBT, if signed separately by the co-signers, must be combined into one transaction before being finalized. This is done by `combinepsbt` RPC.

There is an RPC called `joinpsbts`, but it has a different purpose than `combinepsbt`. `joinpsbts` joins the inputs from multiple distinct PSBTs into one PSBT.

In the example above, PSBTs are the same, but signed by different participants. If the user tries to merge them, the error `Input txid:pos exists in multiple PSBTs` is returned. To be able to merge PSBTs into one, they must have different inputs and outputs.

10. Finalize and Broadcast the PSBT

```bash
finalized_psbt_hex=$(./src/bitcoin-cli -signet finalizepsbt $combined_psbt | jq -r '.hex')

./src/bitcoin-cli -signet sendrawtransaction $finalized_psbt_hex
```

The `finalizepsbt` RPC is used to produce a network serialized transaction which can be broadcast with `sendrawtransaction`.

It checks that all inputs have complete scriptSigs and scriptWitnesses and, if so, encodes them into network serialized transactions.

11. Alternative Workflow (PSBT sequential signatures)

```bash
psbt_1=$(./src/bitcoin-cli -signet -rpcwallet="participant_1" walletprocesspsbt $funded_psbt | jq -r '.psbt')

psbt_2=$(./src/bitcoin-cli -signet -rpcwallet="participant_2" walletprocesspsbt $psbt_1 | jq -r '.psbt')

finalized_psbt_hex=$(./src/bitcoin-cli -signet finalizepsbt $psbt_2 | jq -r '.hex')

./src/bitcoin-cli -signet sendrawtransaction $finalized_psbt_hex
```

Instead of each wallet signing the original PSBT and combining them later, the wallets can also sign the PSBTs sequentially.

After that, the rest of the process is the same: the PSBT is finalized and transmitted to the network.

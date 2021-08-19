# Multisign

Currently, it is possible to create a multig wallet using Bitcoin Core Core only.



### 1.1 Basic Workflow

1. For a 2-of-3 multisig, create 3 descriptor wallets. It is important to be descriptor type in order to retrieve the wallet descriptors. These wallets contain HD seed and private keys, which will be used to sign the PSBTs and derive the xpub.

```bash
for ((n=1;n<=3;n++))
do
 ./src/bitcoin-cli -signet -named createwallet "participant_${n}" descriptors=true
done
```

2. Extract the xpub of each wallet. To do it, the `listdescriptors` is used. By default, Bitcoin Core single-sig wallets are created using path `m/44'/1'/0'` for PKH, `m/84'/1'/0'` for WPKH and `m/49'/1'/0'` for P2WPKH-nested-in-P2SH based accounts. Each of them uses the chain 0 for external addresses and chain 1 for internal ones, as shown in the example below.

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

Note that this step extract the `m/84'/1'/0'` account and does not conform to BIP 45 or BIP 87.
At time of writing, there is no way to extract a specific path from wallets in Bitcoin Core. For this, an external signer/xpub can be used.

3. Define the external and internal multisig, add the checksum and then, join both in a JSON array.

```
external_desc="wsh(sortedmulti(2,${xpubs["external_xpub_1"]},${xpubs["external_xpub_2"]},${xpubs["external_xpub_3"]}))"
internal_desc="wsh(sortedmulti(2,${xpubs["internal_xpub_1"]},${xpubs["internal_xpub_2"]},${xpubs["internal_xpub_3"]}))"

external_desc_sum=$(./src/bitcoin-cli -signet getdescriptorinfo $external_desc | jq '.descriptor')
internal_desc_sum=$(./src/bitcoin-cli -signet getdescriptorinfo $internal_desc | jq '.descriptor')

multisig_ext_desc="{\"desc\": $external_desc_sum, \"active\": true, \"internal\": false, \"timestamp\": \"now\", \"watchonly\": true}"
multisig_int_desc="{\"desc\": $internal_desc_sum, \"active\": true, \"internal\": true, \"timestamp\": \"now\", \"watchonly\": true}"

multisig_desc="[$multisig_ext_desc, $multisig_int_desc]"
```

`external_desc` and `internal_desc` specifies the output type (`wsh`) and the xpubs involveded. It also uses BIP 67 (`sortedmulti`), so the wallet can be recreated without worrying about the order of xpubs.

/<!-- external_desc_sum and multisig_ext_desc -->

`multisig_desc` is the descriptor that will be used to create the multisig wallet.






### Notes:
Conceptually, descriptors describe a list of scriptPubKey (along with information for spending from it). (https://github.com/bitcoin/bitcoin/issues/21199#issuecomment-780772418)

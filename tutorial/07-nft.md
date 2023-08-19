# **`cardano-cli` Exercise 07: NFT**
In this exercise we'll **[upload an image using IPFS](#upload-image-to-ipfs)**, then **[mint a non-fungible token (NFT)](#mint-nft)** with metadata including a link to the hosted image.

We'll mint a duplicate copy of the NFT for extra practice, then **[burn the duplicate NFT](#burn-the-duplicate-nft)**.

***

## **Upload image to IPFS**
Go to **[https://app.pinata.cloud/register](https://app.pinata.cloud/register)** to register an account with Pinata.

Once registered and in the dashboard, find the `+ Add Files` button on the right and click `File`.

Upload a file of your choice to use for your NFT (don't upload copyrighted images).

When the upload completes, click on the value in the `Content Identifier (CID)` column to copy the CID for your image.

You can save this to a temporary `CID` variable for when we need it later:

```sh
CID=QmZjXJ58paaQWNY9SCqJx2UTHMTfuyAECLcbbhBTcZP3Li
```

***

## **Mint NFT**
We'll complete the following steps to mint our NFT:

1. **[Create minting policy](#create-minting-policy)**
2. **[Prepare token representation](#prepare-token-representation)**
3. **[Prepare NFT metadata](#prepare-nft-metadata)**
4. **[Build, sign and submit the minting transaction](#build-sign-and-submit-the-minting-transaction)**

***

### **Create minting policy**
First we'll generate a new key-pair and address for our minting policy:

```sh
cardano-cli address key-gen \
--verification-key-file $KEYS_PATH/nft-policy.vkey \
--signing-key-file $KEYS_PATH/nft-policy.skey

cardano-cli address build \
--payment-verification-key-file $KEYS_PATH/nft-policy.vkey \
--out-file $ADDR_PATH/nft-policy.addr
```

Alternatively, you can use the `key-gen` script to generate a new key-pair/address for our minting policy:

```sh
key-gen nft-policy
```

Now create a new file in `assets/scripts/native` called `nft.script`, and paste the following `json` code:

```json
{
  "type": "all",
  "scripts":
  [
    {
      "type": "before",
      "slot": <INSERT_SLOT_HERE>
    },
    {
      "type": "sig",
      "keyHash": "<INSERT_KEYHASH_HERE>"
    }
  ]
}
```

#### **Add the policy `keyHash`**
Now use the `key-hash` script to get the key-hash for `nft-policy`:

```sh
key-hash nft-policy
```

Copy the key-hash and replace `<INSERT_KEYHASH_HERE>` in `nft.script`.

>**Note:** the `keyHash` is defined as a string and needs to be enclosed in double quotation marks.

#### **Add the minting deadline**
This time, our policy script will include a deadline after which new tokens cannot be minted.

We start by getting the current slot using the `cardano-cli query tip` command (or the `tip` script):

```json
{
    "block": 970436,
    "epoch": 256,
    "era": "Babbage",
    "hash": "13f7616949d7675bd55e408a370a1d7cae7d8e1b0f08bda37b620dc748c98547",
    "slot": 22121952,
    "slotInEpoch": 3552,
    "slotsToEpochEnd": 82848,
    "syncProgress": "100.00"
}
```

Copy the slot number that appears after `"slot"`.  We'll set our deadline to 60 minutes (3600 slots) from the current tip. You can calculate this by hand or use the `expr` command. i.e.:

```sh
DEADLINE=$(expr 22121952 + 3600)
echo $DEADLINE
22125552
```

>**Note:** you'll need to mint two copies of your NFT and burn the second copy before the deadline passes. If you think you'll need more time to complete the exercise, you can choose a slot amount greater than 3600.

Cardano CLI Guru also provides the `calc-slot` script, which automatically queries the current tip and adds the provided number of slots to it:

```sh
DEADLINE=$(calc-slot 3600)
```

Copy the slot number and replace `<INSERT_SLOT_HERE>` in `nft.script`.

>**Note:** the `slot` is defined as an integer and therefore shouldn't be enclosed in quotation marks.

***

### **Prepare token representation**
As in **[Exercise 06: Native Tokens](06-native-tokens.md#prepare-token-representation)**, we need to prepare a representation of our native asset, which requires us to compute the hash of the script to get the policy ID, then join it with a hex-encoded asset name.

>**Note:** replace "JAMB1" below with your desired name for the token.

```sh
PID=$(cardano-cli transaction policyid \
--script-file $NATIVE_SCRIPTS_PATH/nft.script)
```

```sh
TNAME=$(echo -n JAMB1 | xxd -ps | tr -d \\n)
```

```sh
T=$PID.$TNAME
```

Alternatively, you can use the `token` script:

```sh
T=$(token -n nft JAMB1)
```

***

### **Prepare NFT metadata**
We'll now prepare the metadata for the NFT we'll be minting.

See **[CIP 25: Media NFT Metadata Standard](https://cips.cardano.org/cips/cip25/)** for more information about the schema used for NFT metadata.

In `assets/tx`, create a new file `nft-metadata.json` and paste the following `json` code, replacing the content in the `name`, `description`, and `image` fields to personalize your own NFT:

```json
{
        "721": {
            "<INSERT_POLICYID_HERE>": {
              "<INSERT_ASSET_NAME_HERE>": {
                "name": "Jambhala NFT",
                "description": "Om Dzambhala Dzalentraye Svaha",
                "id": 1,
                "image": "ipfs://QmZjXJ58paaQWNY9SCqJx2UTHMTfuyAECLcbbhBTcZP3Li"
              }
            }
        }
}
```

To use the image you uploaded earlier with Pinata, `echo $CID` and replace the CID appearing after `"ipfs://` in the value of the `"image"` property.

Replace `<INSERT_POLICYID_HERE>` with the Policy ID you saved to `$PID`:

```sh
echo $PID
```

If you used the `token` script without saving the policy ID to `$PID`, you can echo the value of `$T` and copy/paste the first portion (before the period separator):

```sh
echo $T
5fe611b2cd8717beff4f0a1d3b09c63cf9927a6f10223b062e32e95a.4a414d42310a
```

>**Note:** This property's value must be the same asset name as the one used later in the minting transaction.

Then replace `<INSERT_ASSET_NAME_HERE>` with the hex-encoded token name you saved to `$TNAME`:

```sh
echo $TNAME
```

If you used the `token` script without saving the token name to `$TNAME`, you can `echo $T` and copy/paste the second portion (after the period separator).

>**Note:** the policy ID and asset name are defined as a string, so must be enclosed in double quotation marks in the metadata JSON.

***

### **Build, sign and submit the minting transaction**
As usual, select a suitable UTXO input from Alice's address (`utxos alice`) and save it to the variable `U`. The input should contain at least a few ADA.

We'll also need to calculate the minimum amount of ADA required for `alice` to mint a single unit of our asset `$T`. Assign the output of this command to a temporary variable `MIN_U`:

```sh
cardano-cli transaction calculate-min-required-utxo \
--protocol-params-file $PARAMS_PATH \
--tx-out $(cat "$ADDR_PATH/alice.addr")+"1 $T"
```

```sh
MIN_U=$(min-utxo alice 1 $T)
```

Similar to the process we followed to mint fungible tokens, we need:
* Our `--tx-out` to also include the minimum UTXO in lovelace (`$MIN_U`), in addition to the quantity of the minted asset (here just one unit) and its token representation (`$T`).
* A `--mint` option with the quantity of the asset being minted and its token representation.
* The `--mint-script-file` option with the filepath to the JSON script file.
* The `--witness-override` option set to 2 to correctly calculate the transaction fee for two witnesses (`nft-policy` and `alice`).

This time we'll also need to include the `--metadata-json-file` option with the filepath to the metadata file.

Our `transaction build` command will look something like this:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-out "$(addr alice)+$MIN_U+1 $T" \
--change-address $(addr alice) \
--mint "1 $T" \
--mint-script-file $NATIVE_SCRIPTS_PATH/nft.script \
--metadata-json-file "$TX_PATH/nft-metadata.json" \
--invalid-hereafter $DEADLINE \
--witness-override 2 \
--out-file $TX_PATH/nft-mint.raw
```

#### **Sign and submit the minting transaction**

```sh
cardano-cli transaction sign \
--tx-body-file $TX_PATH/nft-mint.raw \
--signing-key-file $KEYS_PATH/nft-policy.skey \
--signing-key-file $KEYS_PATH/alice.skey \
--out-file $TX_PATH/nft-mint.signed
```

Or, using the `tx-sign` script:

```sh
tx-sign nft-mint nft-policy alice
```

Now submit the minting transaction:

```sh
cardano-cli transaction submit \
--tx-file $TX_PATH/nft-mint.signed
```

Or, using the `tx-submit` script:

```sh
tx-submit nft-mint
```

After a few minutes, you can query the UTXOs of the recipient to confirm the receipt of the minted tokens.

Now that you've successfully minted an NFT, try minting a duplicate copy of it. Note that as long as the transaction is witnessed by the policy key and the deadline hasn't passed, there is nothing preventing us from creating multiple copies of the same asset. Native scripts aren't able to enforce that only a single copy is minted - at best we can approximate this by imposing sufficiently stringent conditions in our minting policy (witnesses and deadlines). Beyond this we'd require a Plutus smart contract to enforce a single-copy mint.

***

## **Burn the duplicate NFT**
The process of burning the duplicate NFT will be similar to what we did to mint it, with a few key differences:
* Since our input UTXO containing the NFT already has the bare minimum amount of ADA, we'll need a little bit more to cover the transaction fee. We have to supply an additional input UTXO using a second `--tx-in` option.
* We don't need to specify any `--tx-out` option: we'll specify no outputs and have the `--change-address` option send the ADA that remains back to `alice`.
* We make the `--mint` option quantity negative, to burn instead of mint.
* We don't need to supply any metadata with the `--metadata-json-file` option.

Our burn transaction will be built like this (where `$U1` contains the UTXO with the duplicate NFT, and `$U2` contains some other UTXO to cover the fee):

```sh
cardano-cli transaction build \
--tx-in $U1 \
--tx-in $U2 \
--change-address $(addr alice) \
--mint "-1 $T" \
--mint-script-file $NATIVE_SCRIPTS_PATH/nft.script \
--invalid-hereafter $DEADLINE \
--witness-override 2 \
--out-file $TX_PATH/nft-burn.raw
```

Since a token burn is equivalent to a negative mint, the same conditions required to mint the token in the minting policy must also be met to burn tokens. This means the transaction must be submitted before the slot assigned to `$DEADLINE`.
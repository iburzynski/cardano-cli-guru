# **`cardano-cli` Exercise 08: Plutus Gifts**
In this exercise we'll interact with our first Plutus smart contract using `cardano-cli`. Executing a Cardano smart contract requires (at least) two transactions: 
1. A UTxO can be "locked" at a script address (similar to sending funds to a PubKey address), which requires no validation (just like the recipient of a funds transfer doesn't need to provide a signature to authorize the receipt).
2. A UTxO "locked" at the script address can be "unlocked" by providing appropriate inputs to its associated *validator* function to return a `True` value (this successful validation fulfills the same role as a signature in a PubKey address transaction, authorizing a transfer of funds).

We'll practice locking and unlocking UTxOs from the "always succeeds" script - a Plutus contract containing no validation logic, permitting UTxOs locked at the script address to be unlocked via arbitrary transactions.

We'll perform this pair of locking/unlocking transactions using several methods, to highlight some of the features introduced by the Vasil hard fork, namely **[inline datums](https://cips.cardano.org/cips/cip32/)** and **[reference scripts](https://cips.cardano.org/cips/cip33/)**.

## **Script File**
We'll need a script file containing the serialised "always succeeds" contract. The tutorial assumes the presence of this file at `cardano-cli-guru/assets/scripts/plutus/gift.plutus`.

>If you don't have a copy of the script, you can create a `gift.plutus` file in this directory and paste the following:

```json
{
    "type": "PlutusScriptV2",
    "description": "",
    "cborHex": "49480100002221200101"
}
```

## **Script Inputs: Datum & Redeemer**
Although this contract contains no validation logic, remember that every Plutus script is a *function*, which requires input arguments to execute:
* All transactions that **lock** a UTxO at a Plutus script address requires a **datum** (a value with arbitrary contents that can be used to represent a snapshot of an application's "state").
* All transactions that **unlock** a UTxO require the corresponding datum as well as a **redeemer** (another value with arbitrary contents that functions like a key or password).

The "always succeeds" contract ignores both its datum and redeemer inputs, so we can just provide the "Unit" value (an empty placeholder value, represented as `()` in Haskell). We'll need a serialised JSON representation of this value to attach to our transactions in `cardano-cli`.

This tutorial assumes the presence of a file at `cardano-cli-guru/assets/data/unit.json`. You can create this file and paste the following schematized JSON:

```json
{"constructor":0,"fields":[]}
```

## **Script Address**
We'll need to get the script address in order to send funds to the script. As with native scripts, this is created by hashing the script's contents and encoding the result in Bech 32 format via the `address build` subcommand:

```sh
cardano-cli address build \
--payment-script-file $PLUTUS_SCRIPTS_PATH/gift.plutus \
--out-file $ADDR_PATH/gift.addr
```

You can also use Cardano CLI Guru's provided `script-addr` script with the `-p` (Plutus) flag:
```sh
script-addr -p gift
```

## **Lock UTxO (with Datum Hash)**
Prior to the Vasil hard fork and the introduction of **inline datums**, transactions locking UTxOs at a Plutus script address weren't able to include the contents of a datum. Instead, the hash of the datum was included. 

In all unlocking transactions, the corresponding datum contained in the locked UTxO must be provided in some form as a transaction input in order to unlock it: we can think of this as a requirement that the transactions present compatible application "states". When a datum hash is included in the locked UTxO, the unlocking transaction must provide the actual contents of the datum to be checked against the hash.

By attaching a datum hash instead of the datum contents, the amount of required on-chain storage can be reduced (particularly for large datums), which reduces transaction fees. However, this method poses an inconvenience to application developers and users. Per **[CIP-32](https://cips.cardano.org/cips/cip32/)**, which introduced inline datums:

>Datums tend to represent the result of computation done by the party who creates the output, and as such there is almost no chance that the spending party will know the datum without communicating with the creating party.

When datum hashes are included in UTxOs, the contents of the datum must be communicated to the unlocking user, either off-chain or via some indirect means on-chain (which introduces its own inconveniences). Since we'll be performing both transactions ourselves and the datum used will be known, this doesn't pose any issue for us. Additionally, our contract accepts Unit values, which are easy to construct. 

However, if we check the UTxOs at the "always succeeds" script address, we'll see many locked UTxOs there containing datum hashes that don't correspond to the Unit value (which has a hash of `923918e403bf43c34b4ef6b48eb2ee04babed17320d8d1b9ff9ad086e86f44ec`). Since the contents of these datums are unknown to us, we can't include them in our transactions and thus we're unable to unlock them.

For our first transaction pair, we'll lock a UTxO at the contract with a **datum hash** and then unlock it by providing the **datum contents**.

Query the UTxOs at one of your test addresses (`alice`, `bob`, `charlie`, etc.), and select one to use as input for the gift transaction. Copy its `TxHash` and set a temporary variable `U` with the hash followed by a `#` as separator and the `TxIx` value.

We now have everything we need to contruct the locking transaction with the `transaction build` subcommand:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-out $(addr gift)+3000000 \
--tx-out-datum-hash-file $DATA_PATH/unit.json \
--change-address $(addr alice) \
--out-file $TX_PATH/gift-lock-h.raw
```

>**Note:** the transaction above makes a gift of 3 ADA and returns the change to `alice`. You may change the `3000000` amount in `--tx-out` to a lovelace quantity of your choosing.

We use the `--tx-out-datum-hash-file` option to attach the JSON representation of our datum value, which is hashed by `cardano-cli` during transaction construction and included in the locked UTxO.

Now we sign and submit the transaction:

```sh
cardano-cli transaction sign \
--tx-body-file $TX_PATH/gift-lock-h.raw \
--signing-key-file $KEYS_PATH/alice.skey \
--out-file $TX_PATH/gift-lock-h.signed"
```

```sh
cardano-cli transaction submit \
--tx-file "$TX_PATH/gift-lock-h.signed"
```

>Alternatively, using Cardano CLI Guru scripts: `tx-sign gift-lock-h alice` & `tx-submit gift-lock-h`.

## **Unlock UTxO (with Datum)**
To unlock the UTxO we just locked, we'll construct a second transaction using a second user. For this transaction we need to include three additional inputs:
1. The script file (`--tx-in-script-file`), in order to execute the script.
2. A file containing the contents of the datum matching the hash of the locked UTxO (`--tx-in-datum-file`).
3. A file containing the contents of the redeemer we're providing to the validator (`--tx-in-redeemer-file`).
4. An additional UTxO to be used as **collateral** (`--tx-in-collateral`). 

Our `transaction build` will eventually look like this:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-in-script-file $PLUTUS_SCRIPTS_PATH/gift.plutus \
--tx-in-datum-file $DATA_PATH/unit.json \
--tx-in-redeemer-file $DATA_PATH/unit.json \
--tx-in-collateral $C \
--change-address $(addr bob) \
--out-file $TX_PATH/gift-unlock-h.raw
```

But before constructing our transaction, we need to understand more about Cardano's collateral mechanism.

### **Collateral**
Native scripts (also known as *phase-1 scripts*) are restricted in complexity by design. The JSON-like nature of the native scripting language isn't Turing-complete, which means its simple logic can be entirely captured by the ledger rules. Execution costs for native scripts are minimal, so a node executing a script that results in transaction failure will perform only a small amount of uncompensated labor (uncompensated since fees are only charged when a transaction succeeds).

Like native scripts, if a Plutus script is executed by a node and results in transaction failure, the node is unable to be compensated for its computation via the transaction fee. However, unlike Cardano's native script language, Plutus is a Turing-complete language (a modified subset of Haskell), and Plutus scripts can be arbitrarily complex, requiring significantly more computation. To limit the amount of uncompensated validation work performed by nodes, Plutus script transactions follow a two-phase validation scheme*:

1. In **Phase 1**, preliminary checks are made to determine whether the transaction is constructed correctly and can pay its processing fee (if so, the transaction is referred to as **phase-1 valid**).
2. In **Phase 2**, a node on the network actually runs the script(s) included in the transaction. This means if a transaction is **phase-2 invalid**, by the time the transaction fails the network will have incurred costs to initiate and validate the transaction.

>\* This is why another name for Plutus contracts is *phase-2 scripts*.

**Collateral** is an economic assurance required from the (unlocking) to ensure that nodes are adequately compensated should the script result in transaction failure while executing on-chain. If the script is phase-2 valid, the collateral is returned to its owner. If the script fails during (phase-2) execution, the collateral is consumed as a fee.

Since script evaluation is deterministic, whether a transaction will pass phase-2 validation can be determined locally (via wallet or `cardano-cli`) prior to submission. Despite the Turing-completeness of Plutus, every Plutus script is guaranteed to terminate due to the transaction's execution budget, which results in immediate script failure if the cost of computation exceeds the budgeted amount.

If local validation succeeds for a transaction, the user can be sure it will also succeed on-chain, and they will definitely not lose their collateral. Thus, a user acting in good faith should never lose collateral.
>**Note:** when locking UTxOs at a Plutus script address, the script isn't run (all incoming transactions are unvalidated), so no collateral UTxO is required.

*Further reading: [https://iohk.io/en/blog/posts/2021/09/07/no-surprises-transaction-validation-part-2/](https://iohk.io/en/blog/posts/2021/09/07/no-surprises-transaction-validation-part-2/)*

#### **Collateral Amounts**
While the collateral amount can be small, it's sufficient to make a denial of service (DOS) attack on the network prohibitively expensive.

The amount of collateral *required* is set by the `collateralPercentage` protocol parameter (if you've run Cardano CLI Guru's `params` script before, you can find the protocol parameters at `cardano-cli-guru/assets/params.json`). The `collateralPercentage` is a percentage multiplier applied to the amount of the transaction fee to determine the minimum collateral.

The amount of collateral *provided* by the user isn't specified directly: one or more UTxO inputs are added to the transaction (using the `--tx-in-collateral` option), and the provided collateral amount is the sum of these specially designated UTxOs. It is permitted to use the same UTxO as both a regular input (i.e., if some additional input is required from the user) and collateral, since the transaction will only consume either the regular inputs (on success) or the collateral inputs (on failure).

**Before Vasil Hard Fork**
Before the Babbage Ledger Era introduced by the Vasil Hard Fork:
* Collateral could only be consumed in its entirety, even if the normal fee for executing the script was less than the collateral.
* UTxOs containing native (non-ADA) assets could not be used as collateral.

**After Vasil Hard Fork**
* Users can specify exactly how collateral inputs will be spent if the script fails (using `cardano-cli transaction build-raw`).
* Native assets can be present in collateral UTxOs (although they cannot serve as payment for the fee).

*Further reading: [https://github.com/perturbing/vasil-tests/blob/main/collateral-output-cip-40.md](https://github.com/perturbing/vasil-tests/blob/main/collateral-output-cip-40.md)*

### **Claiming a Gift UTxO**
Now that we understand collateral, we're ready to construct the second transaction and claim a locked gift.

Since the "always succeeds" script address contains many locked UTxOs, it can be difficult to find the specific gift we just locked:
* We can use the `TxHash` of the transaction we just completed and infer the `TxIx` of the UTxO that went to the script (most likely `0`).
* We can unlock any other UTxO containing a `TxOutDatumHash` of the Unit value (`923918e403bf43c34b4ef6b48eb2ee04babed17320d8d1b9ff9ad086e86f44ec`).

Using one of the approaches above, find a suitable UTxO at the script address and assign it to the temporary variable `U`.

Then find any UTxO at the address of the user claiming the gift (i.e. `bob`), which we'll include as collateral. Assign this to the temporary variable `C`.

We can now construct, sign, and submit the transaction as follows:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-in-script-file $PLUTUS_SCRIPTS_PATH/gift.plutus \
--tx-in-datum-file $DATA_PATH/unit.json \
--tx-in-redeemer-file $DATA_PATH/unit.json \
--tx-in-collateral $C \
--change-address $(addr bob) \
--out-file $TX_PATH/gift-unlock-h.raw
```

```sh
cardano-cli transaction sign \
--tx-body-file $TX_PATH/gift-lock-h.raw \
--signing-key-file $KEYS_PATH/bob.skey \
--out-file $TX_PATH/gift-unlock-h.signed"
```

```sh
cardano-cli transaction submit \
--tx-file "$TX_PATH/gift-unlock-h.signed"
```

>Alternatively, using Cardano CLI Guru scripts: `tx-sign gift-unlock-h bob` & `tx-submit gift-unlock-h`.

## **Lock/Unlock UTxOs (with Inline Datum)**
The EUTXO model Cardano uses was originally designed to include datum contents in UTxOs rather than their hashes, with all the convenience this brings. A switch was made to datum hashes to avoid bloating UTxO entries, which were constant-sized prior to the Mary hard fork's introduction of a multi-asset ledger. Once multi-assets introduced variable-sized UTxO entries and accounting to support them, inline datums were able to be restored as a component of the Vasil hard fork.

We'll now repeat the same transaction process, but use an inline datum instead of a datum hash. This is achieved using the `--tx-out-inline-datum-file` option on the locking transaction:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-out "$(addr gift)+3000000" \
--tx-out-inline-datum-file "$DATA_PATH/unit.json" \
--change-address "$(addr alice)" \
--out-file $TX_PATH/gift-lock-i.raw
```

>**Note:** don't forget to assign a fresh UTxO to the `U` variable!

Sign and submit (`tx-sign gift-lock-h alice` & `tx-submit gift-lock-h`).

Now we'll construct a second transaction to unlock a gift. As with our previous example, it can be difficult to find the specific gift we just locked:
* We can use the `TxHash` of the transaction we just completed and infer the `TxIx` of the UTxO that went to the script (most likely `0`).
* We can unlock any other UTxO containing a `TxOutDatumInline`.

Assign a suitable UTxO at the script address to `U`, and construct the transaction as follows, using the `--tx-in-inline-datum-present` option in lieu of the datum file:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-in-script-file $PLUTUS_SCRIPTS_PATH/gift.plutus \
--tx-in-inline-datum-present \
--tx-in-redeemer-file $DATA_PATH/unit.json \
--tx-in-collateral $C \
--change-address "$(addr charlie)" \
--out-file $TX_PATH/gift-unlock-i.raw
```

This will use the inline datum present in the input UTxO, so our end user isn't responsible for providing it.

## **Lock/Unlock UTxOs (with Reference Script)**
Another new feature introduced by the Vasil Hard Fork is **reference scripts**, which allows the contents of scripts to be attached to outputs and used to satisfy script requirements during validation, rather than requiring the spending transaction to do so. Similar to the role inline datums serve for datums, reference scripts allow users to reuse the script data included in an existing UTxO rather than be responsible for providing it themselves.

While convenient, script referencing is costly, at least from the standpoint of the initial transaction that attaches the script to an output (and thus stores it on the active part of the chain). However, script referencing lowers the cost of every transaction referencing the script; if the script is used often, this can result in significant savings in transaction fees. It also reduces the quantity of data being stored on-chain, and increases the number of scripts that can be used in a single transaction without hitting the transaction size limit.

To lock a UTxO at the "always succeeds" address and add a reference to the script in the output UTxO, we include the `--tx-out-reference-script-file` option:

```sh
cardano-cli transaction build \
--tx-in $U \
--tx-out "$(addr gift)+3000000" \
--tx-out-inline-datum-file $DATA_PATH/unit.json \
--tx-out-reference-script-file $PLUTUS_SCRIPTS_PATH/gift.plutus \
--change-address "$(addr charlie)" \
--out-file $TX_PATH/gift-give.raw
```

>Note that this transaction also uses an inline datum. This is not required, but is done for convenience.

### **Unlock UTxO with reference script**
Unlocking a UTxO containing a reference script requires several additional options to the `transaction build` subcommand:
* `--spending-tx-in-reference`: provides the UTxO input containing the reference script.
* `--spending-plutus-script-v2`: signals the use of a Plutus v2 reference script.
* `--spending-reference-tx-in-inline-datum-present`: signals that an inline datum is present on the referenced UTxO (reference script version of `--tx-in-linline-datum-present`).
* `--spending-reference-tx-in-redeemer-file`: provides the redeemer to the script (reference script version of `--tx-in-redeemer-file`).

Here is our unlocking transaction:

```sh
cardano-cli transaction build \
--tx-in $U \
--spending-tx-in-reference $U \
--spending-plutus-script-v2 \
--spending-reference-tx-in-inline-datum-present \
--spending-reference-tx-in-redeemer-file $DATA_PATH/unit.json \
--tx-in-collateral $C \
--change-address "$(addr charlie)" \
--out-file $TX_PATH/gift-claim.raw
```
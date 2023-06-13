# **Cardano CLI Guru**
**Cardano CLI Guru** provides a wealth of convenient utility scripts to simplify the use of `cardano-cli`, as well as a guided tutorial to walk you through various `cardano-cli` operations.

This utility uses `direnv` to set and unset various environment variables whenever you navigate in and out of the `cardano-cli-guru` directory, allowing for more convenient use of `cardano-cli` without polluting your `~/.bashrc` or other dotfiles with additional variables. These local variables can be adjusted to your needs by modifying the contents of the `.env` file that is automatically created when you allow `direnv` to run in the directory the first time.


## **Setup**
1. Install `cardano-node` and `cardano-cli`. 
    * You can use the **[Cardano EZ-Installer](https://github.com/iburzynski/cardano-ez-installer)** to easily install both applications via Nix and automatically configure your node for use with all three networks (`preprod` and `preview` testnets, `mainnet`).

2. Install the `direnv` utility. 
    * If you have Nix installed, the easiest way to do this is via the following Nix command:

        ```sh
        nix profile install nixpkgs#direnv
        ```

      >**NOTE:** this command requires the `experimental-features` `nix-command` and `flakes` to be enabled in your `/etc/nix/nix.conf` file:

        ```
        experimental-features = nix-command flakes
        ```

      >You'll need to restart the `nix-daemon` after making changes to `nix.conf`:

        **Linux:**

        ```sh
        sudo systemctl restart nix-daemon
        ```

        **MacOS:**

        ```sh
        sudo launchctl stop org.nixos.nix-daemon
        sudo launchctl start org.nixos.nix-daemon
        ```


3. Clone this repository in your terminal and navigate to the `cardano-cli-guru` directory:

    ```sh
    git clone https://github.com/iburzynski/cardano-cli-guru
    cd cardano-cli-guru
    ```

4. You will see the following error message:
    
    ```sh
    direnv: error /home/your-username/path-to-cardano-cli-guru/.envrc is blocked. Run `direnv allow` to approve its content
    ```

    This is a security measure, since `.envrc` files can run arbitrary shell commands. Make sure you always trust the author of a project and inspect the contents of its `.envrc` file before running `direnv allow`.

    When you're ready, enter `direnv allow` to approve the content:

    ```sh
    direnv allow
    ```

5. Adjust `.env` variables as needed.
    * When you allow `direnv` to run in the `cardano-cli-guru` directory, a `.env` file is automatically created containing various environment variables with default settings. These variables are used by `cardano-cli` and the various utility scripts provided by `cardano-cli-guru` and can be adjusted according to your needs.
    * The `CARDANO_NODE_NETWORK_ID` variable tells `cardano-cli` which network your node is currently using. By default this is set to `2` for `preview` testnet. Change this value to `1` if you want your node to run on the `preprod` testnet, or to `mainnet` for the mainnet.
    * Two variables are defined for interacting with Blockfrost, which is used to retrieve transaction metadata in the Tutorial. You can replace the placeholder value for either `BLOCKFROST_PROJECT_ID_PREPROD`or `BLOCKFROST_PROJECT_ID_PREVIEW` with your Blockfrost Project ID, depending on which network you are using in your Blockfrost project.
    * The remaining `*PATH` variables are set to existing subdirectories of `cardano-cli-guru`, and tell the utility scripts where to store the various output files of the utility scripts. You can change these to point to different directories if you'd like your output files to be stored somewhere else, although it is easier to keep the default locations.
        
        >**NOTE:** if you wish to change any of the `*PATH` variables, make sure your filepaths are defined relative to the `cardano-cli-guru` directory, or use absolute filepaths.

    * If you changed any values in `.env`, run `direnv allow` again in the `cardano-cli-guru` directory so `direnv` updates the variables.

        >**NOTE:** You must run `direnv allow` whenever you modify `.env`, or the variables will still be set to their previous values in the environment.

6. Start `cardano-node`.
    * You're now ready to start your node and begin using `cardano-cli`. In a separate terminal session, run the appropriate command to start `cardano-node` on your desired network (i.e. `preprod-node` or `preview-node` if you installed using **Cardano EZ-Installer**).

        >**NOTE:** Make sure the value of `CARDANO_NODE_NETWORK_ID` in the `.env` file corresponds to the network you are running!

    * It's normal for the node to encounter occasional errors, which it will recover from and continue running. To tell if your node is working properly, look for `Chain extended` log entries with the following format:

        ```sh
        Chain extended, new tip: c472036b83c119b875e3fc230435b741598677ffa45ea3ad8ad9cda3f70a872d at slot 12227931
        ```

    * After giving the node a little time to boot up, try running the `tip` command from the `cardano-cli-guru` directory. You should see output informing you of the current slot number and the percentage that your node is synced. Once your node is 100% synced you can begin using the other utility scripts to interact with the blockchain.

        ```sh
        $ tip
        
        {
            "block": 546242,
            "epoch": 141,
            "era": "Babbage",
            "hash": "7ee471e26ed927ae463d386cdd322fd7f3afb18d0fef462255ce2a2f221d7112",
            "slot": 12227857,
            "syncProgress": "100.00"
        }
        ```

    * When you are finished, make sure to properly close the node connection by typing `CTRL + c` in the terminal session where it's running. If you close the terminal without doing this, the socket will remain open and you won't be able to start the node again unless you manually kill the associated process or restart your system!


At this point you're ready to begin using `cardano-cli` with Cardano CLI Guru! 

***
## **`cardano-cli` Tutorial**

**Cardano CLI Guru** provides a tutorial in `cardano-cli-guru/tutorial`, with guided exercises for building/submitting various types of transactions using `cardano-cli`. The tutorial begins with the simplest possible transaction and progresses through increasingly complex examples.

Many prewritten utility scripts have been provided to make the process less laborious, but you should always inspect the contents of these scripts before using them to understand the structure of the underlying `cardano-cli` commands.

For example, if we inspect the contents of the `tip` script at `cardano-cli-guru/scripts/tip` we'll see the following:

```sh
# cardano-cli-guru/scripts/tip

cardano-cli query tip
```

This script uses `cardano-cli`'s `query` command to query information about the `tip` of the chain.
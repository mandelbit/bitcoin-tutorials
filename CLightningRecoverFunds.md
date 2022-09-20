## How to recover funds from a unilateral channel closure (force close)

The aim of this guide is to present a clear and complete procedure to recover channel funds from a c-lightning node when all is lost but the ```hsm_secret```. (**always create a secure backup copy of the hsm_secret** before depositing funds to your node e.g. by saving on BitWarden).

What's that? You didn't backup your ```hsm_secret```? Well unfortunately, in that case you're up sh*t creek. The ```hsm_secret``` is essential to recover funds (similar to the BIP 39 seed phrase for onchain wallets) and without it unfortunately there is zero chance of recovering your c-lightning funds. If this is you, try to take it as a learning experience and don't be quite so #reckless next time (always backup your secrets!).

If you do have your hsm_secret (but nothing else), and your node has spontaneously combusted, transported to another spacetime continuum or otherwise left us (RIP node) then this guide is for you! If you follow this guide carefully, chances are you'll have your sats safely back in their stack (or scattered around another set of channels) soon.

This guide is motivated by the tragic loss of my own node ( :`( ), (corrupted filesystem due to power outage) running on a Raspberry Pi, and unexplicable loss of the RAID 1 backup of the .lightning directory. When going through the process of recovering funds I found that the information is still quite sparse and disparate, so this is an attempt to save others some time.

# First things first

If you do not have access to the latest channel backup state (or you are in any way not sure whether the channel backup you have is the most recent state), then **do not** attempt to boot your node back up. 

Attempting to restore the node with a so-called 'toxic' (old) state could lead to your node broadcasting this toxic state to its peers, who will see this misinformation as an attack and perform a 'justic transaction', closing the channel and taking **all funds** in the channel themselves. 

Thus, it is **extremely important** that you do not attempt to boot up a node when there is a possiblity that some of the channel state has been lost/if the channel state is out of date. Doing so could result in the loss of all of your funds in each channel where toxic state was broadcasted.

Another important point is that you can only use the method that follows **after** the channels have been (force) closed by the channel counterparty node. It’s likely that you’ll need to wait for the other node to force close the channel.

Another possibility is to reach out to the operator of the other node and ask them to force close your shared (rekt) channel. But **only do this if you are very confident that you can trust the other node operator not to steal your sats** (which they can do by having their node broadcast a malicious state that takes all of your money and that your node cannot contest -since it’s offline). 

**It's very important at this point to leave your corrupted/dead node offline**. 

# Setting up c-lightning

For this recovery, we'll need to use some tools provided by c-lightning so the first step is to clone the repo locally then build it.

Once complete, clone lightning from GitHub and cd into the new directory:

```
git clone https://github.com/ElementsProject/lightning.git
cd lightning
```

For Mac, you’ll need to configure Python 3.x and install `mako` with pip:
```
pyenv local 3.7.4
pip install mako
```

Now build lightning:
```
./configure
make
```

Lightning should now be installed.

# Using the hsmtool

Since we don't really care about running lightning here, and we're just looking to recover funds from the secret, there's no need to test or run `lightningd`. 
We can check our tools are available by running the following command from the root of the cloned lightning directory.

```
./tools/hsmtool help
```

You should see a list of methods provdided by the hsmtool (which we're about to use) along with their respective parameters.

Let's create a new hidden .lightning-recovered directory in the default $HOME/ location to deal with our `hsm_secret` and enter this directory. Location isn't important, you can also use any another secure location on your local machine. 

```
mkdir $HOME/.lightning-recovered
cd $HOME/.lightning-recovered
```

Use your editor of choice to create a new file to store your backed up hsm_secret hex (that you exported and saved using `xxd hsm_secret` before, when you set up your node).

```
nano hsm_secret_hex.txt
```

Paste your hsm_secret in there.

Now lets obtain the secret in the format hsmtool wants to see it in.

```
xxd -r hsm_secret_hex.txt
```

Copy this… 

…and paste the secret into a new file to be read by c-lightning.

```
sudo nano hsm_secret
```

(paste here)

Now move back into your lightning directory (where you git clone’d lightning) in another terminal window.

```
cd <path/to/lightning/dir>/lightning
```

We’re going to be using some of the helpful methods provided by the hsmtool, which you can view using the following command again.

```
./tools/hsmtool help
```

If necessary, decrypt your hsm_secret using
```
./tools/hsmtool decrypt <path/to/hsm_secret>
```

# Retrieving info from node

Go to 1ml.com and search for your node (using the alias you configured when you setup the node). Scroll down to the channel, and save the node_id of your channel counterparty somewhere. Now find the ‘Channel Point’ and paste it into a block explorer. You should see the address used to fund the channel.

Search/click on this address. If the channel has successfully closed, you should see a transaction from this funding address to another address that now holds your (ex)channel funds. Copy this address and save it somewhere, we’ll need it in a minute.

Alternatively, you can use...
```
./cli/lightning-cli listtransactions
```

...to find the `txid` associated with the funding transactions for the relevant channels and the location of your bitcoin following channel closure.

To recover your funds, we’ll need to obtain the private key associated with the address that holds your bitcoin. For this, it’s necessary to obtain something called the per_commitment_point which is required to derive this privkey in the following way:

```
privkey = basepoint_secret + SHA256(per_commitment_point || basepoint)
```
Learn more [here](https://bitcoin.stackexchange.com/a/90719).

If this seems complicated, don’t worry. A manual/deterministic recovery would require you to obtain the relevant commitment from the `dumpcommitments` hsmtool method.

But we're actually going to use an even easier method provided by `hsmtool`, the `guesstoremote` method, to locate the privkey associated with the address holding our funds, using the relevant command:
```
./tools/hsmtool  guesstoremote <P2WPKH address> <node_id> <tries> <path/to/hsm_secret>
```

*P2WPKH address* - this is the address holding your now on-chain funds (one of the two recievers of the channel closing transaction)
*node_id* - the id of the other node involved in the channel (use lightning-cli or 1ml/equivalent to find this)
*tries* - I believe this is the same as max_channel_dbid from the [docs](https://lightning.readthedocs.io/lightning-hsmtool.8.html?highlight=hsmtool) “is your own guess on what the channel_dbid was, or at least the maximum possible value, and is usually no greater than the number of channels that the node has ever had” - just take a guess at whatever number this channel was and multiply it by ten for safe keeping to make sure you find the right commitment
*path/to/hsm_secret* - self-explanatory

The above command may yield the following response:

*Could not find any basepoint matching the provided witness programm.
Are you sure that the channel used `option_static_remotekey` ?*

If so, try to increase the number of tries. If that does not work (shouldn’t need a very high number assuming you’re not Alex Bosworth, in which case, why are you reading this guide?) then probably best checking that you have the correct `P2WPKH address` and associated `node_id`.

A successful result should be of the following form:

*bech32: […]*
*pubkey hash: […]*
*pubkey: […]*
*privkey: […]*

The fruit of our labour is the privkey - derived by hsmtool using the above formula.

However, we need the privkey in wallet import format (WIF) to be able to make use of it to recover the funds. 

# Obtaining the WIF

There are many ways you can obtain the WIF. To save some time, I went with the [bitcoinjs WIF npm package](https://github.com/bitcoinjs/wif) (BitcoinJS is another very popular widely used piece of open source software).

Clone and install the repo locally:
```
git clone https://github.com/bitcoinjs/wif.git
cd wif
npm install
```

Now we can create a file using the code example provided in the README, replace the dummy privkey with our own and run the code.

```
nano recover.js
```
(paste code and add your `privkey`)

```
node recover.js
```
Save the WIF somewhere secure for now. Again, note that this is a **private key**, so protect and secure it as such until you have moved the bitcoin to a safer location (recommend you do this ASAP).

# Recovery

Once we have obtained the WIF, there are a number of ways we can recover the money.

One method would be to import the WIF into a new or existing wallet e.g. by making use of Bitcoin Core’s `importprivkey` command to import the individual address and spendable funds to a Bitcoin Core wallet.dat file. Advantages of using this method include the quality and reliability of BC’s code - as reference implementation and the most reviewed code in the world of Bitcoin. One major disadvantage of just importing the WIF and leaving funds where they are is that there is a chance that in obtaining the WIF, you have exposed it/the privkey to the outer world in some way. To eliminate the risk of this potentiality, its better to spend the funds from this address (which of course can also be done shortly after the Bitcoin Core import).

Another option is to use software like [Coinbin](https://coinb.in) to construct, sign and broadcast a transaction moving your bitcoin from this location to a more secure offline wallet/another lightning node/etc. Note that if you use the website directly, you are to an extent trusting that coinb.in are not going to steal your money when you sign the transaction with your WIF. This risk is mediated by the fact that coinbin is fully open source (so you can check for funny business) with considerable use and likely a lot of eyes on the code (also checking for funny business). For full paranoia mode you can build from source, or even use Bitcoin Core to construct the transaction.

Yet another option is to use [Electrum](https://electrum.org) to import your private keys, and sweep your bitcoin to a destination that you control. To do this, install latest version of Electrum following installation instructions (remember to validate the integrity of your installation). Then, go to `file` > `new/restore wallet`, select `Import Bitcoin address or private keys`, paste your private key (prepending the address type used during execution of `hsm_secret` command). 

The following are a few examples for different script types:
```
p2pkh:KxZcY47uGp9a...       	-> 1DckmggQM...
p2wpkh-p2sh:KxZcY47uGp9a... 	-> 3NhNeZQXF...
p2wpkh:KxZcY47uGp9a...      	-> bc1q3fjfk...
```
Send funds to the desired destination.

I went with the latter option for convenience. If you are unfamiliar with coinb.in there’s a lot of information about usage on their [blog](https://blog.coinb.in/guides).

Now you can spend your funds to a secure address and make peace knowing that you have safely recovered and secured your sats after all hell went loose and you lost everything but your hsm_secret.

Have fun staying secured!

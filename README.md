Download, install and check version for relayer

```
git clone https://github.com/cosmos/relayer.git
cd relayer
make install
cd
rly version
```

Version should be 1.0.0-rc1-152-g112205b or similar

Initialise the repeater

```rly config init```

Create a folder for network configuration files and configs for both networks (KiChain and Rizon)
```mkdir rly_config
cd rly_config
nano kichain-t-4.json
```

Paste following config to created json and save one
```
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://http://localhost:26657",   
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```
The same for Rizond network

```nano groot-011.json```

Paste following config to created json for Rizond and save one (do not forget to replace rpc-addr param for the proper one (if node is installed locally for example))
```
{
  "chain-id": "groot-011",
  "rpc-addr": "http://144.91.92.234:26657",   
  "account-prefix": "rizon",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025uatolo",
  "trusting-period": "48h"
}
```

Add created config setting into the config of the relay
```
rly chains add -f groot-011.json
rly chains add -f kichain-t-4.json
cd
```

Add (or restore from existing) wallets of both networks. 
To add new
```
rly keys add groot-011 your_wallet_name
rly keys add kichain-t-4 your_wallet_name
```

To restore existing (preferable way, just use the same mnemonic for convenience)
```
rly keys restore groot-011 your_wallet_name "mnemonic for wallet"
rly keys restore kichain-t-4 your_wallet_name "mnemonic for from wallet"
```

Add the newly created keys to the config of the relay:
```
rly chains edit groot-011 key your_wallet_name
rly chains edit kichain-t-4 key your_wallet_name
```

Change timeout for confirmation from 10 to 30s:

```nano ~/.relayer/config/config.yaml```
Set timeout: 30s  

Check wallets balances, it must have coins for next steps
If your test Rizon wallet has zero balance take a look the Rizon faucet http://faucet.rizon.world/

```
rly q balance groot-011
rly q balance kichain-t-4
```

Initialize the light client in both networks
```
rly light init groot-011 -f
rly light init kichain-t-4 -f
```

Create a channel between networks

```rly paths generate groot-011 kichain-t-4 transfer```

The output should be as follows: `Generated path(transfer), run 'rly paths show transfer --yaml' to see details`

Open a channel for relaying:

```rly tx link transfer --debug```

Probably you will get the error that the channel does not open. If so, go to the config with the command:

```nano ~/.relayer/config/config.yaml```

Erase the values for lines in the path section on both networks:

```client-id: 07-tendermint-16  connection-id: connection-14```

Than re-initialize the light client with the commands:
```
rly light init groot-011 -f
rly light init kichain-t-4 -f
```

And than run the command to open the channel again

```rly tx link transfer --debug```

The success is the message like "★ Channel created: [groot-011]chan{channel-11}port{transfer} -> [kichain-t-4]chan{channel-41}port{transfer}"

Now check the path list.

```rly paths list -d```

The output should be as follows
0: transfer -> chns(✔) clnts(✔) conn(✔) chan(✔) (groot-011:transfer<>kichain-t-4:transfer)

Try to perform an cross-network transaction. From Rizon to KiChain. The pattern is:

```rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags] ```

For current test we used the string like that:

```rly tx transfer groot-011 kichain-t-4 1000000uatolo tki1ft56wsankvpw02ul4mg86udytn9ex2cw4nnr2a --path=transfer```

Success is a string like that with hash:  ✔ [groot-011]@{424876} - msg(0:transfer) hash(4419E0A9975BBD5A6475C549039564DA9ABF096AAA63D2C2949AED2A601763B0)

Let's check the hash here
* https://testnet.mintscan.io/rizon/txs/4419E0A9975BBD5A6475C549039564DA9ABF096AAA63D2C2949AED2A601763B0
* https://testnet.mintscan.io/rizon/account/rizon1ft56wsankvpw02ul4mg86udytn9ex2cwcn5508
* https://ki.thecodes.dev/tx/7741335A1863A9024A5C053B56B9D3A1BFC875D24D0E8B293F1D5AB12AF382BB

Another hash of successfully performed transaction: E7FA050721C29923F9AA863F1472B5CB14B57BA406F3C0A7772E5346F8A787ED
D67FD231104CFB1D2C7FB21DCC7D934A1D0D831D3B4B978B30AF4FFECE506572

Now try to perform transaction in opposite direction: from KiChain to Rizon

```rly tx transfer kichain-t-4 groot-011 1000000utki rizon1ft56wsankvpw02ul4mg86udytn9ex2cwcn5508 --path=transfer```

Success is a string like that with hash: ✔ [kichain-t-4]@{227477} - msg(0:transfer) hash(9DC6DAE6E5C76452D2F6F3DC7C452D4EB26C1ADDF2B912B3D139FE0526603A7C)

Let's check the hash here
* https://ki.thecodes.dev/tx/9DC6DAE6E5C76452D2F6F3DC7C452D4EB26C1ADDF2B912B3D139FE0526603A7C
* https://testnet.mintscan.io/rizon/txs/68267BC03E6FCADAF33F29C826316B800BF48918BAEC2D0AFBC1A8E58BC47714


Finally let's config relayer as remote service for any wallet sending

Create service file (below is an example! correct your pathes and user name)
```
sudo touch /etc/systemd/system/rlyd.service
sudo nano /etc/systemd/system/rlyd.service
```

Paste the strings
```
[Unit]
Description=relayer client
After=network-online.target, kichaind.service
[Service]
User=hakimus
ExecStart=/home/hakimus/go/bin/rly start transfer
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
```

Start service
```
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```

To transfer use the command like this

```kid tx ibc-transfer transfer transfer channel-N rizon_WALLET_address 1000utki --from name_OF_wallet --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid```

To determine channel-N use **dst** channel-id param value from output of

```rly paths show transfer --yaml```

We used command sample like this
```
cd ./kichain
kid tx ibc-transfer transfer transfer channel-80 rizon1ft56wsankvpw02ul4mg86udytn9ex2cwcn5508 1000utki --from cosinus --fees=5000utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid
```

Reverse transaction from Rizon to KiChain (will work if you have Rizon node installed)

```rizond tx ibc-transfer transfer transfer channel-N tki_wallet_adress 1000uatolo --from your_wallet_name --fees=5000uatolo --gas=auto --chain-id groot-011```

To determine channel-N use **src** channel-id param value from output of

```rly paths show transfer --yaml```

We used command sample like this

```rizond tx ibc-transfer transfer transfer channel-19 tki1ft56wsankvpw02ul4mg86udytn9ex2cw4nnr2a 1000uatolo --from cosinus --fees=5000uatolo --gas=auto --chain-id groot-011```











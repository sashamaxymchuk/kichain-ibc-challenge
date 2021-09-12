# Kichain IBC Challenge Guide
In this guide I'll describe how to run IBC relayer for Ki-Chain IBC Challenge
For this challenge I've run 2 relayers in next directions:
kichain-t-4 <-> testnet-croeseid-4
kichain-t-4 <-> cygnusx-osmo-1

# 1. Installing clients for additional networks on Ki-Chain Node
# 1a. Cygnusx-Osmosis Chain
Download and install the network client
```git clone https://github.com/osmosis-labs/osmosis
cd osmosis
git checkout v1.0.1
make install
cd ~    
```
Create a wallet (be sure to save all output lines, there will be a seed phrase from the wallet, they should always be saved)
```
osmosisd keys add OSMO_WALLET_NAME
```
Or, if you already have a wallet with a balance in this network, you can restore it using the seed phrase
```
osmosisd keys add OSMO_WALLET_NAME --recover
```
Look at the address of your wallet
```
osmosisd keys show OSMO_WALLET_NAME -a
```
Check the balance to make sure that you have tokens on balance
```
osmosisd query bank balances OSMO_WALLET_ADDRESS --node http://54.166.148.90:36657
```
# 1b. Croeseid Chain
Download and install the network client
```
curl -LOJ https://github.com/crypto-org-chain/chain-main/releases/download/v3.0.0-croeseid/chain-main_3.0.0-croeseid_Linux_x86_64.tar.gz
mkdir croeseid
tar -C croeseid -zxvf chain-main_3.0.0-croeseid_Linux_x86_64.tar.gz
cd croeseid / bin
sudo cp chain-maind / usr / local / bin /
cd
```
Create a wallet (be sure to save all output lines, there will be a seed phrase from the wallet, they should always be saved)
```
chain-maind keys add CRO_WALLET_NAME
```
Look at the address of your wallet
```
chain-maind keys show CRO_WALLET_NAME -a
```
You can request tokens by the link
https://crypto.org/faucet
After confirming the sending of tokens, check the balance of the wallet
```
chain-maind q bank balances CRO_WALLET_ADDRESS --node https://testnet-croeseid-4.crypto.org:26657
```
# 2. Run relayer
I used Go implementation of IBC relayer. Also, you can use Hermes - it's rust implementation
Download the relayer and install it
```
git clone https://github.com/cosmos/relayer.git
cd relayer
make install
cd
```
Initialize the relay
```
rly config init
```
Create a folder in which we will put folders with data for the configuration of the relay
```
mkdir rly_config
cd rly_config
```
Create config files for the networks tou choosed:
Ki-chain:
```
nano kichain-t-4_config.json
# Inside the file, insert such data
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://127.0.0.1:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```
Osmosis:
```
nano cygnusx-osmo-1_config.json
{
  "chain-id": "cygnusx-osmo-1",
  "rpc-addr": "http://54.166.148.90:26657",
  "account-prefix": "osmo",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025uosmox",
  "trusting-period": "48h"
}
```
Croeseid:
```
nano testnet-croeseid-4_config.json
{
  "chain-id": "testnet-croeseid-4",
  "rpc-addr": "https://testnet-croeseid-4.crypto.org:26657",
  "account-prefix": "tcro",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025basetcro",
  "trusting-period": "48h"
}
```
Add pre-written settings to the config file of the relay and return to home directory
```
rly chains add -f FILE_NAME
cd
```
Add wallets in both networks to the relayer (it is recommended to create separate wallets for this). Remember to save your seed phrases and addresses
```
rly keys add <CHAIN_ID> RLY_WALLET_NAME
```
Add the created wallets to the config of the relay
```
rly chains edit <CHAIN_ID> RLY_WALLET_NAME
```
Change the wait timeout in the relay settings so that it can accurately process transactions
```
nano ~ / .relayer / config / config.yaml
```
Looking for a string
```
timeout: 10s
```
Replace with
```
timeout: 3m
```
Send tokens to the relayer wallets (5-10 coins each)
```
kid tx bank send <KICHAIN_WALLET_NAME> <KICHAIN_RLY_WALLET_ADDRESS> 10000000utki --home PATH_TO_YOUR_NODE_DIRECTORY --chain-id kichain-t-4
# Repeat this comand replacing kid by other node binary, utki by denom for other chain tokens, --chain-id by actual for chains
# Also for other chains use flag --node <RPC_ADDR_FROM_RLY_CONFIG>
```
Check that the tokens have arrived
```
rly q balance <CHAIN_ID>
```
When the tokens arrive on the wallets of the relay, initialize clients for both networks
```
rly light init <CHAIN_ID> -f
# Repeat for each chain
```
Create IBC-channel between the two networks.
```
rly paths generate kichain-t-4 <DST_CHAIN_ID> transfer --port = transfer
```
Check that the IBC-channel has been successfully created.
```
rly paths list -d
```
Ideally, you should see this output.
'''
0: transfer -> chns (✔) clnts (✔) conn (✔) chan (✔) (kichain-t-4: transfer <> cygnusx-osmo-1: transfer)
'''
If you are lucky and your щгезге is like this, then feel free to skip туче step. If your щгезге looks different, for example like this
'''
0: transfer -> chns (✔) clnts (x) conn (x) chan (x) (kichain-t-4: transfer <> cygnusx-osmo-1: transfer)
'''
go to the next step
Create in the connected networks clients, channels and connections necessary for the operation of the IBC channel
You can use step-by-step subcommands of 
```
rly tx raw -h
```
But you have more easy option:
```
rly tx link transfer
```
Wait for the command to complete. The last line will contain the following fragment
```
★ Channel created:
```
The transaction normally takes few minutes, but sometimes take more than an hour. If your transaction is stuck at some stage or an error pops up, open the config of the relay
```
nano ~ / .relayer / config / config.yaml
```
and remove in the paths section the lines
```
client-id: ...
connection-id: ...
channel-id: ...
```
in src and dst
then re-conduct the transaction
After successful execution of the transaction, re-check whether the checkboxes appeared everywhere in output of
```
rly paths list -d
```
If they do, create a service file (ATTENTION: this guide assumes that a file called kichaind.service is used to start the node. it is different, replace it with the correct one in the [Unit] section of the service file):
```
sudo tee /etc/systemd/system/rlyd.service> / dev / null << EOF
[Unit]
Description = relayer client
After = network-online.target, kichaind.service
[Service]
User = $ USER
ExecStart = $ (which rly) start transfer
Restart = always
RestartSec = 3
LimitNOFILE = 65535
[Install]
WantedBy = multi-user.target
EOF
```
Start the service file
```
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```
To check the logs of the relayer, use:
```
journalctl -u rlyd -f
```
To send an ibc transaction, you can use both the subcommands of the relay and, accordingly, the wallets that are added to it, as well as the subcommands of the clients of the networks and the wallets added to them. Examples:
Relayer:
```
rly tx transfer kichain-t-4 cygnusx-osmo-1 1000000utki <DST_OSMO_ADDRESS> --path transfer
```
Node Client
```
kid tx ibc-transfer transfer PORT_ID CHANNEL_ID RECIPIENT_ADDRESS_IN_OSMO_NETWORK 1000000utki --from KICHAIN_WALLET_NAME --home PATH_TO_YOUR_NODE_DIRECTORY --chain-id kichain-t-4
# Repeat this comand replacing kid by other node binary, utki by denom for other chain tokens, --chain-id by actual for chains
# Also for other chains use flag --node <RPC_ADDR_FROM_RLY_CONFIG>
```
To get CHANNEL_ID and PORT_ID use the command
```
rly paths show transfer --yaml
```
You need the value of port-id and channel-id item of the src section
After the transaction, check the balance of the recipient's wallet to make sure that the tokens have arrived
An entry similar to the following should appear there.
```
- amount: "1000000"
   denom: ibc / 1340831AABAFC9C8127DF07AD624F96CDADD34005107E1C4B8426656FE762D28
```
If your tokens did not arrive to the wallet in another network (such a situation came across when a transaction was created using subcommands of network clients), stop the relay
```
sudo systemctl stop rlyd
```
update your light clients
```
rly tx update-clients transfer
```
and restart the service
```
sudo systemctl start rlyd
```
After that, wait for a while (I recommend about a minute, you can wait a little longer), re-check receiver balance and, if necessary, repeat the transaction.


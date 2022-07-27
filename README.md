# Ansible Process for Polygon RPC Node (Mumbai)

## Requirements

- Geth
	- Goerli
	- 4core 8gb
	- Geth Goerli = ~190GB
- Polygon
	- mumbai
	- 8core 16gb
	- Heimdall testnet = ~40GB
	- Bor testnet = ~160GB
	- need free floating 120GB if plan to download & tar snapshot
- Open Ports (remote machine)
	- 26656
	- 30303
	- 22222 (adapted SSH)
	
## Pre requisites

We use a secure server setup executed through Ansible. Please follow [this](https://github.com/Throbac-Labs/secure-server-setup) repository using your control machine to do initial setup for your Polygon RPC Node server.

This server set up will install/create/enable the following:

- sudo users ubuntu & ansible
- docker
- ufw
- ssh on port 22222

## Full Node Setup for Testnet-v4

Clone & cd into repo
```
git clone https://github.com/Throbac-Labs/polygon-ansible.git
cd polygon-ansible
```
edit "inventory" to add your Polygon RPC Node server IP Address
```
nano inventory
sentry ansible_host=YOUR-POLYGON-NODE-SERVER-IP
```
Save and exit.

Next, test connection to host. 
Remember, this documentation assumes you have used the secure-server-setup above (we use it on both our control machine and polygon node server).
```
ansible-playbook -l sentry playbooks/network.yml --extra-var="bor_branch=v0.2.16 heimdall_branch=v0.2.9 network_version=testnet-v4 node_type=sentry/sentry heimdall_network=mumbai" --list-hosts
```
Now, run the playbook.
This will set up & install all the necessary dependencies, files, and configurations for your Polygon RPC Node. 
```
ansible-playbook -l sentry playbooks/network.yml --extra-var="bor_branch=v0.2.16 heimdall_branch=v0.2.9 network_version=testnet-v4 node_type=sentry/sentry heimdall_network=mumbai"
```
Once the setup completes from our control machine, we have some final configuration inside the Polygon Node Server. 

## Inside Polygon RPC Node Server

SSH into the Polygon server and execute the following:

Add unique identifier & seeds to Heimdall config
```
nano ~/.heimdalld/config/config.toml
moniker=<enter unique identifier>
seeds="4cd60c1d76e44b05f7dfd8bab3f447b119e87042@54.147.31.250:26656"
# save & exit
```
Add ETH RPC URL (yours or 3rd party)
```
nano ~/.heimdalld/config/heimdall-config.toml
eth_rpc_url =<insert Infura or any full node RPC URL to Goerli>
# save & exit
```
Edit Bor start script params
```
nano ~/node/bor/start.sh
--bootnodes "enode://320553cda00dfc003f499a3ce9598029f364fbb3ed1222fdc20a94d97dcc4d8ba0cd0bfa996579dcc6d17a534741fb0a5da303a90579431259150de66b597251@54.147.31.250:30303"
# append the above to the end of the file
# save & exit
```
Update UFW
```
sudo ufw allow 30303
sudo ufw allow 26656
```
## Download Snapshots for fast sync

Daily snapshots can be found here: https://snapshots.matic.today

Heimdall example
```
# i typically use "screen" to avoid having to wait for the below to download & unzip
wget -c https://matic-blockchain-snapshots.s3-accelerate.amazonaws.com/matic-mumbai/heimdall-snapshot-2022-07-26.tar.gz
# tar -xzvf <snapshot file> -C <HEIMDALL_DATA_DIRECTORY>
tar -xzvf heimdall-snapshot-2022-07-26.tar.gz -C ~/.heimdalld/data/
```
Bor example
```
wget -c https://matic-blockchain-snapshots.s3-accelerate.amazonaws.com/matic-mumbai/bor-fullnode-snapshot-2022-07-26.tar.gz
# tar -xzvf <snapshot file> -C <BOR_DATA_DIRECTORY>
tar -xzvf bor-fullnode-snapshot-2022-07-26.tar.gz -C /mnt/bordata/.bor/data/
# if you use a different directory than default as i have above, make sure it exists with proper permissions & you add the path to ~/node/bor/start.sh
```

## Start & Sync Heimdall

Before you can start Bor, Heimdall must be synced.
```
sudo systemctl heimdalld start
sudo systemctl heimdalld-rest-server start
```
Once synced, you can run Bor
```
sudo systemctl bor start
```

## Monitoring

```
# if you followed our server-setup you may need to restart journald to see logs
sudo systemctl restart systemd-journald.service
```
Heimdall Monitoring
```
sudo journalctl -u heimdalld.service -f
curl localhost:26657/status
sudo journalctl -u heimdalld-rest-server.service -f
```
Bor Monitoring
```
sudo journalctl -u bor.service -f
```
Network Monitoring
```
curl localhost:26657/net_info? | jq .result.n_peers
# check heimdall & bor network height here: https://wallet-dev.polygon.technology/staking/
```









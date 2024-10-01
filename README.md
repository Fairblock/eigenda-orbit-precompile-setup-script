# eigenda-orbit-precompile-setup-script

## Instructions

1. Compile the precompile nitro-node image

```shell
git clone https://github.com/Fairblock/nitro-eigenda-precompile
cd nitro-eigenda-precompile
git submodule update --init --recursive --force
```
Execute the following command to generate go abi file in the `nitro/solgen` directory:

```shell
make contracts
```

Compile the docker image:

```shell
make docker
```

Get the WASM module root:

```shell
docker run --rm --name replay-binary-extractor --entrypoint sleep nitro-node-dev infinity
docker cp replay-binary-extractor:/home/user/target/machines/latest/ extracted-replay-binary
docker stop replay-binary-extractor
cat extracted-replay-binary/latest/module-root.txt
```

Create a directory based on this hash, copy `machine.wavm.br`, `module-root.txt`, `replay.wasm`, and `until-host-io-state.bin` files from the extracted-replay-binary directory to the newly created directory:

```shell
mkdir -p target/machines/$(cat extracted-replay-binary/latest/module-root.txt)
cp extracted-replay-binary/latest/* target/machines/$(cat extracted-replay-binary/latest/module-root.txt)
```

Edit the Dockerfile file in the root of the folder, and after all the `RUN ./download-machine.sh ...` lines, add:

```shell
COPY target/machines/<wasm module root> <wasm module root>
RUN ln -sfT <wasm module root> latest
```
Replace each <wasm module root> with the WASM module root you got earlier.

Recompile docker image:

```shell
make docker
```

2. Clone the eigenda-orbit-precompile-setup-script repository and install dependency
```shell
git clone https://github.com/Fairblock/eigenda-orbit-precompile-setup-script
cd eigenda-orbit-precompile-setup-script
yarn install
```

3. Deploy contracts and obtain the configuration file

According to instructions in the website `https://orbit.eigenda.xyz` to deploy contracts and obtain `nodeConfig.json` and `orbitSetupScriptConfig.json` configuration file. Then place the two files in the `config` directory of the `eigenda-orbit-precompile-setup-script`

4. Get L1 RPC and config it in the config files

You can use a rpc from rpc providers or deploy a L1 node and replace them with your rpc in the two files: 

* nodeConfig.json

```json
{
   ...
   "parent-chain": {
    "connection": {
      "url": "<YourRPC>"
    },
    "blob-client": {
      "beacon-url": "<YourRPC>"
    }
  }
  ...
}
```

* orbitSetupScriptConfig.json

```json
{
  ...
  "parent-chain-node-url": "<YourRPC>",
  ...
}
```

5. Setup and start the chain

```shell
EIGENDA_PROXY_PRIVATE_KEY="YourPrivateKey" ETH_RPC_URL="${YourRPC}" docker compose up -d
PRIVATE_KEY="YourPrivateKey" PARENT_CHAIN_RPC_URL="${YourRPC}" ORBIT_RPC_URL="http://127.0.0.1:8449" yarn run setup
```

`EIGENDA_PROXY_PRIVATE_KEY` need to remove the `0x` prefix of the private key.

6. Change wasm module root on the deployed contract

Shut down the chain first.

```
docker compose down
```

then send tx to change wasm module root on contract

```
ETH_RPC_URL=<YourRPC> cast send <upgradeExecutorContractAddress> "executeCall(address,bytes)" <rollupContractAddress> 0x89384960<newWasmModuleRoot>  --gas-limit 10000000 --private-key "0xxxx"
```

`newWasmModuleRoot` need to remove the `0x` prefix.

7. Modify the docker-compose.yaml file to use the precompile image

```yml
  nitro:
-   image: ghcr.io/layr-labs/nitro-eigenda:eigenda-v3.1.2
+   image: nitro-node
    platform: linux/amd64
    ports:
      - "8449:8449"
    volumes:
      - "./config:/home/user/.arbitrum"
    command: --conf.file /home/user/.arbitrum/nodeConfig.json
```

8. Start the chain

```shell
EIGENDA_PROXY_PRIVATE_KEY="YourPrivateKey" ETH_RPC_URL="${YourRPC}" docker compose up -d
```
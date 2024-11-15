# BitDSM-examples

Main repository for the examples apps that could be built on top of BitDSM protocol for ethereum.

## CDP (Collateralized Debt Position) Contract

This example demonstrates how to build a simple CDP (Collateralized Debt Position) contract using BitDSM protocol. The CDP contract allows users to use their locked Bitcoin in bitcoin pods as collateral and borrow against it.

## Setup

To install the dependencies, follow below link to install foundry:

https://book.getfoundry.sh/getting-started/installation

and then run 

```bash
forge build
```

### 1. Deploy Oracle

The oracle system supports two types of price feeds:
1. Chainlink Price Feeds
2. Custom Price Feeds

#### Deploy Oracle Registry

First, deploy the Oracle Registry contract which manages all price feed registrations.
To Create a New Oracle refer to the below solidity code snippet:

```solidity
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);
        Oracle oracle = new Oracle();
```


### 2. Deploy CDP
Once the Oracle is deployed, you can deploy the CDP contract by referring to the below solidity code snippet:
#### Parameters:
- `_BITCOIN_POD_MANAGER`: The address of the Bitcoin Pod we want to register with.
- `OracleAddress`: The address of the Oracle we deployed on last step.

```solidity
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);
        CDP cdp = new CDP(_BITCOIN_POD_MANAGER, _ORACLE);
```


### 3. Register App

To register your application, you'll need to call the `registerApp` function provided in the `AppRegistry` contract with your app's address

#### Parameters:
- `appAddress`: The contract address of your application


To register your app, you need the address to `APPRegistry` contract and your App's (e.g., CDP) contract address

Then create a random salt and expiry
```solidity
        bytes32 salt = bytes32(uint256(1));
        uint256 expiry = block.timestamp + 1 days;
```

Calculate the digest hash for the app registration by calling the `calculateAppRegistrationDigestHash` function from the `AppRegistry` contract. Sign the digest hash and the broadcast the transaction.

```
        solidity vm.startBroadcast(deployerPrivateKey);

        try
            AppRegistry(_APP_REGISTRY).calculateAppRegistrationDigestHash(
                _APP_ADDRESS,
                _APP_REGISTRY,
                salt,
                expiry
            )
        returns (bytes32 digestHash) {
            console.log("Digest Hash:", vm.toString(digestHash));

            (uint8 v, bytes32 r, bytes32 s) = vm.sign(
                deployerPrivateKey,
                digestHash
            );
            bytes memory signature = abi.encodePacked(r, s, v);
            AppRegistry(_APP_REGISTRY).registerApp(
                _APP_ADDRESS,
                signature,
                salt,
                expiry
            );
```

### 4. Update App Metadata

Once the app is registered, you can update the app's metadata URI by calling the `updateAppMetadataURI` function from the `AppRegistry` contract. It is mandatory for the Metadata to be displayed on the BitDSM dashboard on Holesky Testnet.

First you need upload the metadata.json file to IPFS and get the URI. The format for creating the JSON is provided below. 

Metadata JSON file example:
```json
    {
          "name": "App Name",
          "website": "www.example.com",
          "description": "This is a simple example app demostrating the locking and unlocking functionality of BidcoinPodManager"
    }
```

then you can update the metadata URI by calling the `updateAppMetadataURI` function from the `AppRegistry` contract.

```solidity
        app = CDP(_APP_ADDRESS);
                app.updateAppMetadataURI(
                    "https://your_json_uri.json",
                    _APP_REGISTRY
                );
```


### 5. Deregister App

To deregister your app, you need to call the `deregisterApp` function from the `AppRegistry` contract.

#### Parameters:
- `appAddress`: The contract address of your application
- `AppRegistry`: The address of the BitDSMRegistry contract

```solidity
        AppRegistry(_APP_REGISTRY).deregisterApp(_APP_ADDRESS);
```

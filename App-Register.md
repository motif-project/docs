# Register your App with Motif   

This guide contains the instructions to register a deployed application with Motif. 

[Sample Registration Script](https://github.com/MotifFinance/motif-app-examples/blob/main/script/timelock/RegisterApp.s.sol)

## Setup

The code snippets below are created using Foundry framework. To install the dependencies, follow below link to install foundry:

https://book.getfoundry.sh/getting-started/installation

### 1. Register App

To register your application, you'll need to call the `registerApp` function provided in the `AppRegistry` contract with your app's address

#### Parameters:
- `appAddress`: The contract address of your application
- `signature` : The signature over the DigestHash
- `salt` : randomness for signature   
- `expiry` :  Time for signature is valid

To register your app, you need the address to `APPRegistry` contract and your App's (e.g., CDP) contract address

Then create a random salt and expiry
```solidity
        bytes32 salt = bytes32(uint256(1));
        uint256 expiry = block.timestamp + 1 days;
```

Calculate the digest hash for the app registration by calling the `calculateAppRegistrationDigestHash` function from the `AppRegistry` contract. Sign the digest hash and then broadcast the transaction.

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

### 2. Override isValidSignature
To successfully complete the application registration process, it is mandatory to implement the isValidSignature function from the IERC1271 interface in your App contract. The AppRegistry contract uses this implementation to validate signatures according to the custom logic defined in the App.

```solidity

        function isValidSignature(
                bytes32 _hash,
                bytes memory _signature
            ) external view override returns (bool)
```

### 3. Update App Metadata

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


### 4. Deregister App

To deregister your app, you need to call the `deregisterApp` function from the `AppRegistry` contract.

#### Parameters:
- `appAddress`: The contract address of your application
- `AppRegistry`: The address of the BitDSMRegistry contract

```solidity
        AppRegistry(_APP_REGISTRY).deregisterApp(_APP_ADDRESS);
```

# Step By Step Guide To Create A POD & Deposit/Withdraw From BitDSM

Created By: Ahmad
Stakeholders: penny 🐧
Last Edited By: Ahmad

A Bitcoin Pod is a smart contract  that enables secure custody of Bitcoin through a 2-of-2 multisig setup between a user and an operator. This guide explains how to create and interact with a new Pod.

## **Prerequisites**

1. An Ethereum wallet with some ETH for gas.
2. A registered operator's address from BitDSM Registry.
3. A Bitcoin multisig address created with the operator
4. The BitcoinPodManager contract address.
5. Addresses for all deployed contract are [here](https://github.com/BitDSM/BitDSM/tree/implement-v1-ecdsa?tab=readme-ov-file#existing-holesky-testnet-deployment).

## **Step-by-Step Guide**

### **1. Create the Bitcoin Multisig Address**

Before creating a Pod, you need to create a Bitcoin multisig address with your chosen operator. you can use the operator’s open API to generate the multisig address. Please remember that the operator  returns a BTC address and a hex representation of address bytes excluding the bech32 prefix. Please use the hex representation to register on the POD contractor as solidity does not support bech32 scheme.

```js
const response = await fetch('/eigen/get_address', {
    method: 'POST',
    body: JSON.stringify({
        pubKey: "your_bitcoin_public_key",
        podEthAddr: "your_ethereum_address"
    })
});
const multisigAddress = await response.json();
```

it returns the below JSON.

```json
	{
		"newAddress": newAddress,
		"addressHex": addressHex,
	}
```

### **2. Create POD**

**Important Considerations**

- Each Ethereum address can only create one pod.
- Pod ownership cannot be transferred.
- The pod creator becomes the pod owner.

To Create a POD user needs to sent a message to PodManager’s createPod function. Once the CreatePod transaction is sent and executed successfully, Pod Manager will emit a Pod Created event. User can subscribe to event if need be.

```js
const podManagerABI = [...]; // BitcoinPodManager ABI
const podManagerAddress = "0x..."; // BitcoinPodManager contract address

// Create contract instance
const podManager = new web3.eth.Contract(podManagerABI, podManagerAddress);

// Parameters
const operatorAddress = "0x..."; // Registered operator's Ethereum address
const btcAddress = "0x..."; // Bitcoin multisig address in hex format

// Create pod transaction
const tx = await podManager.methods.createPod(
    operatorAddress,
    btcAddress
).send({
    from: userAddress,
    gas: 500000 // Adjust as needed
});

// Get pod address from event
const podAddress = tx.events.PodCreated.returnValues.pod;
```

After the Pod is created user need to send btc to the multisig address. Please ensure that all the funds are sent via one single transaction. If the user use multiple transactions funds will be lost.

### 3. **Send Verify BTC Deposit Request**

After submitting the funds user need to send a verify BTC deposit message containing the BTC transaction ID, the amount of BTC sent and the POD address. Once this transaction is successful, operator will pick it up from there. Operator will verify the transaction and the amount and Call Confirm Deposit.

```js
async function prepareDepositVerification() {
    // Transaction parameters
    const params = {
        pod: "YOUR_POD_ADDRESS",
        // Bitcoin transaction ID in bytes32 format
        transactionId: ethers.utils.formatBytes32String("YOUR_BTC_TXID"),
        // Amount in satoshis
        amount: ethers.BigNumber.from("1000000") // 0.01 BTC = 1,000,000 sats
    };
    
    return params;
}

async function requestDepositVerification() {
    const podManager = await ethers.getContractAt(
        "BitcoinPodManager",
        "DEPLOYED_MANAGER_ADDRESS"
    );
    
    const params = await prepareDepositVerification();
    
    try {
        const tx = await podManager.verifyBitcoinDepositRequest(
            params.pod,
            params.transactionId,
            params.amount
        );
        
        const receipt = await tx.wait();
        
        // Get verification request event
        const verifyEvent = receipt.events.find(
            e => e.event === 'VerifyBitcoinDepositRequest'
        );
        
        console.log("Verification Request Submitted:", {
            pod: verifyEvent.args.pod,
            operator: verifyEvent.args.operator,
            request: {
                transactionId: verifyEvent.args.request.transactionId,
                amount: verifyEvent.args.request.amount.toString(),
                isPending: verifyEvent.args.request.isPending
            }
        });
        
    } catch (error) {
        console.error("Verification request failed:", error);
    }
}

```

### 3. **Create WithDraw Request:**

Important considerations:

- Make sure the the Pod is  not delegated. ensuring that there are nothing staked with an app
- A valid btc address is needed. again as mentioned above this has to be hex encoding of btc address bytes without the Bech32 prefix.
- it will always withdraw the complete amount deposited into the POD.
- User should have means to sign and broadcast Bitcoin PSBT.
- Only the Pod Owner can initiate this request

Once the pre requisites are met. user can create a Withdraw request by sending a withdrawBitcoinPSBTRequest to the PODManager.

```js
    it("Should create PSBT withdrawal request", async function() {
        // Create withdrawal address
        const withdrawAddress = ethers.utils.hexlify(
            Buffer.from("btc_withdrawal_address")
        );

        // Submit withdrawal request
        const tx = await podManager.withdrawBitcoinPSBTRequest(
            podAddress,
            withdrawAddress
        );
        const receipt = await tx.wait();

        // Verify event emission
        const event = receipt.events?.find(
            e => e.event === "BitcoinWithdrawalPSBTRequest"
        );
        expect(event?.args?.pod).to.equal(podAddress);
        expect(event?.args?.operator).to.equal(operator.address);
        expect(event?.args?.withdrawAddress).to.equal(withdrawAddress);

        // Verify withdrawal address storage
        expect(await podManager.getBitcoinWithdrawalAddress(podAddress))
            .to.equal(withdrawAddress);
    });
```

The operator will pick up the new withdraw request event and start the withdraw process. it will create a PSBT moving the funds to the provided withdraw address, Sign his part of the multisig and send the PSBT to the PodManager.

### 4. Retrieve Operator Signed PSBT

User Can retrieve the Signed PSBT from the Pod Manager using the getSignedBitcoinWithdrawTransaction Func.

```js
describe("Bitcoin Withdrawal Transaction", function() {
    let pod: BitcoinPod;
    let manager: SignerWithAddress;
    
    it("Should store and retrieve withdrawal transaction", async function() {
        // Create mock signed transaction
        const mockTx = ethers.utils.hexlify(
            Buffer.from("signed_bitcoin_transaction_data")
        );
        
        // Store transaction (must be called by manager)
        await pod.connect(manager).setSignedBitcoinWithdrawTransaction(mockTx);
        
        // Retrieve transaction
        const storedTx = await pod.getSignedBitcoinWithdrawTransaction();
        expect(storedTx).to.equal(mockTx);
    });

    it("Should handle empty transaction", async function() {
        const tx = await pod.getSignedBitcoinWithdrawTransaction();
        expect(tx).to.equal("0x"); // Empty when no transaction is stored
    });
});
```

Once User gets the PSBT. They should Sign it using their wallet and broadcast it on the btc chain.
Operator watches the tx ID on btc and once the transaction is confirmed, it will send the confirm withdrawal message to the Pod manager (closing the Pod)

user can also use below code. snippet to check the pod balance

```js
describe("BitcoinPod Balance", function() {
    let pod: BitcoinPod;
    
    it("Should return correct balance", async function() {
        const balance = await pod.getBitcoinBalance();
        console.log("Pod Balance:", balance.toString());
    });

    it("Should update balance after mint", async function() {
        // Initial balance
        const initialBalance = await pod.getBitcoinBalance();
        
        // Mint new tokens (must be called by manager)
        const mintAmount = ethers.utils.parseUnits("1", "bitcoin"); // 1 BTC
        await pod.connect(manager).mint(operator.address, mintAmount);
        
        // Check new balance
        const newBalance = await pod.getBitcoinBalance();
        expect(newBalance).to.equal(initialBalance.add(mintAmount));
    });
});
```

# Guide to Creating a Bitcoin Pod and Managing Bitcoin Deposits and Withdrawals

A Bitcoin Pod is a smart contract  that enables secure custody of Bitcoin through a 2-of-2 multisig setup between a user and an operator. This guide explains how to create and interact with a new Pod.

## **Prerequisites**

1. An Ethereum wallet with some ETH for gas.
2. A registered operator's address from Motif Stake Registry.
3. A Bitcoin multisig address created with the operator
4. The BitcoinPodManager contract address.
5. Addresses for all deployed contract are [here](https://github.com/BitDSM/BitDSM?tab=readme-ov-file#existing-holesky-testnet-deployment).

## **Step-by-Step Guide**

### **1. Creating a Bitcoin Multisig Address**

To create a Bitcoin Pod, the first step is to generate a Bitcoin multisig address in collaboration with your selected operator. You can use the operator’s open API to create this address.

The API will return two key pieces of information:
	- BTC Address: The standard Bitcoin address.
	- Hex Representation of Address Bytes: The raw byte format of the address, excluding the Bech32 prefix.

When registering the Bitcoin Pod on the smart contract, ensure you use the `hex representation` of the address, as `Solidity` does not support the `Bech32` format.

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

it returns the following JSON.

```json
	{
		"newAddress": newAddress,
		"addressHex": addressHex,
	}
```

---

### **2. Creating Bitcoin Pod**

**Key Considerations:**  
- Each Ethereum address can create only **one** Pod.  
- Pod ownership is **non-transferable**.  
- The **Pod creator** is automatically assigned as the **Pod owner**.  

To create a Pod, the user must call the `createPod` function of the **PodManager** contract. Once the `createPod` transaction is successfully executed, the **PodManager** will emit a **PodCreated** event. Users can subscribe to this event if they wish to track Pod creation in real-time.  
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
---

### **3. Depositing BTC into the Pod**

Once the Pod is created, the user must deposit BTC to the **multisig address**.  
**Important:** Ensure that all funds are sent in **one single transaction**. If multiple transactions are used, the funds may be **lost**.

---

### **4. Verify BTC Deposit**

After sending the BTC, the user must submit a **VerifyBTCDeposit** request. This request includes:  
- The **BTC transaction ID**  
- The **amount of BTC sent**  
- The **Pod address**  

Once this transaction is successfully submitted in a Bitcoin block, the **operator** will verify the deposit and call **ConfirmDeposit** function.

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
---

### **5. Creating a Withdrawal Request**

**Key Considerations:**  
- Ensure the **Pod is not delegated** (i.e., no active stakes with an app).  
- Provide a **valid BTC address**—this must be the **hex-encoded address bytes** without the **Bech32 prefix**.  
- Withdrawals will always transfer the **entire balance** from the Pod.  
- Users must have the means to **sign and broadcast a Bitcoin PSBT**.  
- Only the **Pod Owner** can initiate a withdrawal.  

To create a withdrawal request, the user must send a **withdrawBitcoinPSBTRequest** to the **PodManager**.  

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

The **operator** will detect the new withdrawal request, generate a **PSBT** (Partially Signed Bitcoin Transaction) moving the funds to the withdrawal address, sign their part of the multisig, and send the signed PSBT to the **PodManager**.

---

### **6. Retrieve the Operator-Signed PSBT**

Users can retrieve the **Signed PSBT** by calling the **getSignedBitcoinWithdrawTransaction** function from the **PodManager**.  

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

Once the user obtains the PSBT:  
1. **Sign** it using their wallet.  
2. **Broadcast** the signed transaction to the Bitcoin network.  

The operator will monitor the Bitcoin blockchain for the transaction confirmation. Once confirmed, the operator will send a **Confirm Withdrawal** message to the **PodManager**, officially **closing the Pod**.

---

### **Checking Pod Balance**

Users can also check their Pod's balance using the following code snippet:

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

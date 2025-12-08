# EntropyHandleLifecycle

Understanding handles and symbolic execution with EntropyOracle

## 🚀 Standard workflow
- Install (first run): `npm install --legacy-peer-deps`
- Compile: `npx hardhat compile`
- Test (local FHE + local oracle/chaos engine auto-deployed): `npx hardhat test`
- Deploy (frontend Deploy button): constructor arg is fixed to EntropyOracle `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`
- Verify: `npx hardhat verify --network sepolia <contractAddress> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

## 📋 Overview

This example demonstrates **handles** concepts in FHEVM with **EntropyOracle integration**:
- What are handles?
- How handles are generated
- Symbolic execution in FHEVM
- Handle lifecycle and permissions
- EntropyOracle handle management

## 🎯 What This Example Teaches

This tutorial will teach you:

1. **What are handles** and how they work
2. **How handles are generated** by FHEVM SDK and EntropyOracle
3. **Symbolic execution** in FHEVM
4. **Handle lifecycle** from creation to use
5. **How to use handles** in FHE operations
6. **EntropyOracle handle management**

## 💡 Why This Matters

Handles are fundamental to FHEVM:
- **Understanding handles** helps debug issues
- **Handles represent encrypted values** symbolically
- **Actual decryption happens off-chain** using FHEVM SDK
- **EntropyOracle returns handles** for encrypted entropy
- **All FHE operations work on handles**, not actual encrypted data

## 🔍 How It Works

### Contract Structure

The contract has four main components:

1. **Store Handle**: Store encrypted value handle
2. **Use Handle**: Use stored handle in FHE operations
3. **Request Entropy**: Request entropy handle from EntropyOracle
4. **Use Entropy Handle**: Use entropy handle in FHE operations

### Step-by-Step Code Explanation

#### 1. Constructor

```solidity
constructor(address _entropyOracle) {
    require(_entropyOracle != address(0), "Invalid oracle address");
    entropyOracle = IEntropyOracle(_entropyOracle);
}
```

**What it does:**
- Takes EntropyOracle address as parameter
- Validates the address is not zero
- Stores the oracle interface

**Why it matters:**
- Must use the correct oracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

#### 2. Store Handle

```solidity
function storeHandle(
    externalEuint64 encryptedInput,
    bytes calldata inputProof
) external {
    require(!initialized, "Already initialized");
    
    // Step 1: Convert external handle to internal handle
    // The handle is a reference, not the actual encrypted data
    euint64 internalHandle = FHE.fromExternal(encryptedInput, inputProof);
    
    // Step 2: Grant contract permission to use this handle
    // This is required for FHE operations
    FHE.allowThis(internalHandle);
    
    // Step 3: Store handle in contract state
    // The handle is stored, not the actual encrypted value
    encryptedValue = internalHandle;
    initialized = true;
}
```

**What it does:**
- Accepts encrypted input (which contains a handle)
- Validates encrypted input using input proof
- Converts external handle to internal handle
- Grants contract permission to use handle
- Stores handle in contract state

**Key concepts:**
- **Handle**: Reference to encrypted value, not actual encrypted data
- **External handle**: Handle from outside the contract (frontend)
- **Internal handle**: Handle used inside the contract
- **Handle storage**: Only the handle is stored, not encrypted data

**Why it's needed:**
- Contract needs to reference encrypted values
- Handles enable symbolic execution
- Actual decryption happens off-chain

#### 3. Use Handle in FHE Operation

```solidity
function useHandle() external returns (euint64) {
    require(initialized, "Not initialized");
    
    // Use stored handle in FHE operation
    // The operation is performed symbolically
    euint64 one = FHE.asEuint64(1);
    euint64 result = FHE.add(encryptedValue, one);
    
    // Result is also a handle (not decrypted value)
    emit HandleUsed();
    
    return result;
}
```

**What it does:**
- Uses stored handle in FHE operation
- Performs addition symbolically
- Returns result handle (not decrypted value)

**Key concepts:**
- **Symbolic execution**: Operations work on handles, not actual values
- **Result handle**: Result is also a handle
- **Off-chain decryption**: Actual decryption happens off-chain using FHEVM SDK

**Why symbolic:**
- FHE operations are performed symbolically
- Actual computation happens off-chain
- Handles track the computation symbolically

#### 4. Request Entropy

```solidity
function requestEntropy(bytes32 tag) external payable returns (uint256 requestId) {
    require(msg.value >= entropyOracle.getFee(), "Insufficient fee");
    
    requestId = entropyOracle.requestEntropy{value: msg.value}(tag);
    entropyRequests[requestId] = true;
    
    return requestId;
}
```

**What it does:**
- Validates fee payment
- Requests entropy from EntropyOracle
- Stores request ID
- Returns request ID (not handle yet)

**Key concepts:**
- **Request ID**: Identifier for the entropy request
- **Not a handle**: Request ID is not the entropy handle
- **Handle comes later**: Entropy handle retrieved after fulfillment

#### 5. Use Entropy Handle

```solidity
function useEntropyHandle(uint256 requestId) external returns (euint64) {
    require(entropyOracle.isRequestFulfilled(requestId), "Entropy not ready");
    
    // Get entropy handle from oracle
    euint64 entropyHandle = entropyOracle.getEncryptedEntropy(requestId);
    FHE.allowThis(entropyHandle);
    
    // Use entropy handle in FHE operation
    euint64 one = FHE.asEuint64(1);
    FHE.allowThis(one);
    euint64 result = FHE.add(encryptedValue, entropyHandle);
    FHE.allowThis(result);
    
    return result;
}
```

**What it does:**
- Validates request ID and fulfillment status
- Gets entropy handle from EntropyOracle
- **Grants permission** to use entropy handle (CRITICAL!)
- Uses entropy handle in FHE operation
- Returns result handle

**Key concepts:**
- **Entropy handle**: Handle for encrypted entropy
- **Handle operations**: Entropy handle used like any other handle
- **Result handle**: Result is also a handle

**Why handle-based:**
- EntropyOracle returns handles, not decrypted values
- Handles can be used in FHE operations
- Actual entropy value remains encrypted

## 🧪 Step-by-Step Testing

### Prerequisites

1. **Install dependencies:**
   ```bash
   npm install --legacy-peer-deps
   ```

2. **Compile contracts:**
   ```bash
   npx hardhat compile
   ```

### Running Tests

```bash
npx hardhat test
```

### What Happens in Tests

1. **Fixture Setup** (`deployContractFixture`):
   - Deploys FHEChaosEngine, EntropyOracle, and EntropyHandleLifecycle
   - Returns all contract instances

2. **Test: Store Handle**
   ```typescript
   it("Should store handle", async function () {
     const input = hre.fhevm.createEncryptedInput(contractAddress, owner.address);
     input.add64(42);
     const encryptedInput = await input.encrypt();
     
     // encryptedInput.handles[0] is the handle
     await contract.storeHandle(encryptedInput.handles[0], encryptedInput.inputProof);
     
     expect(await contract.isInitialized()).to.be.true;
   });
   ```
   - Creates encrypted input (generates handle)
   - Encrypts using FHEVM SDK
   - Calls `storeHandle()` with handle and proof
   - Verifies handle is stored

3. **Test: Use Handle**
   ```typescript
   it("Should use handle in FHE operation", async function () {
     // ... store handle code ...
     const result = await contract.useHandle();
     expect(result).to.not.be.undefined;
   });
   ```
   - Uses stored handle in FHE operation
   - Result is also a handle
   - Operation performed symbolically

4. **Test: Request Entropy**
   ```typescript
   it("Should request entropy", async function () {
     const tag = hre.ethers.id("test-handle");
     const fee = await oracle.getFee();
     const requestId = await contract.requestEntropy(tag, { value: fee });
     expect(requestId).to.not.be.undefined;
   });
   ```
   - Requests entropy with unique tag
   - Pays required fee
   - Returns request ID (not handle)

5. **Test: Use Entropy Handle**
   ```typescript
   it("Should use entropy handle", async function () {
     // ... request entropy code ...
     await waitForEntropy(requestId);
     const result = await contract.useEntropyHandle(requestId);
     expect(result).to.not.be.undefined;
   });
   ```
   - Gets entropy handle from oracle
   - Uses entropy handle in FHE operation
   - Result is also a handle

### Expected Test Output

```
  EntropyHandleLifecycle
    Deployment
      ✓ Should deploy successfully
      ✓ Should have EntropyOracle address set
    Handle Storage
      ✓ Should store handle
    Handle Usage
      ✓ Should use handle in FHE operation
    Entropy Handles
      ✓ Should request entropy
      ✓ Should use entropy handle in FHE operation

  6 passing
```

**Note:** All values appear as handles in test output. Decrypt off-chain using FHEVM SDK to see actual values.

## 🚀 Step-by-Step Deployment

### Option 1: Frontend (Recommended)

1. Navigate to [Examples page](/examples)
2. Find "EntropyHandleLifecycle" in Tutorial Examples
3. Click **"Deploy"** button
4. Approve transaction in wallet
5. Wait for deployment confirmation
6. Copy deployed contract address

### Option 2: CLI

1. **Create deploy script** (`scripts/deploy.ts`):
   ```typescript
   import hre from "hardhat";

   async function main() {
     const ENTROPY_ORACLE_ADDRESS = "0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361";
     
     const ContractFactory = await hre.ethers.getContractFactory("EntropyHandleLifecycle");
     const contract = await ContractFactory.deploy(ENTROPY_ORACLE_ADDRESS);
     await contract.waitForDeployment();
     
     const address = await contract.getAddress();
     console.log("EntropyHandleLifecycle deployed to:", address);
   }

   main().catch((error) => {
     console.error(error);
     process.exitCode = 1;
   });
   ```

2. **Deploy:**
   ```bash
   npx hardhat run scripts/deploy.ts --network sepolia
   ```

## ✅ Step-by-Step Verification

### Option 1: Frontend

1. After deployment, click **"Verify"** button on Examples page
2. Wait for verification confirmation
3. View verified contract on Etherscan

### Option 2: CLI

```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361
```

**Important:** Constructor argument must be the EntropyOracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

## 📊 Expected Outputs

### After Store Handle

- `isInitialized()` returns `true`
- Handle stored in contract state
- Handle can be used in FHE operations

### After Use Handle

- FHE operation completes successfully
- Result is also a handle
- Operation performed symbolically
- `HandleUsed` event emitted

### After Request Entropy

- Request ID returned (not handle)
- Entropy request stored
- `EntropyRequested` event emitted

### After Use Entropy Handle

- Entropy handle retrieved from oracle
- Entropy handle used in FHE operation
- Result is also a handle
- `EntropyHandleUsed` and `HandleUsed` events emitted

## ⚠️ Common Errors & Solutions

### Error: `Handle not found`

**Cause:** Trying to use handle that doesn't exist or wasn't stored.

**Solution:** Ensure handle is stored before using it. Check initialization status.

---

### Error: `SenderNotAllowed()`

**Cause:** Missing `FHE.allowThis()` call on handle.

**Example:**
```solidity
euint64 handle = FHE.fromExternal(encryptedInput, inputProof);
// Missing: FHE.allowThis(handle);
euint64 result = FHE.add(handle, otherHandle); // ❌ FAILS!
```

**Solution:**
```solidity
euint64 handle = FHE.fromExternal(encryptedInput, inputProof);
FHE.allowThis(handle); // ✅ Required!
euint64 result = FHE.add(handle, otherHandle); // ✅ WORKS!
```

**Prevention:** Always call `FHE.allowThis()` on handles before using them in FHE operations.

---

### Error: `Entropy not ready`

**Cause:** Calling `useEntropyHandle()` before entropy is fulfilled.

**Solution:** Always check `isRequestFulfilled()` before using entropy.

---

### Error: `Insufficient fee`

**Cause:** Not sending enough ETH when requesting entropy.

**Solution:** Always send exactly 0.00001 ETH:
```typescript
const fee = await contract.entropyOracle.getFee();
await contract.requestEntropy(tag, { value: fee });
```

---

### Error: Verification failed - Constructor arguments mismatch

**Cause:** Wrong constructor argument used during verification.

**Solution:** Always use the EntropyOracle address:
```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361
```

## 🔗 Related Examples

- [EntropyEncryption](../encryption-encryptsingle/) - Encrypting values (generates handles)
- [EntropyCounter](../basic-simplecounter/) - Using handles in operations
- [Category: handles](../)

## 📚 Additional Resources

- [Full Tutorial Track Documentation](../../../frontend/src/pages/Docs.tsx) - Complete educational guide
- [Zama FHEVM Documentation](https://docs.zama.org/) - Official FHEVM docs
- [GitHub Repository](https://github.com/zacnider/entrofhe/tree/main/examples/handles-handlelifecycle) - Source code

## 📝 License

BSD-3-Clause-Clear

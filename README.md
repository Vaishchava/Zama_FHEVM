# 🔒 Confidential ERC-20 Token using Zama's FHEVM

Create private ERC-20 tokens where balances and transfers are encrypted using Fully Homomorphic Encryption (FHE) from @zama_fhe. Perfect for building confidential DeFi apps.  

---

## 🧰 Step-by-Step Guide (Linux Only)

### 1️⃣ Open Your VPS or Local Terminal

```bash
ssh username@your_ip  # Skip if working locally
```
### 2️⃣ Install Required Packages
```bash
sudo apt update && sudo apt install -y python3 python3-venv python3-pip curl wget screen git lsof
```
### 3️⃣ Install Node.js, npm, and yarn
```bash
sudo apt install -y nodejs npm
npm install -g yarn
```
### 4️⃣ Check Versions
```bash
python3 --version
node -v
npm -v
yarn -v
```
### 5️⃣ Initialize Project with Hardhat
```bash
mkdir confidential-fhe-token && cd confidential-fhe-token
npm init -y
npm install --save-dev hardhat
npx hardhat  # Choose "Create a basic sample project"
```
### 6️⃣ Install Zama’s FHEVM Solidity Library
```bash
npm install @zama.ai/fhevm-solidity
```
### 7️⃣ Create the Contract File
📁 Inside contracts/ConfidentialERC20.sol add:
```bash
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

import "@zama.ai/fhevm-solidity/lib/TFHE.sol";

contract ConfidentialERC20 {
    euint32 public totalSupply;
    mapping(address => euint32) private balances;

 // You typically have a constructor here to initialize totalSupply,
 // and potentially mint initial tokens to the deployer.
    constructor(euint32 _initialSupply) {
        totalSupply = _initialSupply;
        balances[msg.sender] = _initialSupply;
        TFHE.allow(balances[msg.sender], msg.sender);
    }

 // Inside your ConfidentialERC20 contract
    function transfer(address to, einput encryptedAmount, bytes calldata inputProof) public {
 // 1. Validate and convert the encrypted input amount
        euint32 amount = TFHE.asEuint32(encryptedAmount, inputProof);
 // 2. Check if sender has enough balance (encrypted comparison)
        ebool canTransfer = TFHE.le(amount, balances[msg.sender]);
 // 3. Revert if not enough balance (encrypted condition)
        TFHE.revertIf(TFHE.not(canTransfer), "Insufficient balance");

 // 4. Update balances conditionally using TFHE.cmux (Conditional Multiplexer)
 // Only apply add/sub if canTransfer is true
        balances[to] = TFHE.cmux(canTransfer, TFHE.add(balances[to], amount), balances[to]);
        balances[msg.sender] = TFHE.cmux(canTransfer, TFHE.sub(balances[msg.sender], amount), balances[msg.sender]);

// Immediately after the balance updates in the transfer function (Post 5)
        TFHE.allow(balances[to], to);   // Recipient can decrypt their new balance
        TFHE.allow(balances[msg.sender], msg.sender);  // Sender can decrypt their updated balance
    }
}
```
8️⃣ Configure Zama Testnet in hardhat.config.js
Add Zama’s RPC and chain ID:

```bash 
networks: {
  zama_testnet: {
    url: "https://testnet.zama.ai", // Replace with actual RPC from docs
    accounts: ["YOUR_PRIVATE_KEY"]
  }
}
```
Get RPC details from: https://docs.zama.ai/protocol/development
### 9️⃣ Write Deployment Script
📁 Inside scripts/deploy.js:
```bash
async function main() {
  const [deployer] = await ethers.getSigners();
  const Contract = await ethers.getContractFactory("ConfidentialERC20");

  // Replace with encrypted initial value if needed
  const token = await Contract.deploy(1000);
  console.log("Deployed to:", token.address);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
### 🔟 Deploy Contract
```bash 
npx hardhat run scripts/deploy.js --network zama_testnet
```
### 🧪 Get Test Tokens
Use Zama’s faucet:
👉 https://docs.zama.ai/protocol

### 📚 Learn More

Zama Docs: https://docs.zama.ai

FHEVM Solidity: https://github.com/zama-ai/fhevm-solidity


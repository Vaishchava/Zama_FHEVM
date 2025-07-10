# Step 1: Install dependencies
sudo apt update && sudo apt install -y python3 python3-venv python3-pip curl wget screen git lsof nodejs npm
npm install -g yarn

# Step 2: Create project folder
mkdir confidential-fhe-token && cd confidential-fhe-token

# Step 3: Initialize project and install Hardhat
npm init -y
npm install --save-dev hardhat
npx hardhat  # Follow prompt: "Create a basic sample project" → Yes to everything

# Step 4: Install Zama’s FHEVM Solidity library
npm install @zama.ai/fhevm-solidity

# Step 5: Create the ConfidentialERC20 contract file
cat > contracts/ConfidentialERC20.sol <<'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

import "@zama.ai/fhevm-solidity/lib/TFHE.sol";

contract ConfidentialERC20 {
    euint32 public totalSupply;
    mapping(address => euint32) private balances;

    constructor(euint32 _initialSupply) {
        totalSupply = _initialSupply;
        balances[msg.sender] = _initialSupply;
        TFHE.allow(balances[msg.sender], msg.sender);
    }

    function transfer(address to, einput encryptedAmount, bytes calldata inputProof) public {
        euint32 amount = TFHE.asEuint32(encryptedAmount, inputProof);
        ebool canTransfer = TFHE.le(amount, balances[msg.sender]);
        TFHE.revertIf(TFHE.not(canTransfer), "Insufficient balance");

        balances[to] = TFHE.cmux(canTransfer, TFHE.add(balances[to], amount), balances[to]);
        balances[msg.sender] = TFHE.cmux(canTransfer, TFHE.sub(balances[msg.sender], amount), balances[msg.sender]);

        TFHE.allow(balances[to], to);
        TFHE.allow(balances[msg.sender], msg.sender);
    }
}
EOF

# Step 6: Create a deployment script
mkdir -p scripts
cat > scripts/deploy.js <<'EOF'
async function main() {
  const [deployer] = await ethers.getSigners();
  const Contract = await ethers.getContractFactory("ConfidentialERC20");

  // Replace this value with a properly encrypted input for realistic use
  const token = await Contract.deploy(1000);
  console.log("Contract deployed to:", token.address);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
EOF

# Step 7: (Manual) Edit hardhat.config.js to add Zama testnet config
echo "🛠️ Now open 'hardhat.config.js' and add your Zama testnet RPC + private key."

# Step 8: Run deployment (after configuring the network)
# npx hardhat run scripts/deploy.js --network zama_testnet

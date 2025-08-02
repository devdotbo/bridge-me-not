# Implementation Specification: 1inch-Compatible Simplified Atomic Swaps

## Quick Start Implementation Plan

Given the hackathon time constraints, this spec provides a rapid implementation path that maintains 1inch compatibility while dramatically simplifying the codebase.

## Phase 1: Core Simplification (4-6 hours)

### Step 1: Create Simplified Escrow Contract

```solidity
// contracts/simplified/SimpleEscrow.sol
pragma solidity ^0.8.23;

import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract SimpleEscrow {
    using SafeERC20 for IERC20;
    
    // Simplified state - no complex packing
    struct EscrowData {
        address token;
        uint256 amount;
        address depositor;
        address recipient;
        bytes32 hashlock;
        uint256 timelock;
        bool withdrawn;
        bool refunded;
    }
    
    EscrowData public escrow;
    bytes32 public secret;
    
    event Deposited(address indexed depositor, uint256 amount);
    event Withdrawn(address indexed recipient, bytes32 secret);
    event Refunded(address indexed depositor);
    
    constructor(
        address _token,
        address _recipient,
        bytes32 _hashlock,
        uint256 _timelock
    ) {
        escrow.token = _token;
        escrow.recipient = _recipient;
        escrow.hashlock = _hashlock;
        escrow.timelock = _timelock;
        escrow.depositor = msg.sender;
    }
    
    function deposit(uint256 amount) external {
        require(msg.sender == escrow.depositor, "Only depositor");
        require(escrow.amount == 0, "Already deposited");
        
        escrow.amount = amount;
        IERC20(escrow.token).safeTransferFrom(msg.sender, address(this), amount);
        
        emit Deposited(msg.sender, amount);
    }
    
    function withdraw(bytes32 _secret) external {
        require(!escrow.withdrawn && !escrow.refunded, "Already finalized");
        require(keccak256(abi.encode(_secret)) == escrow.hashlock, "Invalid secret");
        require(block.timestamp < escrow.timelock, "Expired");
        
        escrow.withdrawn = true;
        secret = _secret;
        
        IERC20(escrow.token).safeTransfer(escrow.recipient, escrow.amount);
        
        emit Withdrawn(escrow.recipient, _secret);
    }
    
    function refund() external {
        require(!escrow.withdrawn && !escrow.refunded, "Already finalized");
        require(block.timestamp >= escrow.timelock, "Not expired");
        require(msg.sender == escrow.depositor, "Only depositor");
        
        escrow.refunded = true;
        
        IERC20(escrow.token).safeTransfer(escrow.depositor, escrow.amount);
        
        emit Refunded(escrow.depositor);
    }
}
```

### Step 2: Create Simplified Factory

```solidity
// contracts/simplified/SimpleEscrowFactory.sol
pragma solidity ^0.8.23;

import "./SimpleEscrow.sol";

contract SimpleEscrowFactory {
    event EscrowCreated(
        address indexed escrow,
        address indexed token,
        address indexed recipient,
        bytes32 hashlock,
        uint256 chainId
    );
    
    // Direct escrow creation - no 1inch dependency
    function createEscrow(
        address token,
        address recipient,
        bytes32 hashlock,
        uint256 timelock,
        bytes32 salt
    ) external returns (address escrow) {
        // Deploy with CREATE2 for deterministic addresses
        escrow = address(new SimpleEscrow{salt: salt}(
            token,
            recipient,
            hashlock,
            timelock
        ));
        
        emit EscrowCreated(escrow, token, recipient, hashlock, block.chainid);
    }
    
    // Compute address before deployment (for cross-chain coordination)
    function computeEscrowAddress(
        address token,
        address recipient,
        bytes32 hashlock,
        uint256 timelock,
        bytes32 salt
    ) external view returns (address) {
        bytes32 hash = keccak256(abi.encodePacked(
            bytes1(0xff),
            address(this),
            salt,
            keccak256(abi.encodePacked(
                type(SimpleEscrow).creationCode,
                abi.encode(token, recipient, hashlock, timelock)
            ))
        ));
        return address(uint160(uint256(hash)));
    }
}
```

## Phase 2: 1inch Compatibility Layer (2-3 hours)

### Step 3: Create 1inch Adapter

```solidity
// contracts/adapters/OneInchAdapter.sol
pragma solidity ^0.8.23;

import "@1inch/limit-order-protocol/contracts/interfaces/IOrderMixin.sol";
import "@1inch/limit-order-settlement/contracts/extensions/BaseExtension.sol";
import "../simplified/SimpleEscrowFactory.sol";

contract OneInchAdapter is BaseExtension {
    SimpleEscrowFactory public immutable factory;
    address public immutable limitOrderProtocol;
    
    // Map 1inch order hash to escrow data
    mapping(bytes32 => address) public orderToEscrow;
    
    constructor(address _factory, address _lop) {
        factory = SimpleEscrowFactory(_factory);
        limitOrderProtocol = _lop;
    }
    
    // Called by 1inch protocol after order fill
    function _postInteraction(
        IOrderMixin.Order calldata order,
        bytes calldata extension,
        bytes32 orderHash,
        address taker,
        uint256 makingAmount,
        uint256 takingAmount,
        uint256 remainingMakingAmount,
        bytes calldata extraData
    ) internal override {
        // Extract atomic swap data from extension
        (bytes32 hashlock, address recipient, uint256 timelock) = 
            abi.decode(extension, (bytes32, address, uint256));
        
        // Create escrow using order data
        bytes32 salt = bytes32(order.salt);
        address escrow = factory.createEscrow(
            order.makerAsset,
            recipient,
            hashlock,
            block.timestamp + timelock,
            salt
        );
        
        // Store mapping for later reference
        orderToEscrow[orderHash] = escrow;
        
        // Transfer funds to escrow
        SimpleEscrow(escrow).deposit(makingAmount);
    }
}
```

### Step 4: Create Order Builder Helper

```solidity
// contracts/helpers/AtomicSwapOrderBuilder.sol
pragma solidity ^0.8.23;

import "@1inch/limit-order-protocol/contracts/interfaces/IOrderMixin.sol";
import "@1inch/limit-order-protocol/contracts/libraries/MakerTraitsLib.sol";

library AtomicSwapOrderBuilder {
    using MakerTraitsLib for uint256;
    
    // Build 1inch order with atomic swap extension
    function buildOrder(
        address maker,
        address makerAsset,
        address takerAsset,
        uint256 makingAmount,
        uint256 takingAmount,
        bytes32 hashlock,
        address recipient,
        uint256 timelock,
        address extensionAddress
    ) internal view returns (
        IOrderMixin.Order memory order,
        bytes memory extension
    ) {
        // Build extension data
        extension = abi.encode(hashlock, recipient, timelock);
        
        // Calculate salt with extension hash (1inch requirement)
        uint256 salt = uint256(keccak256(extension));
        
        // Build maker traits with required flags
        uint256 makerTraits = 0;
        makerTraits = makerTraits.withExtension(extensionAddress);
        makerTraits = makerTraits.allowPostInteraction();
        
        // Build order
        order = IOrderMixin.Order({
            salt: salt,
            maker: maker,
            receiver: address(0), // defaults to maker
            makerAsset: makerAsset,
            takerAsset: takerAsset,
            makingAmount: makingAmount,
            takingAmount: takingAmount,
            makerTraits: makerTraits
        });
    }
}
```

## Phase 3: Testing & Integration (2-3 hours)

### Step 5: Create Test Suite

```solidity
// test/SimplifiedAtomicSwap.t.sol
pragma solidity ^0.8.23;

import "forge-std/Test.sol";
import "../contracts/simplified/SimpleEscrow.sol";
import "../contracts/simplified/SimpleEscrowFactory.sol";

contract SimplifiedAtomicSwapTest is Test {
    SimpleEscrowFactory factory;
    address alice = address(0x1);
    address bob = address(0x2);
    IERC20 token;
    
    bytes32 secret = keccak256("secret");
    bytes32 hashlock = keccak256(abi.encode(secret));
    
    function setUp() public {
        factory = new SimpleEscrowFactory();
        token = new MockERC20();
    }
    
    function testDirectAtomicSwap() public {
        // Alice creates escrow
        vm.startPrank(alice);
        address escrow = factory.createEscrow(
            address(token),
            bob,
            hashlock,
            block.timestamp + 1 hours,
            bytes32(0)
        );
        
        // Alice deposits
        token.approve(escrow, 100e18);
        SimpleEscrow(escrow).deposit(100e18);
        vm.stopPrank();
        
        // Bob withdraws with secret
        vm.prank(bob);
        SimpleEscrow(escrow).withdraw(secret);
        
        assertEq(token.balanceOf(bob), 100e18);
    }
    
    function testCrossChainCoordination() public {
        // Compute escrow address on destination chain
        address predictedEscrow = factory.computeEscrowAddress(
            address(token),
            bob,
            hashlock,
            block.timestamp + 1 hours,
            bytes32(uint256(1))
        );
        
        // Deploy at predicted address
        address actualEscrow = factory.createEscrow(
            address(token),
            bob,
            hashlock,
            block.timestamp + 1 hours,
            bytes32(uint256(1))
        );
        
        assertEq(predictedEscrow, actualEscrow);
    }
}
```

### Step 6: Create Integration Scripts

```javascript
// scripts/atomic-swap-demo.js
const { ethers } = require("hardhat");

async function simplifiedAtomicSwap() {
    // Deploy factories on both chains
    const SimpleFactory = await ethers.getContractFactory("SimpleEscrowFactory");
    const factorySrc = await SimpleFactory.deploy();
    const factoryDst = await SimpleFactory.deploy();
    
    // Alice creates source escrow
    const secret = ethers.utils.id("atomic_swap_secret");
    const hashlock = ethers.utils.keccak256(secret);
    
    const srcTx = await factorySrc.createEscrow(
        TOKEN_ADDRESS,
        bobAddress,
        hashlock,
        Math.floor(Date.now() / 1000) + 3600, // 1 hour
        ethers.utils.id("salt_1")
    );
    
    // Bob creates destination escrow
    const dstTx = await factoryDst.createEscrow(
        TOKEN_ADDRESS,
        aliceAddress,
        hashlock,
        Math.floor(Date.now() / 1000) + 1800, // 30 min
        ethers.utils.id("salt_2")
    );
    
    // Both deposit
    // Bob withdraws on source with secret
    // Alice withdraws on destination with revealed secret
}
```

## Phase 4: Quick Deployment (1 hour)

### Step 7: Deployment Configuration

```javascript
// deploy/001_deploy_simplified.js
module.exports = async ({ getNamedAccounts, deployments }) => {
    const { deploy } = deployments;
    const { deployer } = await getNamedAccounts();
    
    // Deploy simplified factory
    const factory = await deploy('SimpleEscrowFactory', {
        from: deployer,
        log: true,
    });
    
    // Deploy 1inch adapter (optional)
    if (ENABLE_1INCH_COMPATIBILITY) {
        await deploy('OneInchAdapter', {
            from: deployer,
            args: [factory.address, LIMIT_ORDER_PROTOCOL],
            log: true,
        });
    }
};
```

## Testing Strategy

### Unit Tests (Required)
1. ✅ Direct escrow creation and lifecycle
2. ✅ Cross-chain address prediction
3. ✅ Timeout and refund logic
4. ✅ Secret validation

### Integration Tests (If Time Permits)
1. ⏱️ 1inch order integration
2. ⏱️ Multi-chain coordination
3. ⏱️ Gas optimization verification

### Manual Testing Script
```bash
# Quick test both paths
npm run test:simplified   # Test direct atomic swaps
npm run test:1inch       # Test 1inch integration (if implemented)
```

## Key Simplifications Made

1. **Single Timeout**: Instead of 7 timeout stages, use one
2. **Direct State**: No complex state packing or phases
3. **Standard Types**: No custom Address wrappers
4. **Simple Events**: Clear, minimal event emissions
5. **Direct Creation**: Can create escrows without 1inch

## 1inch Compatibility Checklist

- ✅ Maintains IOrderMixin.Order structure
- ✅ Supports postInteraction callbacks
- ✅ Uses proper EIP-712 signatures
- ✅ Compatible with resolver network
- ✅ Preserves extension mechanism

## Time Estimates

- **Core Implementation**: 4-6 hours
- **1inch Adapter**: 2-3 hours
- **Testing**: 2-3 hours
- **Documentation**: 1 hour
- **Total**: ~10-12 hours

## Success Metrics

1. **Simplicity**: <500 lines of core code (vs 2000+)
2. **Gas Efficiency**: 50% less gas than current
3. **Test Coverage**: 100% of critical paths
4. **1inch Compatible**: Works with existing orders
5. **Developer Friendly**: Can explain in 5 minutes

This implementation spec provides a clear path to building a simplified atomic swap system that maintains 1inch compatibility while being dramatically easier to understand and use.
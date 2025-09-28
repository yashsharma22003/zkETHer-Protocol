## zKETHer 
## Overview

zkEther is a privacy-preserving deposit & withdrawal pool built using zero-knowledge proofs (ZKPs) and incremental Merkle trees.

It allows users to:

1. Deposit supported ERC20 tokens (DAI, USDC, USDT, etc.) into the pool.

2. Receive wrapped compliant zTokens (via Minter).

3. Withdraw tokens later by proving ownership via a ZK proof, without revealing their identity.

The system integrates:

1. Incremental Merkle Tree → privacy-preserving deposit tracking.

2. Verifier → validates zkSNARK/Honk proofs.

3. Minter → wraps/unlocks ERC20s into zTokens.

4. Factory → deploys pools and zTokens.

5. Mock Tokens → DAI, USDC, USDT for testing.

## Architecture
zPool.sol
The zPool contract is the core of the private transaction system, acting as a mixer. It allows users to deposit and withdraw tokens while maintaining privacy using zero-knowledge proofs. The contract inherits from IncrementalMerkleTree and ReentrancyGuard, ensuring an append-only data structure for commitments and preventing reentrancy attacks.

## State Variables
verifier: An instance of IVerifier used to verify zero-knowledge proofs.

minterToken: An IERC20 token representing the wrapped token that is minted on deposit and burned on withdrawal.

minter: An instance of the Minter contract responsible for handling the minting and burning of wrapped tokens.

DENOMINATION: The fixed amount of tokens that can be deposited or withdrawn in a single transaction.

ptoken: The underlying IERC20 token for the pool.

owner: The address that owns the contract and has privileged access to certain functions.

depositFee: A fixed fee in Ether (0.001 ether) required for each deposit.

s_nullifierHashes: A mapping to track spent nullifier hashes, preventing double-spending.

s_commitments: A mapping to track deposited commitments, preventing a commitment from being deposited more than once.

## Functions
constructor(...): Initializes the contract with the verifier, Merkle tree parameters, denomination, and token addresses. It also deploys a new Minter contract and sets the owner.

updateVerifier(IVerifier _newVerifier): Allows the owner to update the verifier contract.

deposit(bytes32 _commitment, uint256 _amount, address _token):

Accepts a _commitment, _amount, and _token address.

Requires a depositFee to be paid with the transaction.

Checks that the commitment is unique and the deposit amount matches the DENOMINATION.

Transfers the specified amount of the underlying token from the user to the zPool contract.

Approves the Minter contract to handle the tokens.

Calls minter.mintZToken to mint the wrapped token.

Inserts the commitment into the Merkle tree.

Emits a Deposit event.

withdraw(bytes calldata _proof, bytes32 _root, bytes32 _nullifierHash, address payable _recipient, address _token):

Allows a user to withdraw tokens anonymously.

Checks that the _nullifierHash has not been spent and the Merkle tree _root is known.

Constructs and verifies the zero-knowledge proof using the verifier contract. The proof confirms that the user knows a secret commitment without revealing it.

Adds the _nullifierHash to the s_nullifierHashes mapping to prevent double-spending.

Calls minter.burnZToken to burn the wrapped token.

Audit Note: The line IERC20(_token).transfer(_recipient, DENOMINATION); is commented out, meaning the transfer of the underlying token to the recipient is currently not happening. This is a critical bug.

Emits a Withdrawal event.

## Minter.sol
The Minter contract is a utility contract that handles the creation and destruction of wrapped "ZTokens." It acts as a registry, mapping underlying tokens (e.g., DAI, USDC) to their corresponding wrapped ZTokens.

## State Variables
tokenToZToken: A mapping that stores the address of the wrapped ZToken for each underlying token.

owner: The address that owns the contract.

## Functions
constructor(): Initializes the owner to the contract deployer.

addToken(address token, address zToken): Allows the owner to add a new token pair to the tokenToZToken mapping.

mintZToken(address sender, address token, uint256 amount):

Called by the zPool contract during a deposit.

Requires that the token is supported.

Transfers the underlying token from msg.sender to the Minter contract.

Mints the corresponding ZToken to the sender.

Emits a ZTokenMinted event.

burnZToken(address receiver, address token, uint256 amount):

Called by the zPool contract during a withdrawal.

Requires that the token is supported.

Transfers the underlying token to the receiver.

Burns the corresponding ZToken from the receiver.

Emits a ZTokenBurned event.

ZFactory.sol
The ZFactory contract is a factory for deploying zPool and Minter contracts. It maintains a registry of all deployed instances.

## State Variables
deployedZTokens: An array of all deployed Minter contract addresses.

deployedZPools: An array of all deployed zPool contract addresses.

owner: The address that owns the contract.

lastZTokenFor: A mapping to keep track of the last Minter contract deployed for a given underlying token.

isPool: A mapping to quickly check if an address is a deployed zPool.

isZToken: A mapping to quickly check if an address is a deployed Minter.

## Functions
constructor(...):

Initializes the owner.

Deploys an initial zPool and Minter instance.

Registers the token mapping in the newly created Minter.

Records the new pool and token addresses in the factory's state.

createZPool(...):

Allows the owner to deploy a new zPool contract with specified parameters.

Deploys a new Minter contract for the new pool and registers the token mapping.

Adds the new pool address to deployedZPools and isPool.

zTokenCount(): Returns the total number of deployed Minter contracts.

zPoolCount(): Returns the total number of deployed zPool contracts.

createMinterTokenAndRegister(address underlying, address wrapToken): Deploys a new Minter contract and registers a new token mapping.

createMinterToken(): Deploys a new Minter contract without registering any token mappings.

## Mock Tokens
The MockDAI, MockUSDC, and MockUSDT contracts are simple ERC20 implementations used for testing and development. They inherit from OpenZeppelin's ERC20 contract and include mint and burn functions to easily create and destroy tokens for testing purposes. These contracts are not part of the core functionality but are essential for simulating token interactions within the zPool and Minter contracts.

This is a modular, multi-contract design centered around a privacy-preserving token mixer. The primary goal is to enable users to deposit and withdraw tokens anonymously using zero-knowledge proofs.

## Core Components
The system is composed of four main contracts and a set of mock tokens for development:

zPool.sol (The Mixer): This is the central contract where the privacy-enhancing logic resides. It acts as a non-custodial mixer that accepts token deposits and facilitates private withdrawals.

Deposits: Users deposit a fixed amount (DENOMINATION) of a specific token and pay a small Ether fee. In return, the zPool contract generates a commitment—a cryptographic hash that represents the user's deposit without revealing their address or a unique identifier. This commitment is then inserted into a Merkle tree, a data structure that allows for efficient and verifiable proof of inclusion.

Withdrawals: To withdraw, a user provides a zero-knowledge proof that they own a commitment in the Merkle tree and a nullifier hash that prevents double-spending. The proof confirms their ownership without revealing which commitment they are redeeming. The zPool verifies the proof, marks the nullifier as spent, and then initiates the token transfer.

Security: It inherits from ReentrancyGuard to prevent reentrancy attacks and uses a verifier contract to validate the cryptographic proofs.

Minter.sol (The Wrapper): This contract acts as a bridge between the underlying tokens (like DAI, USDC) and their wrapped "ZToken" counterparts. It handles the minting and burning of these wrapped tokens.

Minting: When a user deposits into the zPool, the Minter takes the underlying token and mints an equivalent amount of a wrapped ZToken to the user's address.

Burning: During a withdrawal, the Minter burns the ZToken from the user and releases the underlying token back to them.

Registry: It maintains a mapping (tokenToZToken) to link each underlying token address to its corresponding wrapped ZToken address.

ZFactory.sol (The Deployment Manager): This factory contract is responsible for deploying new instances of zPool and Minter contracts.

Initialization: The factory deploys an initial set of contracts upon its own creation.

Management: It provides functions for an owner to create new pools for different tokens and maintains a registry of all deployed instances. This allows for a scalable system where multiple zPools can be created, each handling a different token (e.g., a zPool for DAI, another for USDC).

Mock Tokens (MockDAI, MockUSDC, etc.): These are simple ERC20 token contracts used for development and testing. They include mint and burn functions to simulate the issuance and destruction of tokens, which is crucial for testing the zPool and Minter contract logic without relying on real-world tokens.

## Architectural Flow
Deployment: The ZFactory contract is deployed, which in turn deploys an initial zPool and Minter. The Minter is configured to recognize the underlying and wrapped token pair.

## Deposit Process:
a. A user calls the deposit function on the zPool contract, sending a _commitment and the required _amount of a specific token.
b. The zPool receives the token from the user.
c. The zPool calls minter.mintZToken, which takes the user's underlying token and mints the wrapped ZToken to their address.
d. The zPool inserts the _commitment into its Merkle tree.

## Withdrawal Process:
a. A user generates a zero-knowledge proof off-chain, proving they have a valid commitment.
b. The user calls the withdraw function on the zPool, providing the proof, the Merkle tree root, their unique nullifier hash, and their desired recipient address.
c. The zPool validates the zero-knowledge proof using the verifier contract.
d. If the proof is valid, the zPool calls minter.burnZToken.
e. The Minter burns the wrapped ZToken from the user's address.
f. The Minter (or zPool - though the zPool's transfer function is commented out) transfers the underlying token to the recipient's address.
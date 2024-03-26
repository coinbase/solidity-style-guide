# Coinbase Solidity Style Guide

This is a guide for Coinbase engineers developing EVM-based smart contracts. We use Solidity when developing such contracts, so we call it a "Solidity Style Guide." This guide also covers development and testing practices. We are sharing this publicly in hopes that it is useful to others.

## 1. Style

### A. Unless an exception is specifically noted, we follow the official [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html).

### B. Exceptions

#### 1. Internal functions in a library should not have an underscore prefix in their name.

The style guide states

> Underscore Prefix for Non-external Functions and Variables

One of the motivations for this rule is that it is a helpful visual clue.

> Leading underscores allow you to immediately recognize the intent of such functions...

We agree that a leading underscore is a useful visual clue, and this is why we oppose using them for internal library functions that can be called from other contracts. Visually, it looks wrong.

```
Library._function()
```

or

```
using Library for bytes
bytes._function()
```

You might ask, "Why not just use public functions in these cases?"

Whether a library functions are internal or external has important implications. From the [documentation](https://docs.soliditylang.org/en/latest/contracts.html#libraries)

> ... the code of internal library functions that are called from a contract and all functions called from therein will at compile time be included in the calling contract, and a regular JUMP call will be used instead of a DELEGATECALL.

Developers may prefer internal functions because they are more gas efficient to call.

If a function should never be called from another contract, it should be marked private and its name should have a leading underscore.

### C. Additions

#### 1. Prefer custom errors.

Custom errors are in some cases more gas efficient and allow passing useful information.

#### 2. Custom error names should be CapWords style.

For example, `InsufficientBalance`.

#### 3. Event names should be past tense.

For example, `UpdatedOwner` not `UpdateOwner`.

Events should track things that _happened_ and so should be past tense. Using past tense also helps avoid naming collisions with structs or functions.

We are aware this does not follow precedent from early ERCs, like [ERC-20](https://eips.ethereum.org/EIPS/eip-20). However it does align with some more recent high profile Solidity, e.g. [1](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/976a3d53624849ecaef1231019d2052a16a39ce4/contracts/access/Ownable.sol#L33), [2](https://github.com/aave/aave-v3-core/blob/724a9ef43adf139437ba87dcbab63462394d4601/contracts/interfaces/IAaveOracle.sol#L25-L31), [3](https://github.com/ProjectOpenSea/seaport/blob/1d12e33b71b6988cbbe955373ddbc40a87bd5b16/contracts/zones/interfaces/PausableZoneEventsAndErrors.sol#L25-L41).

#### 4. Avoid using assembly.

Assembly code is hard to read and audit. We should avoid it unless the gas savings are very consequential, e.g. > 25%.

#### 5. Avoid unnecessary named return arguments.

In short functions, named return arguments are unnecessary.

NO:

```solidity
function add(uint a, uint b) public returns (uint result) {
  result = a + b;
}
```

Named return arguments can be helpful in functions with multiple returned values.

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData)
```

However, it is important to be explicit when returning early.\
NO:

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData) {
  context = "";
  validationData = 1;

  if (condition) {
    return;
  }
}
```

YES:

```solidity
function validate(UserOperation calldata userOp) external returns (bytes memory context, uint256 validationData) {
  context = "";
  validationData = 1;

  if (condition) {
    return (context, validationData);
  }
}
```

# Coinbase Solidity Style Guide

This is a guide for Coinbase engineers developing EVM-based smart contracts. We use Solidity when developing such contracts, so we call it a "Solidity Style Guide." This guide also covers development and testing practices. We are sharing this publicly in case it is useful to others.

## Why?

We should be as specific and thorough as possible when defining our style, testing, and development practices. Any time we save not having to debate these things on pull requests is productive time that can go into other discussion and review. Following the style guide is evidence of care.

![tom-sachs-gq-style-spring-2019-05](https://github.com/coinbase/solidity-style-guide/assets/6678357/9e904107-e83f-4d89-a405-d3f1394d8de4)

## 1. Style

### A. Unless an exception or addition is specifically noted, we follow the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html).

### B. Exceptions

#### 1. Names of internal functions in a library should not have an underscore prefix.

The style guide states

> Underscore Prefix for Non-external Functions and Variables

One of the motivations for this rule is that it is a helpful visual clue.

> Leading underscores allow you to immediately recognize the intent of such functions...

We agree that a leading underscore is a useful visual clue, and this is why we oppose using them for internal library functions that can be called from other contracts. Visually, it looks wrong.

```solidity
Library._function()
```

or

```solidity
using Library for bytes
bytes._function()
```

Note, we cannot remedy this by insisting on the use public functions. Whether a library functions are internal or external has important implications. From the [Solidity documentation](https://docs.soliditylang.org/en/latest/contracts.html#libraries)

> ... the code of internal library functions that are called from a contract and all functions called from therein will at compile time be included in the calling contract, and a regular JUMP call will be used instead of a DELEGATECALL.

Developers may prefer internal functions because they are more gas efficient to call.

If a function should never be called from another contract, it should be marked private and its name should have a leading underscore.

### C. Additions

#### 1. Errors

##### A. Using Custom Errors Over Require

Utilize custom errors for clearer and more gas-efficient error handling.

```solidity
error InsufficientFunds(uint256 requested, uint256 available);

function withdraw(uint256 amount) public {
    if (amount > balance) {
        revert InsufficientFunds(amount, balance);
    }
    balance -= amount;
}
```

> [!TIP]
> 💡 Custom errors save gas and provide more detailed error messages compared to traditional `require` strings.

##### B. Require with Custom Error (Solidity 0.8.26+)

Use the new `require(condition, error)` syntax to include custom errors in `require` statements, available in Solidity 0.8.26 and later.

```solidity
error InsufficientFunds(uint256 requested, uint256 available);

function withdraw(uint256 amount) public {
    require(amount <= balance, InsufficientFunds(amount, balance));
    balance -= amount;
}
```

> [!IMPORTANT]
> This new syntax provides a more efficient way to handle errors directly within `require` statements, enhancing both readability and gas efficiency.

##### C. Limit Require Messages

Prefer using custom errors over `require` with strings for better efficiency. If you must use `require` with a string message, keep it under 32 bytes to reduce gas costs.

> [!IMPORTANT]
> Custom errors are more gas-efficient and provide clearer error handling. Whenever possible, use them instead of `require` with a string message.

**Yes:**

```solidity
require(balance >= amount, "Insufficient funds");
```

> [!CAUTION]
> Keeping `require` messages concise (under 32 bytes) minimizes additional gas costs and improves efficiency.

**No:**

```solidity
require(balance >= amount, "The balance is insufficient for the withdrawal amount requested.");
```

> [!WARNING]
> Longer messages significantly increase gas costs. Avoid using verbose messages in `require` statements.


##### D. Custom error names should be CapWords style.

For example, `InsufficientBalance`.

#### 2. Events

##### A. Events names should be past tense.

Events should track things that _happened_ and so should be past tense. Using past tense also helps avoid naming collisions with structs or functions.

We are aware this does not follow precedent from early ERCs, like [ERC-20](https://eips.ethereum.org/EIPS/eip-20). However it does align with some more recent high profile Solidity, e.g. [1](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/976a3d53624849ecaef1231019d2052a16a39ce4/contracts/access/Ownable.sol#L33), [2](https://github.com/aave/aave-v3-core/blob/724a9ef43adf139437ba87dcbab63462394d4601/contracts/interfaces/IAaveOracle.sol#L25-L31), [3](https://github.com/ProjectOpenSea/seaport/blob/1d12e33b71b6988cbbe955373ddbc40a87bd5b16/contracts/zones/interfaces/PausableZoneEventsAndErrors.sol#L25-L41).

YES:

```solidity
event OwnerUpdated(address newOwner);
```

NO:

```solidity
event OwnerUpdate(address newOwner);
```

##### B. Prefer `SubjectVerb` naming format.

YES:

```solidity
event OwnerUpdated(address newOwner);
```

NO:

```solidity
event UpdatedOwner(address newOwner);
```

#### 3. Named arguments and parameters

##### A. Avoid unnecessary named return arguments.

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

However, it is important to be explicit when returning early.

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

##### B. Prefer named arguments.

Passing arguments to functions, events, and errors with explicit naming is helpful for clarity, especially when the name of the variable passed does not match the parameter name.

YES:

```
pow({base: x, exponent: y, scalar: v})
```

NO:

```
pow(x, y, v)
```

##### C. Prefer named parameters in mapping types.

Explicit naming parameters in mapping types is helpful for clarity, especially when nesting is used.

YES:

```
mapping(address account => mapping(address asset => uint256 amount)) public balances;
```

NO:

```
mapping(uint256 => mapping(address => uint256)) public balances;
```

#### 4. Structure of a Contract

##### A. Prefer composition over inheritance.

If a function or set of functions could reasonably be defined as its own contract or as a part of a larger contract, prefer defining it as part of a larger contract. This makes the code easier to understand and audit.

Note this _does not_ mean that we should avoid inheritance, in general. Inheritance is useful at times, most especially when building on existing, trusted contracts. For example, _do not_ reimplement `Ownable` functionality to avoid inheritance. Inherit `Ownable` from a trusted vendor, such as [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/) or [Solady](https://github.com/Vectorized/solady).

##### B. Avoid writing interfaces.

Interfaces separate NatSpec from contract logic, requiring readers to do more work to understand the code. For this reason, they should be avoided.

##### C. Avoid using assembly.

Assembly code is hard to read and audit. We should avoid it unless the gas savings are very consequential, e.g. > 25%.

#### 4. Versioning

##### A. Avoid unnecessary version Pragma constraints.

While the main contracts we deploy should specify a single Solidity version, all supporting contracts and libraries should have as open a Pragma as possible. A good rule of thumb is to the next major version. For example

```solidity
pragma solidity ^0.8.0;
```

#### 5. Struct and Error Definitions

##### A. Prefer declaring structs and errors within the interface, contract, or library where they are used.

##### B. If a struct or error is used across many files, with no interface, contract, or library reasonably being the "owner," then define them in their own file. Multiple structs and errors can be defined together in one file.

#### 6. Imports

##### A. Use named imports.

Named imports help readers understand what exactly is being used and where it is originally declared.

YES:

```solidity
import {Contract} from "./contract.sol"
```

NO:

```solidity
import "./contract.sol"
```

For convenience, named imports do not have to be used in test files.

##### B. Order imports alphabetically (A to Z) by file name.

YES:

```solidity
import {A} from './A.sol'
import {B} from './B.sol'
```

NO:

```solidity
import {B} from './B.sol'
import {A} from './A.sol'
```

##### C. Group imports by external and local with a new line in between.

For example

```solidity
import {Math} from '/solady/Math.sol'

import {MyHelper} from './MyHelper.sol'
```

In test files, imports from `/test` should be their own group, as well.

```solidity
import {Math} from '/solady/Math.sol'

import {MyHelper} from '../src/MyHelper.sol'

import {Mock} from './mocks/Mock.sol'
```

#### 7. Comments

##### A. Commenting to group sections of the code is permitted.

Sometimes authors and readers find it helpful to comment dividers between groups of functions. This permitted, however ensure the style guide [ordering of functions](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-functions) is still followed.

For example

```solidity
/// External Functions ///
```

```solidity
/*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
/*                   VALIDATION OPERATIONS                    */
/*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/
```

##### B. ASCII Art

ASCII art is permitted in the space between the end of the Pragmas and the beginning of the imports.

## 2. Development

### A. Use [Forge](https://github.com/foundry-rs/foundry/tree/master/crates/forge) for testing and dependency management.

### B. Testing

#### 1. Test file names should follow Solidity Style Guide conventions for files names and also have `.t` before `.sol`.

For example, `ERC20.t.sol`

#### 2. Test contract names should include the name of the contract or function being tested, followed by "Test".

For example,

- `ERC20Test`
- `TransferFromTest`

#### 3. Test names should follow the convention `test_functionName_outcome_optionalContext`

For example

- `test_transferFrom_debitsFromAccountBalance`
- `test_transferFrom_debitsFromAccountBalance_whenCalledViaPermit`
- `test_transferFrom_reverts_whenAmountExceedsBalance`

If the contract is named after a function, then function name can be omitted.

```solidity
contract TransferFromTest {
  function test_debitsFromAccountBalance() ...
}
```

#### 4. Prefer tests that test one thing.

This is generally good practice, but especially so because Forge does not give line numbers on assertion failures. This makes it hard to track down what, exactly, failed if a test has many assertions.

YES:

```solidity
function test_transferFrom_debitsFrom() {
  ...
}

function test_transferFrom_creditsTo() {  
  ...
}

function test_transferFrom_emitsCorrectly() {
  ...
}

function test_transferFrom_reverts_whenAmountExceedsBalance() {
  ...
}
```

NO:

```solidity
function test_transferFrom_works() {
  // debits correctly
  // credits correctly
  // emits correctly
  // reverts correctly
}
```

Note, this does not mean a test should only ever have one assertion. Sometimes having multiple assertions is helpful for certainty on what is being tested.

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  ...
  assertEq(balanceOf(to), amount);
}
```

#### 5. Use variables for important values in tests

YES:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  uint amount = 10;
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

NO:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  transferFrom(from, to, 10);
  assertEq(balanceOf(to), 10);
}
```

#### 6. Prefer fuzz tests.

All else being equal, prefer fuzz tests.

YES:

```solidity
function test_transferFrom_creditsTo(uint amount) {
  assertEq(balanceOf(to), 0);
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

NO:

```solidity
function test_transferFrom_creditsTo() {
  assertEq(balanceOf(to), 0);
  uint amount = 10;
  transferFrom(from, to, amount);
  assertEq(balanceOf(to), amount);
}
```

### C. Project Setup

#### 1. Avoid custom remappings.

[Remappings](https://book.getfoundry.sh/projects/dependencies?#remapping-dependencies) help Forge find dependencies based on import statements. Forge will automatically deduce some remappings, for example

```rust
forge-std/=lib/forge-std/src/
solmate/=lib/solmate/src/
```

We should avoid adding to these or defining any remappings explicitly, as it makes our project harder for others to use as a dependency. For example, if our project depends on Solmate and so does theirs, we want to avoid our project having some irregular import naming, resolved with a custom remapping, which will conflict with their import naming.

### D. Upgradability

#### 1. Prefer [ERC-7201](https://eips.ethereum.org/EIPS/eip-7201) "Namespaced Storage Layout" convention to avoid storage collisions.

### E. Structs

#### 1. Where possible, struct values should be packed to minimize SLOADs and SSTOREs.

#### 2. Timestamp fields in a struct should be at least uint32 and ideally be uint40.

`uint32` will give the contract ~82 years of validity `(2^32 / (60*60*24*365)) - (2024 - 1970)`. If space allows, uint40 is the preferred size.

## 3. NatSpec

### A. Unless an exception or addition is specifically noted, follow [Solidity NatSpec](https://docs.soliditylang.org/en/latest/natspec-format.html).

### B. Additions

#### 1. All external functions, events, and errors should have complete NatSpec.

Minimally including `@notice`. `@param` and `@return` should be present if there are parameters or return values.

#### 2. Struct NatSpec

Structs can be documented with a `@notice` above and, if desired, `@dev` for each field.

```solidity
/// @notice A struct describing an accounts position
struct Position {
  /// @dev The unix timestamp (seconds) of the block when the position was created.
  uint created;
  /// @dev The amount of ETH in the position 
  uint amount;
}
```

#### 3. Newlines between tag types.

For easier reading, add a new line between tag types, when multiple are present and there are three or more lines.

YES:

```solidity
/// @notice ...
///
/// @dev ...
/// @dev ...
/// 
/// @param ...
/// @param ...
/// 
/// @return
```

NO:

```solidity
/// @notice ...
/// @dev ...
/// @dev ...
/// @param ...
/// @param ...
/// @return
```

#### 4. Author should be Coinbase.

If you using the `@author` tag, it should be

```solidity
/// @author Coinbase
```

Optionally followed by a link to the public Github repository.

# Reentrancy Attack

## Table of Contents

- [Glossary](#glossary)
- [The Attack](#the-attack)
    - [Victim contract](#victim)
    - [Attacking contract](#attack)
    - [Step by Step](#step-by-step)
        - [Visual Representation](#visual-representation)
- [Preventative Techniques](#preventative-techniques)
    - [Using `transfer`](#using-transfer)
    - [Executing Logic Before Sending Ether](#executing-logic-before-sending-ether)

## Glossary

- [Fallback Function](https://solidity.readthedocs.io/en/v0.5.3/contracts.html#fallback-function)
- [Function Selector](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html#function-selector)
- [Transfer Function](https://solidity.readthedocs.io/en/latest/units-and-global-variables.html#address-related)

## The Attack

Smart contracts, written in _Solidity_ for the _Ethereum Virtual Machine (EVM)_, are able to make external calls to either _Externally Owned Accounts (EOAs)_ or other smart contracts.

When a _reentrancy attack_ occurs, an attacker abuses an external call made by the victim smart contract to execute a _fallback function_ within the attacking smart contract that then performs an additional external call back into the victim smart contract, hence the term _reentrancy_.

This presents a problem when the victim smart contract's external call is executed **before** some mission critical logic takes place to maintain expected state and behavior of the victim smart contract.

For example, the following code represents two smart contracts, one named `Victim` and the other, `Attack`:

### Victim

```solidity
contract Victim {
    mapping(address => uint256) public balances;

    function depositFunds() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds(uint256 _amountToWithdraw) public {
        require(balances[msg.sender] >= _amountToWithdraw);

        // This is where the external call is happening,
        // and where the vulnerability lies
        require(msg.sender.call.value(_amountToWithdraw)());

        // For this contract, this is the mission critical logic
        // that's taking place AFTER the external call.
        // Here we are maintaining the appropriate state
        // for this contract by deducting what msg.sender
        // has withdrawn from their balance,
        // but only AFTER we've sent ether to msg.sender
        balances[msg.sender] -= _amountToWithdraw;
    }
}
```

### Attack

```solidity
import "Victim.sol";

contract Attack {
    Victim public victim;

    constructor(address _victimAddress) {
        // Initialize the victim variable with the
        // Victim smart contract address
        victim = Victim(_victimAddress);
    }

    function attackVictim() public payable {
        // We're going to be attacking the Victim contract
        // to the nearest ether
        require(msg.value >= 1 ether);
        
        victim.depositFunds.value(1 ether)();
        victim.withdrawFunds(1 ether);
    }

    function collectEther() public {
        msg.sender.transfer(this.balance);
    }

    // This is the fallback function that is executed
    // when the Victim smart contract makes it's
    // external call to this contract
    function () payable {
        if (victim.balance > 1 ether) {
            victim.withdrawFunds(1 ether);
        }
    }
}
```

### Step by Step

If you haven't seen the vulnerability yet, follow the steps below to get a step-by-step picture of how the attack would play out:

**NOTE** For this example, let's imagine other Ethereum users has deposited a total of **10 ether** into the `Victim` contract

1. The `Victim` contract is deployed
2. The attacker gets the address of the deployed `Victim` contract, and deploys the `Attack` contract passing the `Victim` contract's address to `Attack`'s `constructor`
3. The attacker would call the `attackVictim` function with **1 ether** and provide **a lot** of `gas` for the transaction
    - The amount of `gas` that is sent with the transaction, is important because if not enough `gas` is sent, the transaction could _revert_ before all the funds were withdrawn causing the attacker's would-be ether to remain in the `Victim` contract
4. The `attackVictim` function will then call the `depositFunds` function on the `Victim` contract, which will increase the balace for the `Attack` contract's address to **1 ether**
5. Then `attackVictim` calls the `withdrawFunds` function on the `Victim` contract, passing it **1 ether**
6. The first `require(balances[msg.sender] >= _amountToWithdraw);` check in the `withdrawFunds` function will execute and pass, as the `Attack` contract's address will have a balance of **1 ether**
7. Next `withdrawFunds` will execute: `require(msg.sender.call.value(_amountToWithdraw)());` which will trigger the _fallback function_ in the `Attack` contract, and will also transfer **1 ether** to the `Attack` contract, bringing the total amount of ether in the `Victim` contract to **10** (keep in mind that the starting value was **10 ether**, but we also deposited **1 ether** in the beginning of the `attackVictim` function)
    - The _fallback function_ is executed because the `Victim` contract will not specify a _function selector_, so the `Attack` contract will automatically execute the _fallback function_ specified in the contract
8. Next the _fallback function_ in the `Attack` contract will pass the if statement: `if (victim.balance > 1 ether)` because the balance of the `Victim` contract is **10 ether**, causing...
9. The `Attack` contract to then call the next part of the _fallback function_: `victim.withdrawFunds(1 ether);` which will then _reenter_ the `Victim` contract at the beginning of the `withdrawFunds` fuction
    - It's important to note that the mission critical logic that maintains the appropirate state for each address' balance in the `withdrawFunds` function: `balances[msg.sender] -= _amountToWithdraw;` will have not been executed, because it takes place **AFTER** the `require(msg.sender.call.value(_amountToWithdraw)());` which triggers the _fallback function_ and causes the _reentry_ into the beginning of the `withdrawFunds` function
10. Steps 6 - 9 will repeat until the if statement: `if (victim.balance > 1 ether)` in the `Attack` contract's _fallback function_ fails and allows...
11. The `withdrawFunds` function to execute: `balances[msg.sender] -= _amountToWithdraw;` which will deduct the **1 ether** we deposited in step 4 from the `Attack` contract's address
    - At this point the _reentrancy_ loop is finished and the transaction inititated in step 3 is completed

At the end of this attack, the attacker now has the original **1 ether** that was deposited when calling `attackVictim`, but now also has the **10 ether** that was stored in the `Victim` contract by other Ethereum users.

## Preventative Techniques

### Using `transfer`

The `transfer` method is a built-in Solidity method on the `address` type, e.g.: `myEthereumAddress.transfer(amountToSend)`

When using the `transfer` method, the transaction is restricted to only forwaring **2300 gas** to the reciepient address

So, if the `Victim` contract would have used `msg.sender.transfer(_amountToWithdraw)` instead of `msg.sender.call.value(_amountToWithdraw)()` in the `withdrawFunds` function, it wouldn't have mattered if the attacker passed an exorbitant amount of _gas_ when calling the `attackVictim` function, the `transfer` method would have only forwarded **2300 gas** to the _fallback function_ in the `Attack` contract which would have not been enough to make the _reentracy_ call back into the `Victim` contract; The transaction would have reverted when it ran out of _gas_ leaving the **10 ether** deposited by other Ethereum users within the `Victim` contract.

#### Visual Representation

The diagram starts at Step #2

![Rentrancy Attack Diagram](./assets/reentrancy-attack.jpg)

### Executing Logic Before Sending Ether

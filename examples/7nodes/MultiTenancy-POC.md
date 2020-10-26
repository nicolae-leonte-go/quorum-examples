# Multi tenancy with multiple private states POC
## Set Up
Make sure to build quorum and tessera from the following branches:
* quorum - QuorumEngineering/quorum/multitenant-mps-poc
* tessera - QuorumEngineering/tessera/multitenant

Adjust your path/TESSEAR_JAR and run:
```shell script
./raft-init.sh
./raft-start.sh tessera
```

The above sets up tessera node1 with key1, key5, key6 and key7 and starts the raft network.

## Exercising multiple states
### Accumulator contract
The following contract will be used to highlight the different states the contrat can be in on node1. 
Use `acc-contact.js` to deploy the contract for all node1's public keys. 
```solidity
pragma solidity ^0.5.0;

contract accumulator {
  uint public storedData;

  event IncEvent(uint value);

  constructor(uint initVal) public {
    storedData = initVal;
  }

  function inc(uint x) public {
    storedData = storedData + x;
    emit IncEvent(storedData);
  }

  function get() view public returns (uint retVal) {
    return storedData;
  }
}
```

### Console setup

Open 4 consoles to node1:

* Console1 - key1 (BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=) 
```shell script
PSI="BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=" geth attach http://localhost:22000
```
* Console2 - key5 (R56gy4dn24YOjwyesTczYa8m5xhP6hF2uTMCju/1xkY=) 
```shell script
PSI="R56gy4dn24YOjwyesTczYa8m5xhP6hF2uTMCju/1xkY=" geth attach http://localhost:22000
```
* Console3 - key6 (UfNSeSGySeKg11DVNEnqrUtxYRVor4+CvluI8tVv62Y=) 
```shell script
PSI="UfNSeSGySeKg11DVNEnqrUtxYRVor4+CvluI8tVv62Y=" geth attach http://localhost:22000
```
* Console3 - key7 (ROAZBWtSacxXQrOe3FGAqJDyJjFePR5ce4TSIzmJ0Bc=) 
```shell script
PSI="ROAZBWtSacxXQrOe3FGAqJDyJjFePR5ce4TSIzmJ0Bc=" geth attach http://localhost:22000
```

Open one console to node4:

```shell script
geth attach qdata/dd4/geth.ipc 
```

### Steps
#### Deploy the contract and check the state in every console
On the node4 console run:
```shell script
> loadScript('acc-contract.js')


Contract transaction send: TransactionHash: 0x0d7e977718c994d5666ddd1e2cedc49b166ed5b1cb39ad176f2cc94d695ca908 waiting to be mined...
true
Contract mined! Address: 0x180893a0ec847fa8c92786791348d7d65916acbb
```

On every console run:
```javascript
var address = "0x180893a0ec847fa8c92786791348d7d65916acbb";
var abi = [{"constant":true,"inputs":[],"name":"storedData","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"retVal","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"x","type":"uint256"}],"name":"inc","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[{"name":"initVal","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"name":"value","type":"uint256"}],"name":"IncEvent","type":"event"}];
var acc = eth.contract(abi).at(address)
acc.IncEvent().watch( function (error, log) {
    console.log("\nIncEvent details:")
    console.log("    NewValue:", log.args.value)
});
acc.get()
```
The `acc.IncEvent().watch(...)` creates the relevant filter for the InvEvent and then starts watching for new events. Confirm that the state of the contract in all consoles is 1.

On node4's console do (increment for key1, key5, key6):
```javascript
acc.inc(1,{from:eth.accounts[0],privateFor:["BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=", "R56gy4dn24YOjwyesTczYa8m5xhP6hF2uTMCju/1xkY=", "UfNSeSGySeKg11DVNEnqrUtxYRVor4+CvluI8tVv62Y="]});
``` 
On all consoles (except for node1's fourth console - PSI=ROAZBWtSacxXQrOe3FGAqJDyJjFePR5ce4TSIzmJ0Bc=) you should see:

```
IncEvent details:
    NewValue: 2
```

Do an `acc.get()` in all consoles. You should get 2 in all but node1 console 4 (where you should still get 1).

On node4's console do (increment for key1, key5):
```javascript
acc.inc(1,{from:eth.accounts[0],privateFor:["BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=", "R56gy4dn24YOjwyesTczYa8m5xhP6hF2uTMCju/1xkY="]});
``` 

Do an `acc.get()` in all consoles. You should get `3` in node1 consoles 1 and 2, `2` in node1 console 3, `1` node1 console 1 and 4 in node4's console.

Do an `eth.getTransactionReceipt("0x34f881b717ddb3b1ab05fb948d375b593f4c74576258915e06a7f9cd9d0d15f4")` (the last transaction from node4) in all consoles and observe the different results (similar to the observed states above).
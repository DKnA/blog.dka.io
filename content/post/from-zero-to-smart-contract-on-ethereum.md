---
title: "From zero to Smart Contract on Ethereum: How to setup and deploy a smart contract"
date: 2017-08-14T20:53:09+02:00
draft: false
author: "Piotr Adamski"
authorLink: "https://twitter.com/mcveat"
---

Smart contracts are agreements written in a specialized programming language that allows for automatic verification and execution. These contracts can be used in a variety of instances, from simple loans to currency.

Using blockchain technology, smart contracts are traceable, transparent and irreversible. If you need a review on blockchain, [you may find this article quite helpful](https://medium.com/the-intrepid-review/how-does-the-blockchain-work-for-dummies-explained-simply-9f94d386e093).

Ethereum is a a blockchain implementation used to deploy smart contracts. We're going to create a private blockchain, connect to it, mine for blocks and deploy a contract.

# Build a Private Blockchain

Using private blockchain over the official test or main Ethereum networks has two main advantages. With proper setup transactions cost nothing and are executed at much faster rate. Here are the two tools used to build your blockchain:

`geth` is a Go implementation of the Ethereum client.

`solc` is the Solidity compiler. [Solidity](https://solidity.readthedocs.io/en/develop/) is [one](https://github.com/ethereum/cpp-ethereum/wiki/LLL-PoC-6/7a575cf91c4572734a83f95e970e9e7ed64849ce) [of the few](https://github.com/ethereum/serpent) [supported](https://github.com/ethereum/viper) programming languages that Ethereum smart contracts are written in.

To install Solidity on MacOS X, run the following commands:

```
brew update
brew upgrade
brew tap ethereum/ethereum
brew install ethereum
brew install ethereum/ethereum/solidity
```

> This post is written at the time when most recent stable versions of `geth` and `solc` were `1.6.7-stable` and `0.4.14+commit.c2215d46.Darwin.appleclang` respectively.

# Create a Genesis Block

The genesis block is the first block in the chain. In Ethereum, the block defines fundamental constants of the chain and turns certain features on or off. A genesis block definition is needed to start a blockchain. Our network will use the following definition:

<script src="https://gist.github.com/mcveat/6ddbc73d1dde311c3ff46e7ec9bcf456.js"></script>

Two important fields here are `difficulty` and `gasLimit`.

**Difficulty** controls the time it takes to mine a new block. The higher the value, the more time it takes to discover a valid block. New transactions are written to the blockchain only when new blocks are found. You'll want to set difficulty to a relatively low value to avoid waiting too long.

**Gas limit** describes the maximum computational effort per block. Each operation, whether calculating a hash, executing a contract or adding numbers, has a fixed gas price assigned.

# Initialize the Network

To initialize the new network using the genesis block, call the command below:

```
geth --datadir data --networkid 15 init genesis.json
```

`datadir` is an argument that points to the directory where `geth` will store the blockchain and keystore data. This command may overwrite main network data. If you're using the main network, make sure to point it to some other directory.

`networkid` is used to isolate your network from other networks. New nodes can join if they are using the same protocol version and network id. 0 is retired, 1 identifies Frontier, the main network, while 2 and 3 are for public test networks, Morden and Ropsten.

# Connect to the Network

Now your network has been created. Open your interactive javascript console and enter:

```
geth --datadir data --networkid 15 console
```

Anytime you need to leave, exit by calling:

```
exit
```

# Create Accounts

Each piece of information stored in the chain has to be represented as a transaction between accounts. Let's create accounts.  Use `geth` console to execute two statements below:

```javascript
bob = personal.newAccount("bobs password")
john = personal.newAccount("johns password")
```

The`newAccount` method takes the password as an argument and returns the address of the account. Each account is described by an address and an Ether balance. Account key files are stored in the `data/keystore` directory. Accessing the account requires account key file and password, so this information should be backed up and kept secret.

Let's check Bob's balance.

```javascript
eth.getBalance(bob)
```

New accounts are empty. To initiate a transaction, an account has to have a positive Ether balance. There are two ways to acquire Ether. You can receive it from someone or you can mine. 

# Mine for Blocks

The process of mining is based on searching for a valid block. The account that discovers a new valid block gets paid in Ether. 

Let's mine some blocks then!

```javascript
miner.setEtherbase(bob)
miner.start(1)
```

The `setEtherbase` method sets the address of the account that will receive Ether for the mined blocks. Every instance of `geth` can run only one miner.

The `start` method initiates the mining process. The argument it takes determines the number of threads that will be used for mining. You probably shouldn’t set it higher than the number of cores your machine has.

Let it run for a moment. Every time a new block is mined you should see log messages much like this:


```
INFO [08-01|19:04:07] Commit new mining work                   number=1 txs=0 uncles=0 elapsed=181.385µs
INFO [08-01|19:04:39] Successfully sealed new block            number=1 hash=9382ed…6937ab
INFO [08-01|19:04:39] 🔨 mined potential block                  number=1 hash=9382ed…6937ab
```

Call the `stop` method to stop mining:

```javascript
miner.stop()
```

Check the balance of Bob's account. There should be more zeros this time around.

> You can [download create-accounts.js script](https://gist.github.com/mcveat/4e55372d1568063d6902e72a4c1eb29d) that contains all the code that was used in this part. In order to run it before opening the console specify path to the script in `preload` argument of `geth` command: `geth --datadir data --networkid 15 --preload scripts/create-accounts.js console`.

<!--
There is an alternate method that sets balances in genesis block. Probably not necessary in this article

```
> bob
"0xfa4e08705260990257f8b2b145d9601ab6a3e84b"
> john
"0x5db943431beb852b88a51be2bbf77e6c411bdf12"
```

<script src="https://gist.github.com/mcveat/5d50d979125dc7e8062126bff6d5441c.js"></script>

```
rm -R data/geth/
```

```
> eth.getBalance("0xfa4e08705260990257f8b2b145d9601ab6a3e84b")
123456789
```
-->

# Get a Contract

Now you have enough Ether to deploy your contract. You may use the one below.

<script src="https://gist.github.com/mcveat/ea4330c671d2c5fa341a6003295397cc.js"></script>

The contract above is an updated version of the [Minimum Viable Token](https://www.ethereum.org/token#minimum-viable-token) contract from Ethereum's official documentation. Constructor method `KrugerToken` is responsible for creating the contract with a set balance of tokens assigned to the contract creator. The token balances of all accounts that have ever interacted with the contract are stored in its `balanceOf` property. Additionally, the contract allows accounts with a non-zero balance to transfer tokens to another account. You can do that using `transfer` method.

# Compile the Contract

Since `geth` version 1.6 [integrations with compilers were dropped from the console](https://github.com/ethereum/EIPs/issues/209), `solc` is meant to be a standalone tool from now on. Let's put it to work and compile our contract.

```
solc --bin --abi src/KrugerToken.sol
```

The command above outputs two representations of a compiled contract. Binary is nothing else than one would expect - a hexadecimal blob of Ethereum Virtual Machine code. **ABI**, Application Binary Interface, is static,  a strongly typed interface to the contract expressed in json.

You need both to deploy a contract. In order to interact with one, you need just ABI and an address of the block that stores the contract.

Let's assign them to variables in `geth` console:

```javascript
abi = [{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transfer","outputs":[],"payable":false,"type":"function"},{"inputs":[{"name":"initialSupply","type":"uint256"}],"payable":false,"type":"constructor"}]

code = "0x6060604052341561000f57600080fd5b604051602080610310833981016040528080519060200190919050505b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020819055505b505b61028f806100816000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806370a0823114610049578063a9059cbb14610096575b600080fd5b341561005457600080fd5b610080600480803573ffffffffffffffffffffffffffffffffffffffff169060200190919050506100d8565b6040518082815260200191505060405180910390f35b34156100a157600080fd5b6100d6600480803573ffffffffffffffffffffffffffffffffffffffff169060200190919080359060200190919050506100f0565b005b60006020528060005260406000206000915090505481565b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002054101561013b57600080fd5b6000808373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002054816000808573ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020540110156101c657600080fd5b806000803373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200190815260200160002060008282540392505081905550806000808473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600082825401925050819055505b50505600a165627a7a72305820b1e8c908d9cdc4837ab94f73527a626af9fd9ef444d7874c1bd2f43ec35a70a80029"
```

# Unlock Account

Creating a contract costs gas. You need to unlock the account to pay transaction fees. Calling the method below unlocks the account for 5 minutes:

```javascript
personal.unlockAccount(bob, "bobs password")
```

You can specify duration in seconds as a third argument to the `unlockAccount` method. Whenever the transaction fails with the `authentication needed: password or unlock` error, you'll have to unlock that account again.

# Deploy Contract

To deploy your contract, execute these two statements:

```javascript
contract = eth.contract(abi)
contractInstance = contract.new(1000, {data: code, from: bob, gas: 1000000})
```

The `contract` method creates a contract object based on ABI. It can be used to create a new contract or instantiate one that already exists in the blockchain. On this occasion, you'll want to use the `new` method to create a new instance. First, you specify arguments of a contract constructor, if it has any. In our case, `1000` is passed as an initial supply of Kruger tokens. The last argument is an options object. Its `data` field specifies byte code of the contract, `from` identifies the account that the transaction should be made from and `gas` limits amount of gas used to execute this transaction.

If you inspect the returned contract instance you may notice that it has no address. That is because the contract will be stored on the blockchain with the corresponding transaction on the next block that is mined. You can verify using code below:

```javascript
txpool.status
receipt = eth.getTransactionReceipt(contractInstance.transactionHash)
```

One pending transaction should appear in the transactions pool `status` object and the transaction receipt should be `null`.

Restart the miner again and mine a couple blocks. When you're done, there should be no pending transactions and the receipt will contain the contract address along with other information. Your contract found its way to the chain.

> Once again, you can [download deploy-contract.js script](https://gist.github.com/mcveat/a6054baf3269a7d563e9b7ccd6e2f37b) that contains all the code that was used in this part. Just like before, preload the script before starting the `geth` console by running: `geth --datadir data --networkid 15 --preload "scripts/deploy-contract.js" console`


# Referencing and Interacting with Deployed Contract

To reference a contract that we've deployed, use contract object again. This time around use the `at` method pointed to the contracts address:

```javascript
KrugerToken = contract.at(receipt.contractAddress)
```

Having the contract instantiated from the blockchain code, let's transfer some Kruger tokens from Bob to John:

```javascript
KrugerToken.transfer.sendTransaction(john, 10, {from: bob})
```

Because the `transfer` method modifies token balances, you'll want to record it on the blockchain. `sendTransaction` executes the contract method by sending the transaction. First two arguments correspond to arguments defined for the method in the contract. In this case John is receiving 10 tokens. The last argument is an options object. In this case it specifies the sender. Sender is the account paying for the transaction. Call may fail if corresponding account is locked or empty.

> `sendTransaction` method has its. counterpart: `call`. It executes contract method locally. Execution is not saved to blockchain and it does not cost any Ether.

You can use `balanceOf` property of the contract to check John's token balance.

```javascript
KrugerToken.balanceOf(john)
```

As you can probably guess, that balance did not get updated because the corresponding transaction had not been written to the blockchain. You can verify with transaction pool status.

Mine a few blocks and check again. You should see balances updated.

```javascript
KrugerToken.balanceOf(john)
KrugerToken.balanceOf(bob)
```

And that would be it! 

> As before, you can [grab interact-with-contract.js script](https://gist.github.com/mcveat/946c4f11b8511f2b5d5d286e77e508c8) which contains the code used in this section. This time though, in preloading in the `geth` console, you have to point it to a previous script as well: `geth --datadir data --networkid 15 --preload "scripts/deploy-contract.js,scripts/interact-with-contract.js" console`. Alternatively, you can [download all.js script](https://gist.github.com/mcveat/583a5296d6f69a1838571f47ab2b99f2) which combines scripts from all steps. Finally, there is a [script that truly takes you from zero to smart contract](https://gist.github.com/mcveat/54a5d89052edb5bae8787f2b71d03e80). This installs tools, creates a new network, deploys the contract and sends tokens in one go!
 
The process presented above may not be the most convenient one available ([truffle framework might be one](http://truffleframework.com/)). Yet, deploying a smart contract on the lowest level possible helps build a better understanding of the technology used. I hope it helps you.
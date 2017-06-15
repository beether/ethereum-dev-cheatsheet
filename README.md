# ethereum-dev-cheatsheet
A cheatsheet for developing on the Ethereum platform

* [Ethereum](#cb): A description of how Ethereum works.
	* [Accounts](#cb-a) 
	* [Transactions](#cb-t)
* [Dapp Architecture](#da): A summary of Dapp architecture.
	* [Blockchain](#da-b)
	* [Nodes](#da-n)
	* [Frontend](#da-f)
* [Interacting with Contracts](#ic): Notes about ABIs
* [Solidity Cheatsheet](#sc): Summary of common things in Solidity
	* [Notes](#sc-g)
	* [Sending Messages](#sc-s)
* [Web3 Cheatsheet](#wc): Summary of common things in Web3
* [Truffle Cheatsheet](#tc): Summary of common things in Truffle
	* [Notes](#tc-n)
	* [Compiling/Building](#tc-c)
	* [Deploying/Testing](#tc-d) 

# <a name="cb"></a> Ethereum

Ethereum is a blockchain that allows for executable code (called Contracts) to be put onto the blockchain and to be interacted with by all other accounts.


## <a name="cb-a"></a>Accounts

There are two types of accounts:

* Externally owned account (aka, wallet)
* Contract

**Both types**

* have a balance (in ETH)
* can transfer balance to other accounts

**External accounts**

* can send transactions to the blockchain
	* transactions can simply transfer ETH
	* or they can call contract functions
	* or they can create contracts
* controlled by private keys

**Contracts**

* only ever do stuff via a transaction from an `external account`
* can send messages to other contracts
* can have functions executed via transactions and messages
* do not have private keys (see first bullet)

## <a name="cb-t"></a>Transactions

### IMPORTANT

**All things on the blockchain get initiated by a transaction by an `external account`.**  Everything that ever happens starts with `external accounts` sending transactions to the blockchain.

### The Process

#### 1) Transaction Created

When an `external account` (sender) creates a transaction they specify the following:

* **From**: The account creating this transaction
* **To**: The target recipient of the transaction
* **Data**: If `To` is a contract, then which function to call and with what parameters
* **gasPrice**: How much they are willing to pay (in wei) per one unit of gas
* **gasLimit**: How much total gas they are willing to pay for
* **signature**: Proving this transaction is from **From**

Note:  the maximum transaction fee incurred will be **gasPrice** times **gasLimit**.  The sender must have at least this `balance`.  Some other validations occur as well, such as ensuring the transaction has a proper signature, everything is encoded correctly, etc.

It is sent to the queue.  A `transaction hash` is created that represents this transaction.

#### 2) Pending (waiting to be mined)

Miners operate as follows:

* Look at all pending transactions
* Find the ones with the highest gasPrice (since they'll make the most)
* Find as many as can fit within the current block (there are gas and memory restrictions)
* Execute that transactions.  For each one:
	* as it gets executed, count how much gas is used.  If it ever exceeds `gasLimit` stop immediately, store the tx as failed onto the blockchain.
	* execute code, update storage, put new contract data in block, do internal transactions, etc, etc.
	* charge a transaction fee to `sender` (gasUsed times gasPrice)
	* add all relevant information into the block
	* also add a `txReceipt` to the block
* Start mining the block!

If the miner successfully mines, he broadcasts this to his peers.  They will do the exact same process of executing all transactions to ensure the results are correct.  If they accept the block, they start working on the next block.

This is the first `confirmation`.  Each additional block mined after this one is a confirmation.

#### 3) Done

At this point the `txHash` is on the blockchain and should be contain all the info about what happened.  How much gas was used, internal messages, etc.

# <a name="da"></a>Dapp Architecture

I think "Dapp" is really a misnomer.  What you have are three layers:

## <a name="da-b"></a>Blockchain
Smart contracts deployed onto the blockchain that do stuff and store data.  These are usually written in Solidity, but can be written in anything that compiles to `EVM`, a shitty, slow, super-limited assembly-like language obsessed with everything except usability.

## <a name="da-n"></a>Nodes
A node is the middle-ware that connects the world to the blockchain.  They expose the to the world via an RPC. So, you can do HTTP calls to them to get answers to some queries like:

* Look up any block
* Find balance of any address
* See what contracts are storing what
* Call `constant` contract functions (since they do not change state)

Under the hood, a node is part of the Ethereum network of miners and gets notified when new blocks are mined.  A node contains the full blockchain.

When developing, you'll use TestRPC -- it's a fake node that pretends blocks are being mined and that contains its own blockchain.  Very handy.

## <a name="da-f"></a>FrontEnd
This is plain old JS that does HTTP calls to some Node, and uses the results to show you some UI about what is in the blockchain.

Depending on the browser and what extensions exist, the UI can include prompting the user to create a transaction with specific details, such as "call this contract function with these params passing this much wei with this gas limit and this gas price".

This will cause MetaMask and Mist to show a UI where the user can confirm the transaction.

### MetaMask Absurdity

When the frontend creates a transaction for metamask to confirm, MetaMask will show you NONE OF the following information:

* full address (or link to it on etherscan)
* contract function name
* function params

It's basically like "hey this website wanted to do this transaction which I won't tell you anything about.  You cool with that?"

I haven't used Mist so I don't know if it's as stupid.

# <a name="ic"></a>ABIs and Interacting With Contracts

When contracts are deployed to the blockchain, they contain ZERO informatoin about how they can be interacted with.  Basically, they are gigantic blobs of data, and you have to invoke them _just right_ to get the results you want.

IMO this is a sadistic non-user-friendly approach, but I'm sure there are "good" reasons such as shaving a few KB per block or a few microseconds off mining time.

At any rate, in order to properly interact with contracts, one needs to know the ABI.  The ABI includes all available functions (and their parameter types) so that valid calls can be made.  Think of an ABI as a schema to what is inside the contract.

Under the hood, a client will convert what the user intends to do into the `data` portion of a transaction.  The full specs on this can be found in Ethereum docs somewhere... but you really don't want to know.


# <a name="sc"></a>Solidity

## <a name="sc-g"></a>Gotchas

* Good luck with strings.
* No floating point.  You never really do `a/b`.  Instead, you do `(someValue * a)/b`.  It's fucking stupid.
* External Transactions can not receive return values.  Yes, seriously.
* Functions cannot return dynamic arrays.

## <a name="sc-s"></a>Sending Messages (Internal Transactions)

In the below example, `addr` is the recipient address.

---

`var _success = addr.send(valueInWei)`

* Calls fallback function if `addr` is a contract
* Returns boolean for success or not
* Sends small amount of gas (2300)

---

`addr.transfer(valueInWei)`

* Calls fallback function if `addr` is a contract
* throws on failure
* Sends small amount of gas (2300)

---

`var _succees = addr.call.value(valueInWei).gas(uint)();`

* Calls fallback function if `addr` is a contract
* Returns boolean for success or not
* Sends custom amount of gas (defaults to 0: unlimited)
* Sends custom value

---

`var _returnedValue = addr.someFunction(param1, param2,...);`

* Returns whatever the function returns
* Sends unlimited gas
* No value
* !! Note:  Solidity must know the type of `addr` and that it has `payable someFunction`

---

`var _returnedValue = addr.someFunction.value(valueInWei).gas(uint)(param1, param2, ...);`

* Returns whatever the function returns 
* Sends custom amount of gas (defaults to 0: unlimited)
* Sends custom value
* !! Note:  Solidity must know the type of `addr` and that it has `payable someFunction`

---

There is some other version of `addr.call` where you hash the signature of the method.  Look it up in the docs.



# Web / truffle-contract

Truffle-contract is basically Web3, but returns promises.  Not sure what else it does.

## Transactions

Most everything you deal with will be a transaction, and will in some way call `sendTransaction`.  [Web3 docs can be found here](https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethsendtransaction).

Here's how it is called.  Most of the `myContract.doStuff` will simply fill in the proper values to fields (like `data` and `to`)

Example:

```
var options = {
	// String - address this is from (and will be signed by)
	from: "0xabc...",
	// String - address of who its to
	to: "0xdef...",
	// Number|String|BigNumber - how much wei to send
	value: 1000,
	// Number|String|BigNumber - maximum gas to be used
	gas: 1e10,
	// Number|String|BigNumber - price of gas, defaults to mean network gas price
	gasPrice: 22e12 // (22 gwei),
	// String - optional byte string of contract call with params, or contract creation code
	data: "0x3832...",
	// Number - allows you to overwrite your own pending transactions
	nonce: 123
}
var txHash = web3.eth.sendTransaction(options);
```

Web3 returns a txHash... I'm not sure if it's pre-mined or post-mined.

truffle-contract returns a promise fulfilled with an object:

```
{
	tx: "0x123...",
	receipt: {
		transactionHash: '0x4b0cb3d24b374b27eb05dda5343e3435208c18171774afdd0c7147b9c80894cf',
    	transactionIndex: 0,
    	blockHash: '0x4513c0bb872b86f5c741d6b71f4373d7fda94ee4b865ce39423c94e29d03b3c5',
    	blockNumber: 2556,
    	gasUsed: 830927,
    	cumulativeGasUsed: 830927,
    	contractAddress: null,
    	logs: [ [Object], [Object], [Object] ]
	}
	
	// Only when called via a contract instance
	// These will be logs specific to this object.
	logs: [{ ...log1... }, { ...log2... }],
}
```

## Events

### Watching block events

[Official docs on filters.](https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethfilter)  These docs are pretty good.

Todo: put code samples.

### Watching contract instance events

[Official event docs here.](https://github.com/ethereum/wiki/wiki/JavaScript-API#contract-events)  These docs are pretty good.

You can watch for events on a specific object instance.

Use either `instance.<EventName>(eventFilter, filterOpts)` or `instance.allEvents(filterOps);`.  They both return the same thing.

```
// a filter object.  see above section for details
// note:  I'm not sure what the default fromBlock is
var filterOpts = { ... };

// an object whose keys match the args of events
// and whose values will be used to filter
var eventFilter = {
	arg1: "value must match this",
	arg2: ["can be this", "or this"]
};
var eventFilter = null;

var watcherForEventName = myContract.EventName(eventFilter, filterOpts);
var watcherForAll = myContract.allEvents(filterOpts);
console.log(watcherForEventName.get());

/*
logs the following in truffle-contract:
[
	{
		logIndex: 0,
    	transactionIndex: 0,
    	transactionHash: '0x36caf6b32b87a5145581df63d02fb02857d8935977721d324e8daadb6f2663f3',
   		blockHash: '0x6cec9e3a2ec6972291f71650758001e258f3f973b7fc10a0f81de24b468306f6',
    	blockNumber: 2168,
    	address: '0x8f7d11d92a76d10107f14f42142c4409e2e0a37c',
    	type: 'mined',
    	event: 'EventName',
    	args: {
    		arg1: <value>,
    		arg2: <value>
    	}
    }, {...}    
]
*/

```

## Functions

# <a name="tc"></a>Truffle

## <a name="tc-n"></a>Notes

* Using node8 async/await will make your life much more enjoyable, both in deploy and in tests.

```
module.exports = function(deployer, network, accounts) {
	deployer.then(async function(){
		console.log("Deploying first thing...");
		await deployer.deploy(FirstContract);
		firstContract = deployer.at(FirstContract.address);
		
		console.log("Deploying the second thing...");
		await deployer.deploy(SecondContract);
		...
```

## <a name="tc-c"></a>Compiling / Building

* If you edit a file `A.sol` that is imported by file `B.sol`, `truffle compile` will not re-compile `B.sol`.  Just get in the habit of using `truffle compile --all`.
* If you experience a `solc` exception about `5 5`, it's because there is a syntax error somewhere in _any_ of your files.  You're fucked.
* If you are using my fork of `truffle-compile` you can do `truffle compile --parse` and you'll see any syntax errors on files about to be parsed.  (This feature is currently a pull request)

## <a name="tc-d"></a>Deploying / Migrating

* You cannot return a promise from your deployment function -- it is ignored.  Truffle will consider a single migration completed as soon as `deployer` promise chain is finished.  Consider putting everything inside a deployer.then():  

```
// incorrect -- truffle will continue to next deployment because it thinks this is finished
module.exports = async function(deployer, network, accounts) {
	await deployer.deploy(SomeContract);
	// as soon as the last deployer.deploy() or .then() is done
	// truffle thinks deployment is done.  anything below will
	// be run _after_... which can screw stuff up.
	var c = SomeContract.at(SomeContract.address);
	await c.doSomeCall();
	
// correct -- truffle will wait for this to finish before continuing
module.exports = function(deployer, network, accounts) {
	// note: async function always returns a promise
	deployer.then(async function(){
		await deployer.deploy(SomeContract);
		var c = SomeContract.at(SomeContract.address);
		await c.doSomeCall();
```

* `deployer.deploy` bug: if you do not pass exact amount of constructors, it will pass none of them.  It does not do validation on this, either. 
 
```
// assume SomeContract takes 3 args
// incorrect - This creates SomeContract with empty args.
deployer.deploy(SomeContract, 5, 5);
// correct
deployer.deploy(SomeContract, 5, 5, 5);
```

* `deployer.deploy` returns a promise fulfilled with _nothing_... not even the address.  To get the address of something deployed:

```
await deployer.deploy(SomeContract);
var c = SomeContract.at(SomeContract.address);

// or using promises
deployer.deploy(SomeContract).then(function(){
	var c = SomeContract.at(SomeContract.address);
})
```

# Testing

## Fast Forwarding TestRPC
A lot of contracts you write might be time based, and so you may wish to "fast forward" testrpc to ensure your contracts work as designed.  I was surprised how hard I had to search to find out how to do this.

Anyway, here's how to do it using web3:

```
function fastForward(timeInSeconds){
	if (!Number.isInteger(timeInSeconds))
		throw new Error("Passed a non-number: " + timeInSeconds);
	if (timeInSeconds <= 0)
		throw new Error("Can not fastforward a negative amount: " + timeInSeconds);
	
	// move time forward.
	web3.currentProvider.send({
        jsonrpc: "2.0",
        method: "evm_increaseTime",
        params: [timeInSeconds],
        id: new Date().getTime()
    });
	// mine a block to make sure future calls use updated time.
	web3.currentProvider.send({
        jsonrpc: "2.0",
        method: "evm_mine",
        params: null,
        id: new Date().getTime()
    });
}
```

# Patterns

## Upgrading Smart Contracts

I personally have a deployed `Registry` contract that contains `name=>address` mappings for singleton contracts.  When my contracts need to talk to one another, they _always_ ask the registry first.

When I need to upgrade a singleton, I deploy a new instance, and upgrade the registry name for it.  Since all my contracts always ask the registry for the latest name, no further action should be necessary.

However, this only works because my singleton contracts do not store a lot of state, and when I deploy new versions I copy the state over.

If you require copying state over that is huge or unknown (eg, mappings), you will have to use the `Relay` (aka `delegatecall`) pattern.

[For more on upgrading contracts, go here](https://github.com/ConsenSys/smart-contract-best-practices#eng-techniques)

## Handling errors:  Returning vs Throwing

When you `throw` in Solidity, nothing is returned.  No error message.  No trace.  Nothing.  This is fucking stupid, and makes testing things a nightmare.

For example:

```
contract Foo {
	function doStuff(){
		require(msg.sender == "0xABC...");
		require(msg.value > 100);
		... other code ...
	}
}
```


```
it("fails when wrong amount is sent", function(done){
	myFoo.doStuff({from: "0xBCA...", 90})
		.then(function(){ done("We expected this call to fail"); )
		.catch(done);
});
```

This test will pass -- but for the wrong reason.  It failed because you passed the wrong address.  You never really tested that it fails because the wrong amount was sent.

[Anyway, read more about returning vs throwing here.](https://ethereum.stackexchange.com/questions/10046/throw-vs-return)
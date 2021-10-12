# Full Stack Ethereum Workshop

## The pieces

Let's go over the main pieces we will be using and how they fit into the stack.

### 1. Ethereum development environment

When building smart contracts, you will need a way to deploy your contracts, run tests, and debug Solidity code without dealing with live environments.

You will also need a way to compile your Solidity code into code that can be run in a client-side application – in our case, a React app. We'll learn more about how this works a little later.

Hardhat is an Ethereum development environment and framework designed for full stack development and is the framework that I will be using for this tutorial.

Other similar tools in the ecosystem are [Ganache](https://www.trufflesuite.com/ganache) and [Truffle](https://www.trufflesuite.com/).

### 2. Ethereum Web Client Library

In our React app, we will need a way to interact with the smart contracts that have been deployed. We will need a way to read for data as well as send new transactions. 

[ethers.js](https://docs.ethers.io/v5/) aims to be a complete and compact library for interacting with the Ethereum Blockchain and its ecosystem from client-side JavaScript applications like React, Vue, Angular, or Svelte. It is the library we'll be using.

Another popular option in the ecosystem is [web3.js](https://web3js.readthedocs.io/en/v1.3.4/)

### 3. Metamask

[Metamask](https://metamask.io/download.html) helps to handle account management and connecting the current user to the blockchain. MetaMask enables users to manage their accounts and keys in a few different ways while isolating them from the site context.

Once a user has connected their MetaMask wallet, you as a developer can interact with the globally available Ethereum API (`window.ethereum`) that identifies the users of web3-compatible browsers (like MetaMask users), and whenever you request a transaction signature, MetaMask will prompt the user in as comprehensible a way as possible.

### 4. React

React is a front end JavaScript library for building web applications, user interfaces, and UI components. It's maintained by Facebook and many many individual developers and companies.

React and its large ecosystem of metaframeworks like [Next.js](https://nextjs.org/), [Gatsby](https://www.gatsbyjs.com/), [Redwood](https://redwoodjs.com/), [Blitz.js](https://blitzjs.com/), and others enable all types of deployment targets including traditional SPAs, static site generators, server-side rendering, and a combination of all three. React continues to be seemingly dominating the front-end space and I think will continue to do so for at least the near future.

## What we will be building

In this workshop, we'll be building, deploying, and connecting to a couple of basic smart contracts: 

1. A contract for creating and updating a message on the Ethereum blockchain
2. A contract for minting tokens, then allowing the owner of the contract to send tokens to others and to read the token balances, and for owners of the new tokens to also send them to others.

We will also build out a React front end that will allow a user to:

1. Read the greeting from the contract deployed to the blockchain
2. Update the greeting
3. Send the newly minted tokens from their address to another address
4. Once someone has received tokens, allow them to also send their tokens to someone else
5. Read the token balance from the contract deployed to the blockchain

### Prerequisites

1. Node.js installed on your local machine
2. [MetaMask](https://metamask.io/) Chrome extension installed in your browser

You do not need to own any Ethereum for this workshop as we will be using test Ether on a test network for the entire tutorial.

## Getting started

To get started, we'll create a new React application:

```sh
npx create-react-app react-dapp
```

Next, change into the new directory and install [`ethers.js`](https://docs.ethers.io/v5/) and [`hardhat`](https://github.com/nomiclabs/hardhat) using either __NPM__ or __Yarn__:

```
npm install ethers hardhat @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers
```

### Installing & configuring an Ethereum development environment

Next, initialize a new Ethereum Development Environment with Hardhat:

```sh
npx hardhat

? What do you want to do? Create a sample project
? Hardhat project root: <Choose default path>
```

Now you should see the following artifacts created for you in your root directory:

__hardhat.config.js__ - The entirety of your Hardhat setup (i.e. your config, plugins, and custom tasks) is contained in this file.
__scripts__ - A folder containing a script named __sample-script.js__ that will deploy your smart contract when executed
__test__ - A folder containing an example testing script
__contracts__ - A folder holding an example Solidity smart contract

Because of [a MetaMask configuration issue](https://hardhat.org/metamask-issue.html), we need to update the chain ID on our HardHat configuration to be __1337__. We also need to update the location for the [artifacts](https://hardhat.org/guides/compile-contracts.html#artifacts) for our compiled contracts to be in the __src__ directory of our React app.

To make these updates, open __hardhat.config.js__ and update the `module.exports` to look like this:

```javascript
module.exports = {
  solidity: "0.8.4",
  paths: {
    artifacts: './src/artifacts',
  },
  networks: {
    hardhat: {
      chainId: 1337
    }
  }
};
```

## Our smart contract

Next, let's have a look at the example contract given to us at __contracts/Greeter.sol__:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "hardhat/console.sol";


contract Greeter {
  string greeting;

  constructor(string memory _greeting) {
    console.log("Deploying a Greeter with greeting:", _greeting);
    greeting = _greeting;
  }

  function greet() public view returns (string memory) {
    return greeting;
  }

  function setGreeting(string memory _greeting) public {
    console.log("Changing greeting from '%s' to '%s'", greeting, _greeting);
    greeting = _greeting;
  }
}
```

This is a very basic smart contract. When deployed, it sets a Greeting variable and exposes a function (`greet`) that can be called to return the greeting.

It also exposes a function that allows a user to update the greeting (`setGreeting`). When deployed to the Ethereum blockchain, these methods will be available for a user to interact with.

### Reading and writing to the Ethereum blockchain

There are two types of ways to interact with a smart contract, reading or writing / transactions. In our contract, `greet` can be considered reading, and `setGreeting` can be considered writing / transactional.

When writing or initializing a transaction, you have to pay for the transaction to be written to the blockchain. To make this work, you need to pay [gas](https://www.investopedia.com/terms/g/gas-ethereum.asp#:~:text=What%20Is%20Gas%20(Ethereum)%3F,on%20the%20Ethereum%20blockchain%20platform) which is the fee, or price, required to successfully conduct a transaction and execute a contract on the Ethereum blockchain.

As long as you are only reading from the blockchain and not changing or updating anything, you don't need to carry out a transaction and there will be no gas or cost to do so. The function you call is then carried out only by the node you are connected to, so you don't need to pay any gas and the read is free.

From our React app, the way that we will interact with the smart contract is using a combination of the `ethers.js` library, the contract address, and the [ABI](https://docs.soliditylang.org/en/v0.5.3/abi-spec.html) that will be created from the contract by hardhat.

What is an ABI? ABI stands for application binary interface. You can think of it as the interface between your client-side application and the Ethereum blockchain where the smart contract you are going to be interacting with is deployed.

ABIs are typically compiled from Solidity smart contracts by a development framework like HardHat. You can also often find the ABIs for a smart contract on [Etherscan](https://etherscan.io/)

### Compiling the ABI

Now that we have gone over the basic smart contract and know what ABIs are, let's compile an ABI for our project.

To do so, go to the command line and run the following command:

```sh
npx hardhat compile
```

Now, you should see a new folder named __artifacts__ in the __src__ directory. The __artifacts/contracts/Greeter.json__ file contains the ABI as one of the properties. When we need to use the ABI, we can import it from our JavaScript file:

```javascript
import Greeter from './artifacts/contracts/Greeter.sol/Greeter.json'
```

We can then reference the ABI like this:

```
console.log("Greeter ABI: ", Greeter.abi)
```

> Note that Ethers.js also enables [human readable ABIs](https://blog.ricmoo.com/human-readable-contract-abis-in-ethers-js-141902f4d917), but will will not be going into this during this tutorial.

### Deploying and using a local network / blockchain

Next, let's deploy our smart contract to a local blockchain so that we can test it out.

To deploy to the local network, you first need to start the local test node. To do so, open the CLI and run the following command:

```sh
npx hardhat node
```

When we run this command, you should see a list of addresses and private keys. 

![Hardhat node addresses](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e176nc82ik77hei3a48s.jpg)

These are 20 test accounts and addresses created for us that we can use to deploy and test our smart contracts. Each account is also loaded up with 10,000 fake Ether. In a moment, we'll learn how to import the test  account into MetaMask so that we can use it.

Next, we need to deploy the contract to the test network. First update the name of __scripts/sample-script.js__ to __scripts/deploy.js__.

Now we can run the deploy script and give a flag to the CLI that we would like to deploy to our local network:

```sh
npx hardhat run scripts/deploy.js --network localhost
```

Once this script is executed, the smart contract should be deployed to the local test network and we should be then able to start interacting with it.

> When the contract was deployed, it used the first account that was created when we started the local network.

If you look at the output from the CLI, you should be able to see something like this:

 ```sh
Greeter deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
```

This address is what we will use in our client application to talk to the smart contract. Keep this address available as we will need to use it when connecting to it from the client application.

To send transactions to the smart contract, we will need to connect our MetaMask wallet using one of the accounts created when we ran `npx hardhat node`. In the list of contracts that the CLI logs out, you should see both an __Account number__ as well as a __Private Key__:

```bash
➜  react-dapp git:(main) npx hardhat node
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/

Accounts
========
Account #0: 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 (10000 ETH)
Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

...
```

We can import this account into MetaMask in order to start using some of the fake Eth available there. To do so, first open MetaMask and update the network to be Localhost 8545:

![MetaMask Localhost](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qnbsbcm4y1md6cwjttpx.jpg)

Next, in MetaMask click on __Import Account__ from the accounts menu:

![Import account](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n7vbzlov869gwk9rtwl1.jpg)

Copy then paste one of the __Private Keys__ logged out by the CLI and click __Import__. Once the account is imported, you should see the Eth in the account:

![Imported account](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x5lob4yug3jznhy9z0qt.jpg)

Now that we have a smart contract deployed and an account ready to use, we can start interacting with it from the React app.

### Connecting the React client

In this tutorial we are not going to be worrying about building a beautiful UI with CSS and all of that, we are focused 100% on the core functionality to get you up and running. From there, you can take it and make it look good if you'd like.

With that being said, let's review the two objectives that we want from our React application:

1. Fetch the current value of `greeting` from the smart contract
2. Allow a user to update the value of the `greeting`

With those things being understood, how do we accomplish this? Here are the things we need to do to make this happen:

1. Create an input field and some local state to manage the value of the input (to update the `greeting`)
2. Allow the application to connect to the user's MetaMask account to sign transactions
3. Create functions for reading and writing to the smart contract

To do this, open `src/App.js` and update it with the following code, setting the value of `greeterAddress` to the address of your smart contract.:

```js
import './App.css';
import { useState } from 'react';
import { ethers } from 'ethers'
import Greeter from './artifacts/contracts/Greeter.sol/Greeter.json'

// Update with the contract address logged out to the CLI when it was deployed 
const greeterAddress = "your-contract-address"

function App() {
  // store greeting in local state
  const [greeting, setGreetingValue] = useState()

  // request access to the user's MetaMask account
  async function requestAccount() {
    await window.ethereum.request({ method: 'eth_requestAccounts' });
  }

  // call the smart contract, read the current greeting value
  async function fetchGreeting() {
    if (typeof window.ethereum !== 'undefined') {
      const provider = new ethers.providers.Web3Provider(window.ethereum)
      const contract = new ethers.Contract(greeterAddress, Greeter.abi, provider)
      try {
        const data = await contract.greet()
        console.log('data: ', data)
      } catch (err) {
        console.log("Error: ", err)
      }
    }    
  }

  // call the smart contract, send an update
  async function setGreeting() {
    if (!greeting) return
    if (typeof window.ethereum !== 'undefined') {
      await requestAccount()
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const signer = provider.getSigner()
      const contract = new ethers.Contract(greeterAddress, Greeter.abi, signer)
      const transaction = await contract.setGreeting(greeting)
      await transaction.wait()
      fetchGreeting()
    }
  }

  return (
    <div className="App">
      <header className="App-header">
        <button onClick={fetchGreeting}>Fetch Greeting</button>
        <button onClick={setGreeting}>Set Greeting</button>
        <input onChange={e => setGreetingValue(e.target.value)} placeholder="Set greeting" />
      </header>
    </div>
  );
}

export default App;
```

To test it out, start the React server:

```sh
npm start
```

When the app loads, you should be able to fetch the current greeting and log it out to the console. You should also be able to make updates to the greeting by signing the contract with your MetaMask wallet and spending the fake Ether.

![Setting and getting the greeting value](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9a57jbzrwylr2l0rujxm.png)

## Minting tokens

One of the most common use cases of smart contracts is creating tokens, let's look at how we can do that. Since we know a little more about how all of this works, we'll be going a little faster.

In the main __contracts__ directory create a new file named __Token.sol__.

Next, update __Token.sol__ with the following smart contract:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "hardhat/console.sol";

contract Token {
  string public name = "Lazy Cats Token";
  string public symbol = "LCT";
  uint public totalSupply = 1000000;
  mapping(address => uint) balances;

  constructor() {
    balances[msg.sender] = totalSupply;
  }

  function transfer(address to, uint amount) external {
    require(balances[msg.sender] >= amount, "Not enough tokens");
    balances[msg.sender] -= amount;
    balances[to] += amount;
  }

  function balanceOf(address account) external view returns (uint) {
    return balances[account];
  }
}
```

> Note that this token contract is for demo purposes only and is not [ERC20](https://eips.ethereum.org/EIPS/eip-20) compliant. We will be covering ERC20 tokens later.

This contract will create a new token called "Lazy Cats Token" and set the supply to 1000000.

Next, compile this contract:

```sh
npx hardhat compile
```

Now, update the deploy script at __scripts/deploy.js__ to include this new Token contract:

```javascript
const hre = require("hardhat");

async function main() {
  const [deployer] = await hre.ethers.getSigners();

  console.log(
    "Deploying contracts with the account:",
    deployer.address
  );
  
  const Greeter = await hre.ethers.getContractFactory("Greeter");
  const greeter = await Greeter.deploy("Hello, World!");

  const Token = await hre.ethers.getContractFactory("Token");
  const token = await Token.deploy();
  
  await greeter.deployed();
  await token.deployed();

  console.log("Greeter deployed to:", greeter.address);
  console.log("Token deployed to:", token.address);
}

main()
  .then(() => process.exit(0))
  .catch(error => {
    console.error(error);
    process.exit(1);
  });
```

Now, we can deploy this new contract to the local or Ropsten network:

```sh
npx hardhat run scripts/deploy.js --network localhost
```

Once the contract is deployed, you can start sending these tokens to other addresses.

To do so, let's update the client code we will need in order to make this work:

```javascript
import './App.css';
import { useState } from 'react';
import { ethers } from 'ethers'
import Greeter from './artifacts/contracts/Greeter.sol/Greeter.json'
/* add token import */
import Token from './artifacts/contracts/Token.sol/Token.json'

const greeterAddress = "your-contract-address"
/* add token address */
const tokenAddress = "your-contract-address"

function App() {
  const [greeting, setGreetingValue] = useState()
  /* add new state variables */
  const [userAccount, setUserAccount] = useState()
  const [amount, setAmount] = useState()

  async function requestAccount() {
    await window.ethereum.request({ method: 'eth_requestAccounts' });
  }

  async function fetchGreeting() {
    if (typeof window.ethereum !== 'undefined') {
      const provider = new ethers.providers.Web3Provider(window.ethereum)
      console.log({ provider })
      const contract = new ethers.Contract(greeterAddress, Greeter.abi, provider)
      try {
        const data = await contract.greet()
        console.log('data: ', data)
      } catch (err) {
        console.log("Error: ", err)
      }
    }    
  }

  /* add function to fetch token balance */
  async function getBalance() {
    if (typeof window.ethereum !== 'undefined') {
      const [account] = await window.ethereum.request({ method: 'eth_requestAccounts' })
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const contract = new ethers.Contract(tokenAddress, Token.abi, provider)
      const balance = await contract.balanceOf(account);
      console.log("Balance: ", balance.toString());
    }
  }

  async function setGreeting() {
    if (!greeting) return
    if (typeof window.ethereum !== 'undefined') {
      await requestAccount()
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      console.log({ provider })
      const signer = provider.getSigner()
      const contract = new ethers.Contract(greeterAddress, Greeter.abi, signer)
      const transaction = await contract.setGreeting(greeting)
      await transaction.wait()
      fetchGreeting()
    }
  }

  /* add function to send tokens */
  async function sendCoins() {
    if (typeof window.ethereum !== 'undefined') {
      await requestAccount()
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const signer = provider.getSigner();
      const contract = new ethers.Contract(tokenAddress, Token.abi, signer);
      const transation = await contract.transfer(userAccount, amount);
      await transation.wait();
      console.log(`${amount} Coins successfully sent to ${userAccount}`);
    }
  }

  return (
    <div className="App">
      <header className="App-header">
        <button onClick={fetchGreeting}>Fetch Greeting</button>
        <button onClick={setGreeting}>Set Greeting</button>
        <input onChange={e => setGreetingValue(e.target.value)} placeholder="Set greeting" />

        <br />
        {/* add new ui for token exchange */}
        <button onClick={getBalance}>Get Balance</button>
        <button onClick={sendCoins}>Send Coins</button>
        <input onChange={e => setUserAccount(e.target.value)} placeholder="Account ID" />
        <input onChange={e => setAmount(e.target.value)} placeholder="Amount" />
      </header>
    </div>
  );
}

export default App;
```

Next, run the app:

```sh
npm start
```

We should be able to click on __Get Balance__ and see that we have 1,000,000 coins in our account logged out to the console.

You should also be able to view them in MetaMask by clicking on __Add Token__:

![Add token](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0t2ip26i5d2ltjc9j2a6.jpg)

Next click on __Custom Token__ and enter the token contract address and then __Add Token__. (if asked for token decimals, choose 0) Now the tokens should be available in your wallet:

![NDT](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5op32iqbeszizri72qc0.jpg)

Next, let's try to send those coins to another address using our app.

To do so, copy the address of another account and send them to that address using the updated React UI. When you check the token amount, it should be equal to the original amount minus the amount you sent to the address.

## ERC20 Token

The [ERC20 Token Standard](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) defines a set of rules that apply to all ERC20 tokens which allow them to easily interact with each other. ERC20 makes it really easy for someone to mint their own tokens that will have interoperability with others on the Ethereum blockchain.

Let's look at how we may build our own token using the ERC20 standard.

First, install the [OpenZepplin](https://github.com/OpenZeppelin/openzeppelin-contracts) smart contract library where we will be importing the base `ERC20` Token:

```sh
npm install @openzeppelin/contracts
```

Next, we'll create our token by extending (or inheriting from) the `ERC20` contract:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract NDToken is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _mint(msg.sender, 100000 * (10 ** 18));
    }
}
```

The constructor allows you to set the token name and symbol, and the `_mint` function allows you to mint the tokens and set the amount.

By default, ERC20 sets the number of decimals to 18, so in our `_mint` function we multiply 100,000 by 10 to the 18 power to mint a total of 100,000 tokens, each with 18 decimal places (similarly to how 1 Eth is made up of 10 to the 18 [wei](https://www.investopedia.com/terms/w/wei.asp).

To deploy, we need to pass in the constructor values (`name` and `symbol`), so we might do something like this in our deploy script:

```javascript
const NDToken = await hre.ethers.getContractFactory("NDToken");
const ndToken = await NDToken.deploy("Lazy Cats Token", "LCT");
```

By extending the original ERC20 token, your token will inherit all of the following functions and functionality:

```solidity
function name() public view returns (string)
function symbol() public view returns (string)
function decimals() public view returns (uint8)
function totalSupply() public view returns (uint256)
function balanceOf(address _owner) public view returns (uint256 balance)
function transfer(address _to, uint256 _value) public returns (bool success)
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
function approve(address _spender, uint256 _value) public returns (bool success)
function allowance(address _owner, address _spender) public view returns (uint256 remaining)
```

Once deployed, you can use any of these functions to interact with the new smart contract. For another example of an ERC20 token, check out [Solidity by example)(https://solidity-by-example.org/app/erc20/)

## Challenge

Update your app to use real ERC20 tokens using the OpenZeppelin standard and the `transfer` and `balanceOf` functions.

## Next steps

If you'd like to learn more, here are some good resources.

- [Building a Full Stack NFT Marketplace on Ethereum with Polygon](https://dev.to/dabit3/building-scalable-full-stack-apps-on-ethereum-with-polygon-2cfb)
- [BuildSpace](https://buildspace.so/)
- [Ethereum Dev Speerun](https://medium.com/@austin_48503/%EF%B8%8Fethereum-dev-speed-run-bd72bcba6a4c)
- [UseWeb3](https://www.useweb3.xyz/)
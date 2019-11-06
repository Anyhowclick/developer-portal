---
id: Integrations-RESTfulAPIGuide
title: RESTful API
---
[//]: # (tagline)
## Introduction
This guide will walk you through on how you can interact with our protocol implementation using our RESTful APIs. The most common group of users that can benefit from this guide are developers who have minimal smart contract experience, traders and wallets.

## Risk Mitigation
There are some risks when utilising Kyber. To safeguard users, we kindly ask that you refer to the [Slippage Rates Protection](integrations-slippagerateprotection.md) and [Price Feed Security](integrations-pricefeedsecurity.md) sections on what these risks are, and how to mitigate them.

## Overview
In this guide, we will be going through 2 scenarios. The first scenario covers how to perform a token to token swap using the RESTful apis and the second is about how one can obtain token information and historical price data.

## Things to note
1) When converting from Token to ETH/Token, the user is required to call the `/enabled_data` endpoint **first** to give an allowance to the smart contract executing the `trade` function i.e. the `KyberNetworkProxy.sol` contract.
2) Refer to the [API overview](api_abi-restfulapioverview.md#network-url) for the test and mainnet network URLs to use.

## Scenario 1: Token to Token Swap
Suppose we want to convert 100 BAT to DAI tokens, which is a token to token conversion. Note that ETH is used as the base pair i.e. BAT -> ETH -> DAI.

### Import relevant packages
* We use `web3` for broadcasting transactions to the blockchain
* The `ethereumjs-tx` library is used to sign and serialize a raw transaction to be broadcasted
* The `node-fetch` module is used for making API queries

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

// Importing the relevant packages
const Web3 = require("web3");
const Tx = require("ethereumjs-tx").Transaction;
const fetch = require('node-fetch');
```

### Connect to an Ethereum node
In this example, we will connect to Infura's ropsten node.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

// Connecting to ropsten infura node
const NETWORK = "ropsten"
const PROJECT_ID = "ENTER_PROJECT_ID" //Replace this with your own Project ID
const WS_PROVIDER = `wss://${NETWORK}.infura.io/ws/v3/${PROJECT_ID}`
const web3 = new Web3(new Web3.providers.WebsocketProvider(WS_PROVIDER));
```

### Define constants
Next, we will define the constants that we will be using for this guide.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

//Base URL for API queries
//Refer to API/ABI >> RESTFul API Overview >> Network URL section
const NETWORK_URL = `https://${NETWORK}-api.kyber.network`;

//User Details
const PRIVATE_KEY = Buffer.from("ENTER_USER_PRIVATE_KEY", "hex"); // Remove the 0x prefix
const USER_ADDRESS = web3.eth.accounts.privateKeyToAccount("0x" + PRIVATE_KEY.toString('hex')).address;

// Wallet Address for Fee Sharing Program
const REF_ADDRESS = "0x483C5100C3E544Aef546f72dF4022c8934a6945E";

//Token Addresses
const BAT_TOKEN_ADDRESS = "0xDb0040451F373949A4Be60dcd7b6B8D6E42658B6"; //Ropsten BAT token address
const DAI_TOKEN_ADDRESS = "0xaD6D458402F60fD3Bd25163575031ACDce07538D"; //Ropsten DAI token address

//Token Quantity
const BAT_QTY = 100; //100 BAT tokens to swap from

//Gas amount affecting speed of tx
const GAS_PRICE = "medium";
```

### Define function for broadcasting transactions
We will refactor the broadcast transaction functionality into its own function.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

async function broadcastTx(rawTx) {
  // Extract raw tx details, create a new Tx
  let tx = new Tx(rawTx, { chain: NETWORK, hardfork: 'petersburg' });
  // Sign the transaction
  tx.sign(PRIVATE_KEY);
  // Serialize the transaction (RLP Encoding)
  const serializedTx = tx.serialize();
  // Broadcast the tx
  txReceipt = await web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex')).catch(error => console.log(error));
  // Log the tx receipt
  console.log(txReceipt);
}
```

### Check if BAT and DAI tokens are supported
Create a function to check whether a token is supported on Kyber. We make use of the `/currencies` endpoint, which returns basic information about all tokens supported on Kyber. Details about possible path parameters and output fields can be [found here](api_abi-restfulapi.md#currencies).

It is recommended to use the token contract address as the identifier instead of the token symbol, as multiple tokens may share the same symbol.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

function caseInsensitiveEquals(a, b) {
  return typeof a === 'string' && typeof b === 'string' ?
    a.localeCompare(b, undefined, {
      sensitivity: 'accent'
    }) === 0 :
    a === b;
}

async function isTokenSupported(tokenAddress) {
  let tokensBasicInfoRequest = await fetch(`${NETWORK_URL}/currencies`);
  let tokensBasicInfo = await tokensBasicInfoRequest.json();
  let tokenSupported = tokensBasicInfo.data.some(token => {
    return caseInsensitiveEquals(tokenAddress, token.id)
  });
  if (!tokenSupported) {
    console.log('Token is not supported');
  }
  return tokenSupported;
}
```

### Check and Approve Bat token contract
We use the `/users/<user_address>/currencies` endpoint to check whether the KyberNetwork contract has been approved for selling BAT tokens on behalf of the user. This endpoints returns a JSON of enabled statuses of ERC20 tokens for the given walletAddress. Details about the path parameters and output fields can be [found here](api_abi-restfulapi.md#users-user-address-currencies).

If the BAT token is not enabled for trading, querying the `users/<user_address>/currencies/<currency_id>/enable_data?gas_price=<gas_price>` endpoint returns a transaction payload needed to be signed and broadcasted by the user to enable the KyberNetwork contract to trade BAT tokens on his behalf. Details about the path parameters and output fields can be [found here](api_abi-restfulapi.md#users-user-address-currencies-currency-id-enable-data).

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

async function checkAndApproveTokenContract(tokenAddress, userAddress, gasPrice) {
  let enabledStatusesRequest = await fetch(`${NETWORK_URL}/users/${userAddress}/currencies`);
  let enabledStatuses = await enabledStatusesRequest.json();
  let txsRequired = 0;
  for (let token of enabledStatuses.data) {
    if (caseInsensitiveEquals(tokenAddress, token.id)) {
      txsRequired = token.txs_required;
      break;
    }
  }
  switch (txsRequired) {
    case 1:
      console.log("Approving to max amount");
      // No allowance so approve to maximum amount (2^255)
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      break;
    case 2:
      // Allowance has been given but is insufficient.
      // Have to approve to 0 first to avoid this issue https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
      // Approve to 0
      console.log("Approving to 0");
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      // Approve to maximum amount (2^255)
      console.log("Approving to max amount");
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      break;
    default:
      // Shouldn't need to do anything else in other scenarios.
      break;
  }
}

async function enableTokenTransfer(tokenAddress, userAddress, gasPrice) {
  let enableTokenDetailsRequest = await fetch(`${NETWORK_URL}/users/${userAddress}/currencies/${tokenAddress}/enable_data?gas_price=${gasPrice}`);
  let enableTokenDetails = await enableTokenDetailsRequest.json();
  let rawTx = enableTokenDetails.data;
  await broadcastTx(rawTx);
}
```

### Get DAI token amount receivable
Create a function to get an approximate of the destination token amount for the specified amount of source token. We will use the `/quote_amount` endpoint in this function.
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

async function getQuoteAmount(srcToken, destToken, srcQty) {
  let quoteAmountRequest = await fetch(`${NETWORK_URL}/quote_amount?base=${srcToken}&quote=${destToken}&base_amount=${srcQty}&type=sell`)
  let quoteAmount = await quoteAmountRequest.json();
  quoteAmount = quoteAmount.data;
  return quoteAmount * 0.97;
}
```

### Convert BAT to DAI
We now have all the required information to peform the trade transaction. Querying `https://api.kyber.network/trade_data?user_address=<user_address>&src_id=<src_id>&dst_id=<dst_id>&src_qty=<src_qty>&min_dst_qty=<min_dst_qty>&gas_price=<gas_price>&wallet_id=<wallet_id>&only_official_reserve=false` will return the transaction payload to be signed and broadcasted by the user to make the conversion. Details about the path parameters and output fields can be [found here](api_abi-restfulapi.md#trade-data).

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

async function executeTrade(useraddress, srcToken, dstToken, srcQty, minDstQty, gasPrice, refAddress) {
  let tradeDetailsRequest = await fetch(`${NETWORK_URL}/trade_data?user_address=${useraddress}&src_id=${srcToken}&dst_id=${dstToken}&src_qty=${srcQty}&min_dst_qty=${minDstQty}&gas_price=${gasPrice}&wallet_id=${refAddress}`);
  let tradeDetails = await tradeDetailsRequest.json();
  let rawTx = tradeDetails.data[0];
  await broadcastTx(rawTx);
}
```

### Define Main function
The main function will combine the different functions together to obtain the conversion rate, check that conditions are met for the trade, and execute the trade.

```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

async function main() {
  //Step 1: If either token is not supported, quit
  if (!await isTokenSupported(BAT_TOKEN_ADDRESS) || !await isTokenSupported(DAI_TOKEN_ADDRESS)) {
    // Quit the program
    process.exit(0);
  }

  //Step 2: Check if BAT token is enabled. If not, enable.
  await checkAndApproveTokenContract(BAT_TOKEN_ADDRESS, USER_ADDRESS, GAS_PRICE)

  // Step 3: Get expected DAI qty from selling 100 BAT tokens
  let minDstQty = await getQuoteAmount(BAT_TOKEN_ADDRESS, DAI_TOKEN_ADDRESS, BAT_QTY);

  //Step 4: Perform the BAT -> DAI trade
  await executeTrade(USER_ADDRESS, BAT_TOKEN_ADDRESS, DAI_TOKEN_ADDRESS, BAT_QTY, minDstQty, GAS_PRICE, REF_ADDRESS);

  // Quit the program
  process.exit(0);
}
```

### Full code example
Before running this code example, the following fields need to be modified:
1. Change `ENTER_PROJECT_ID` to your Infura Project ID.
2. Change `ENTER_USER_PRIVATE_KEY` to the private key (without `0x` prefix) of the Ethereum wallet holding Ether
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

// Importing the relevant packages
const Web3 = require("web3");
const Tx = require("ethereumjs-tx").Transaction;
const fetch = require('node-fetch');

// Connecting to infura node
const NETWORK = "ropsten"
const PROJECT_ID = "ENTER_PROJECT_ID" //Replace this with your own Project ID
const WS_PROVIDER = `wss://${NETWORK}.infura.io/ws/v3/${PROJECT_ID}`
const web3 = new Web3(new Web3.providers.WebsocketProvider(WS_PROVIDER));

//Base URL for API queries
//Refer to API/ABI >> RESTFul API Overview >> Network URL section
const NETWORK_URL = `https://${NETWORK}-api.kyber.network`;

//User Details
const PRIVATE_KEY = Buffer.from("ENTER_USER_PRIVATE_KEY", "hex"); // Remove the 0x prefix
const USER_ADDRESS = web3.eth.accounts.privateKeyToAccount("0x" + PRIVATE_KEY.toString('hex')).address;

// Wallet Address for Fee Sharing Program
const REF_ADDRESS = "0x483C5100C3E544Aef546f72dF4022c8934a6945E";

//Token Addresses
const BAT_TOKEN_ADDRESS = "0xDb0040451F373949A4Be60dcd7b6B8D6E42658B6"; //Ropsten BAT token address
const DAI_TOKEN_ADDRESS = "0xaD6D458402F60fD3Bd25163575031ACDce07538D"; //Ropsten DAI token address

//Token Quantity
const BAT_QTY = 100; //100 BAT tokens to swap from

//Gas amount affecting speed of tx
const GAS_PRICE = "medium";

async function main() {
  //Step 1: If either token is not supported, quit
  if (!await isTokenSupported(BAT_TOKEN_ADDRESS) || !await isTokenSupported(DAI_TOKEN_ADDRESS)) {
    // Quit the program
    process.exit(0);
  }

  //Step 2: Check if BAT token is enabled. If not, enable.
  await checkAndApproveTokenContract(BAT_TOKEN_ADDRESS, USER_ADDRESS, GAS_PRICE)

  // Step 3: Get expected DAI qty from selling 100 BAT tokens
  let minDstQty = await getQuoteAmount(BAT_TOKEN_ADDRESS, DAI_TOKEN_ADDRESS, BAT_QTY);

  //Step 4: Perform the BAT -> DAI trade
  await executeTrade(USER_ADDRESS, BAT_TOKEN_ADDRESS, DAI_TOKEN_ADDRESS, BAT_QTY, minDstQty, GAS_PRICE, REF_ADDRESS);

  // Quit the program
  process.exit(0);
}

function caseInsensitiveEquals(a, b) {
  return typeof a === 'string' && typeof b === 'string' ?
    a.localeCompare(b, undefined, {
      sensitivity: 'accent'
    }) === 0 :
    a === b;
}

async function isTokenSupported(tokenAddress) {
  let tokensBasicInfoRequest = await fetch(`${NETWORK_URL}/currencies`);
  let tokensBasicInfo = await tokensBasicInfoRequest.json();
  let tokenSupported = tokensBasicInfo.data.some(token => {
    return caseInsensitiveEquals(tokenAddress, token.id)
  });
  if (!tokenSupported) {
    console.log('Token is not supported');
  }
  return tokenSupported;
}

async function checkAndApproveTokenContract(tokenAddress, userAddress, gasPrice) {
  let enabledStatusesRequest = await fetch(`${NETWORK_URL}/users/${userAddress}/currencies`);      
  let enabledStatuses = await enabledStatusesRequest.json();
  let txsRequired = 0;
  for (let token of enabledStatuses.data) {
    if (caseInsensitiveEquals(tokenAddress, token.id)) {
      txsRequired = token.txs_required;
      break;
    }
  }
  switch (txsRequired) {
    case 1:
      console.log("Approving to max amount");
      // No allowance so approve to maximum amount (2^255)
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      break;
    case 2:
      // Allowance has been given but is insufficient.
      // Have to approve to 0 first to avoid this issue https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
      // Approve to 0
      console.log("Approving to 0");
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      // Approve to maximum amount (2^255)
      console.log("Approving to max amount");
      await enableTokenTransfer(tokenAddress, userAddress, gasPrice);
      break;
    default:
      // Shouldn't need to do anything else in other scenarios.
      break;
  }
}

async function enableTokenTransfer(tokenAddress, userAddress, gasPrice) {
  let enableTokenDetailsRequest = await fetch(`${NETWORK_URL}/users/${userAddress}/currencies/${tokenAddress}/enable_data?gas_price=${gasPrice}`);
  let enableTokenDetails = await enableTokenDetailsRequest.json();
  let rawTx = enableTokenDetails.data;
  await broadcastTx(rawTx);
}

async function broadcastTx(rawTx) {
  // Extract raw tx details, create a new Tx
  let tx = new Tx(rawTx, { chain: NETWORK, hardfork: 'petersburg' });
  // Sign the transaction
  tx.sign(PRIVATE_KEY);
  // Serialize the transaction (RLP Encoding)
  const serializedTx = tx.serialize();
  // Broadcast the tx
  txReceipt = await web3.eth.sendSignedTransaction('0x' + serializedTx.toString('hex')).catch(error => console.log(error));
  // Log the tx receipt
  console.log(txReceipt);
}

async function getQuoteAmount(srcToken, destToken, srcQty) {
  let quoteAmountRequest = await fetch(`${NETWORK_URL}/quote_amount?base=${srcToken}&quote=${destToken}&base_amount=${srcQty}&type=sell`)
  let quoteAmount = await quoteAmountRequest.json();
  quoteAmount = quoteAmount.data;
  return quoteAmount * 0.97;
}

async function executeTrade(useraddress, srcToken, dstToken, srcQty, minDstQty, gasPrice, refAddress) {
  let tradeDetailsRequest = await fetch(`${NETWORK_URL}/trade_data?user_address=${useraddress}&src_id=${srcToken}&dst_id=${dstToken}&src_qty=${srcQty}&min_dst_qty=${minDstQty}&gas_price=${gasPrice}&wallet_id=${refAddress}`);
  let tradeDetails = await tradeDetailsRequest.json();
  let rawTx = tradeDetails.data[0];
  await broadcastTx(rawTx);
}

main();
```

## Scenario 2: Obtaining Token and Market Info
### Basic Token Information
The `/currencies` endpoint returns basic information about all tokens supported on Kyber. Details about possible path parameters and output fields can be [found here](api_abi-restfulapi.md#currencies).

#### Code Example
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

const fetch = require('node-fetch')

async function getSupportedTokens() {
  let tokensBasicInfoRequest = await fetch('https://api.kyber.network/currencies?only_official_reserve=false')
  let tokensBasicInfo = await tokensBasicInfoRequest.json()
  console.log(tokensBasicInfo)
}

await getSupportedTokens()
```
#### Output
```json
{  
	"error":false,
	"data":[  
		{  
			"symbol": "OMG",
			"name": "OmiseGO",
			"address": "0x4bfba4a8f28755cb2061c413459ee562c6b9c51b",
			"decimals": 18,
			"id": "0x4bfba4a8f28755cb2061c413459ee562c6b9c51b",
			"reserves_src": [
				"0xEB52Ce516a8d054A574905BDc3D4a176D3a2d51a",
				"0x3EaFE85fBEd7FD20a3b34C4857147aD385685066"
			],
			"reserves_dest": [
				"0xEB52Ce516a8d054A574905BDc3D4a176D3a2d51a",
				"0x3EaFE85fBEd7FD20a3b34C4857147aD385685066"
			]
		},
		{  
			"name":"Ethereum",
			"decimals":18,
			"address":"0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
			"symbol":"ETH",
			"id":"0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
    },
  …
  ]
}
```

### Token Price & Volume Information
The `/market` endpoint returns price and volume information on token to ETH pairs supported on Kyber. Details about possible path parameters and output fields can be [found here](api_abi-restfulapi.md#market).

#### Code Example
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

const fetch = require('node-fetch')

async function getMarketInformation() {
  let marketInfoRequest = await fetch('https://api.kyber.network/market?only_official_reserve=false')
  let marketInfo = await marketInfoRequest.json()
  return marketInfo
}

await getMarketInformation()
```

#### Output
```json
{
  "error": false,
  "data": [
    {
      "timestamp": 1536806619250,
      "pair": "KNC_ETH",
      "quote_symbol": "KNC",
      "base_symbol": "ETH",
      "past_24h_high": 0.001937984496124031,
      "past_24h_low": 0.001857617770187944,
      "usd_24h_volume": 5566.2079180166,
      "eth_24h_volume": 31.8094685833,
      "token_24h_volume": 16865.433010686364,
      "current_bid": 0.001867351485999998,
      "current_ask": 0.0018868074209224932,
      "last_traded": 0.0018868074209224932
    },
    {
      "timestamp": 1536806619251,
      "pair": "OMG_ETH",
      "quote_symbol": "OMG",
      "base_symbol": "ETH",
      "past_24h_high": 0.018518518518518517,
      "past_24h_low": 0.017266283397471997,
      "usd_24h_volume": 13871.8906588085,
      "eth_24h_volume": 78.4248866967,
      "token_24h_volume": 4381.367829085394,
      "current_bid": 0.017379117142599983,
      "current_ask": 0.0175141743763495,
      "last_traded": 0.01777996566748282
    }
  ]
}
```

### Token/ETH and Token/USD Price Information
The `/change24h` endpoint returns current token to ETH and USD rates and price percentage changes against the previous day. Details about possible path parameters and output fields can be [found here](api_abi-restfulapi.md#change24h).

#### Code Example
```js
// DISCLAIMER: Code snippets in this guide are just examples and you
// should always do your own testing. If you have questions, visit our
// https://t.me/KyberDeveloper.

const fetch = require('node-fetch')

async function getPast24HoursTokenInformation() {
  let past24HoursTokenInfoRequest = await fetch('https://api.kyber.network/change24h?only_official_reserve=false')
  let past24HoursTokenInfo = await past24HoursTokenInfoRequest.json()
  return past24HoursTokenInfo
}

await getPast24HoursTokenInformation()
```

#### Output
```json
{
  "ETH_KNC": {
    "timestamp": 1548065183567,
    "token_name": "Kyber Network",
    "token_symbol": "KNC",
    "token_decimal": 18,
    "token_address": "0xdd974d5c2e2928dea5f71b9825b8b646686bd200",
    "rate_eth_now": 0.001144689345999999,
    "change_eth_24h": -2.7115507894449724,
    "change_usd_24h": 13.883783963625097,
    "rate_usd_now": 0.1443257387379925
  },
  "ETH_OMG": {
    "timestamp": 1548065183567,
    "token_name": "OmiseGO",
    "token_symbol": "OMG",
    "token_decimal": 18,
    "token_address": "0xd26114cd6ee289accf82350c8d8487fedb8a0c07",
    "rate_eth_now": 0.0105194607632,
    "change_eth_24h": -2.0145305819746198,
    "change_usd_24h": -10.129,
    "rate_usd_now": 0.8
  },
  ...
}
```

## Inclusion of Permissionless Reserves
By default, the RESTful APIs only interact with reserves that were added in a **permissioned** manner. Most of these endpoints support a `only_official_reserve` parameter for the inclusion of permissionless reserves ([see RESTful API](api_abi-restfulapi.md)). You may find more information about the difference between permissioned and permissionless reserves [in this section](reserves-types.md#permissionless-vs-permissioned).

## Fee Sharing Program
You have the opportunity to join our *Fee Sharing Program*, which allows fee sharing on each swap that originates from your platform. Learn more about the program [here](integrations-feesharing.md)!

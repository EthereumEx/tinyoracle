# TinyOracle

This is a simple example library serving as a guide on how to implement an oracle (aka data provider) with the current Ethereum infrastructure.

What are data providers? Essentially Ethereum contracts run in their own walled garden. They can communicate with each other, can save and retrieve data stored on the blockchain, but external data can only enter the system by interaction from the user (e.g. passing data to a method). Data providers (or oracles) are contracts which have a connection to the outside world. Other contracts can request outside data from them. For example making HTTP GET/POST transactions.

## Use case

For example you want to enable your contract to query for external data, you might do the following (in Solidity):

```js
import "api.sol";

contract SampleClient is usingTinyOracle {
  bytes public response;

  function __tinyOracleCallback(uint256 id, bytes _response) onlyFromTinyOracle external {
    response = _response;
  }

  function query() {
    string memory tmp = "hello world";
    query(bytes(tmp));
  }

  function query(bytes query) {
    queryTinyOracle(query);
  }
}
```

Whenever a query is submitted to ```queryTinyOracle```, it will end up being caught by a server side listener. After processing, the response will be sent back to a specific method (the above ```__tinyOracleCallback```) of the querying contract.

```queryTinyOracle``` will return a uin256 identifier and the same will be available in the ```__tinyOracleCallback```. This is a unique transaction id to enable running multiple calls from a client, because the whole process is asynchronous.

## Solution

It is a fairly simple concept:
- Listen for specific events
- Transact with the caller

To accomplish this we need the following:
- A node accessible with RPC
- A handful of contracts, one of them being our trusted the endpoint for on-chain transactions and emitting an event
- Listening to that specific event over RPC
- Sending back a transaction to a pre-agreed method in the caller contract
- *Job done.*

## Step by step

1. Have ```geth``` fully set up, including an account with ethers. (Shorthand in the following sections for this account is *sender*.)

2. Start the RPC server and unlock the account *sender* used for sending responses:
```
geth --rpc --rpcaddr "127.0.0.1" --rpcport "8545" --unlock 0
```
For the account list use:
```
geth account list
```

3. Deploy ```dispatch.sol```. Take note of its address (shorthand is *dispatch*).

4. Deploy ```lookup.sol```. Take note of its address (shorthand is *lookup*). Call ```setQueryAddress()``` with *dispatch*, and ```setResponseAddress()``` with *sender*.

5. Edit ```api.sol```: replace the address of ```lookupContact``` with the value of *lookup*.

6. Run the RPC listener & dispatch code:
```
tinyoracle --rpcport 8545 --rpchost 127.0.0.1 --interval 10 --dispatch <dispatch> --sender <sender>
```
Interval above is the frequency of polling for incoming requests, where 10 means every 10 seconds. Replace all the parameters with the ones set up earlier.

7. Deploy ```sample-client.sol```. Transact with ```query()``` and call ```getResponse()``` to verify the response was received.

8. Next steps: further improve TinyOracle and send a pull request.

## Important notes

As you are sending responses back as a transaction, which costs ether, it would make sense to charge the clients a fee.

**This code is not intended for use in production.** Any failure to process the event can mean it is lost forever and a response will never be sent. It is suggested to store the received events in a database and process the responses in a separate thread or application.

Offering data through this toolkit to the public will not make it a trusted source. Your user can trust it as long as it trusts you and that the lookup contract is controlled by you.

## Future improvements

1. Received events should be stored in a queue or database and processed asynchronously.

2. There should be a storage for requests in the dispatcher, so that in case of RPC errors, events can still be recalled. This has a cost on storage.

## Under the hood

TinyOracle is running on **Node.js** and uses [json-rpc2](https://github.com/pocesar/node-jsonrpc2) to communicate with the Ethereum RPC endpoint. Data is decoded and encoded using [ethereumjs-abi](https://github.com/axic/ethereumjs-abi).

## Acknowledgements

Thanks goes to [Oraclize.it](http://www.oraclize.it/home/features) who run an actual useful oracle service available today on Ethereum and have inspired this guide.

## License

    Copyright (C) 2015 Alex Beregszaszi

    Permission is hereby granted, free of charge, to any person obtaining a copy of
    this software and associated documentation files (the "Software"), to deal in
    the Software without restriction, including without limitation the rights to
    use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
    the Software, and to permit persons to whom the Software is furnished to do so,
    subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
    FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
    COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
    IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
    CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

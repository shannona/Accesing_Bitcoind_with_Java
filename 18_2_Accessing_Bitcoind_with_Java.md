# 18.2: Accessing Bitcoind with Java

> **NOTE:** This is a draft in progress, so that I can get some feedback from early reviewers. It is not yet ready for learning.

Interacting with the `bitcoind` directly and using command-line `curl` can get simple if you understand how it works, but there's a project [JavaBitcoindRpcClient](https://github.com/Polve/JavaBitcoindRpcClient) that provides the functionality in a Java-API level, making it even easier to interact with your Bitcoin Server.


## Setup Java

To install Java on the VPS Server, you are able to use the `apt-get` command. We will also use [Apache Maven](http://maven.apache.org/) to manage the dependencies, so we will install it together.

```
$ apt-get install openjdk-9-jre-headless maven
```

You can verify your Java installation:
```
$ java -version
openjdk version "9-internal"
OpenJDK Runtime Environment (build 9-internal+0-2016-04-14-195246.buildd.src)
OpenJDK 64-Bit Server VM (build 9-internal+0-2016-04-14-195246.buildd.src, mixed mode)
```

## Setup Dependency

If you use Maven in your Java project, you can include the dependency:
```xml
<dependency>
  <groupId>wf.bitcoin</groupId>
  <artifactId>JavaBitcoindRpcClient</artifactId>
  <version>1.1.0</version>
</dependency>
```

Or if you use Gradle:
```groovy
compile 'wf.bitcoin:JavaBitcoindRpcClient:1.1.0'
```

If you want a sample project and some instructions on how to run it on the server that we just created, you can refer to the [Bitcoind Java Sample Project](https://github.com/brunocvcunha/bitcoind-java-client-sample/).

## Build Your Connection

### Make an RPC Call

To use `JavaBitcoindRpcClient`, you need to create a `BitcoindRpcClient` instance. The arguments in the URL are username, password, IP address and port. You should know this information from your work with `curl` . As you'll recall, the IP address 127.0.0.1 and port 18332 should be correct for the standard testnet setup described in this documents, while you can extract the user and password from `~/.bitcoin/bitcoin.conf`.

```java
  BitcoindRpcClient rpcClient = new BitcoinJSONRPCClient("http://bitcoinrpc:d8340efbcd34e312044c8431c59c792c@127.0.0.1:18332");
```

> **MAINNET VS TESTNET:** The port would be 8332 for a mainnet setup.

If `rpcClient` is successfully initialized, you'll be able to send off RPC commands.

Later, when you're all done with your `bitcoind` connection, you should close it:
```
rpcClient.stop();
```

### Making your first RPC Call

In order to use an RPC method using `JavaBitcoindRpcClient`, you'll find that the `BitcoindRpcClient` provides most of the functionality that can be accessed through `bitcoin-cli` or `curl`, using the same method names. For more details about the commands that you are able to execute and what to expect back, you should refer to [3.2: Knowing Your Bitcoin Setup](03_2_Knowing_Your_Bitcoin_Setup.md).

For example, to execute the `getmininginfo` command to get the block information and the difficulty on the network, you should use the `getMiningInfo()` method:
```java
MiningInfo info = rpcClient.getMiningInfo();
System.out.println("Mining Information");
System.out.println("------------------");
System.out.println("Chain......: " + info.chain());
System.out.println("Blocks.....: " + info.blocks());
System.out.println("Difficulty.: " + info.difficulty());
System.out.println("Hash Power.: " + new BigDecimal(info.networkHashps()).toPlainString());
```
The output for this line should be similar to this:
```
Mining Information
------------------
Chain......: test
Blocks.....: 1254920
Difficulty.: 1.0
Hash Power.: 6585163152453.466796875
```
### Making an RPC Call with Arguments

When you make a RPC call with arguments you need to setup parameters depending method or type of object you use.   

### Look up Address

You can look up addresses on your wallet passing it as an argument.

```java
public AddressInfo getAddressInfo(String address) throws GenericRpcException {
    return new AddressInfoMapWrapper((Map<String, ?>) query("getaddressinfo", address));
  }
  String address = "2MxkZh3Gh3kMBCkLZWM1h6DPAMyCJ2WDNtN";
```

### Look up Funds

You can look up your balance.

```java
public BigDecimal getBalance(String account) throws GenericRpcException {
    return (BigDecimal) query("getbalance");
  }
  String account = "email@email.com";
  String balance = rpcClient.getBalance(account);
  System.out.println("Balance: " + balance);
  
```
Output: 

```
Balance: 0.00000000
```

### Creating an Address

You can create a new address on your wallet attaching a specific label to it, as well as dump the private key for a specific address.
For more information about the wallet setup, you can check [3.3: Setting Up Your Wallet](03_3_Setting_Up_Your_Wallet.md).

```java
  String address = rpcClient.getNewAddress("Learning-Bitcoin-from-the-Command-Line");
  System.out.println("New Address: " + address);

  String privKey = rpcClient.dumpPrivKey(address);
  System.out.println("Priv Key: " + privKey);
```
Output:
```
New Address: mpsFtZ8qTJPRGZy1gaaUw37fHeUSPLkzzs
Priv Key: cTy2AnmAALsHokYzJzTdsUBSqBtypmWfmSNYgG6qQH43euUZgqic
```

### Create a Transaction

You can create a raw transaction using createRawTransaction method passing as arguments and ArrayList containing inputs and outputs to be used.

```java
public String create() throws GenericRpcException {
 return bitcoin.createRawTransaction(new ArrayList<>(inputs), outputs);
}
```

### Sending Transactions

You can easily send a transaction using the method `sendToAddress()`.
For more information about sending transactions, you can check [4: Sending Bitcoin Transactions](04_0_Sending_Bitcoin_Transactions.md).


```java
String sendToAddress = rpcClient.sendToAddress("mgnNsZj6tPzpd7JwTTidUKnGoDTkcucLT5", 1);
System.out.println("Send: " + sendToAddress);
```
This program will output a transaction id, for example:
```
a2d2f629d6666ca6e440169a322850cd9d133f637f7a02a02a0a7477bc5687d4
```

In case you want to adjust the transaction fee, you can use the `setTxFee` method before sending the output:

```java
rpcClient.setTxFee(new BigDecimal(0.001).setScale(3, BigDecimal.ROUND_DOWN));
```

### Listening to Transactions or Blocks

You may want to write applications that keep listening the Blockchain, and execute a specific code when something happens, such as a transaction that involves an address in your wallet, or even the generation of a new block in the network.
To do that, `JavaBitcoindRpcClient` provides support to `BitcoinAcceptor`, where you can attach listeners in the network.

Example:
```java
  BitcoinAcceptor acceptor = new BitcoinAcceptor(rpcClient, blockHash, 6, new BitcoinPaymentListener() {

      @Override
      public void transaction(Transaction tx) {
          System.out.println("Transaction: " + tx);

      }

      @Override
      public void block(String block) {
          System.out.println("Block: " + block);

      }
  });
  acceptor.run();
```

Every time some transaction is sent, or a new block is generated, you should see a similar output in your console:
```
Transaction: {account=Tests, address=mhopuJzgmTwhGfpNLCJ9CRknugY691oXp1, category=receive, amount=5.0E-4, label=Tests, vout=1, confirmations=0, trusted=false, txid=361e8fcff243b74ebf396e595a007636654f67c3c7b55fd2860a3d37772155eb, walletconflicts=[], time=1513132887, timereceived=1513132887, bip125-replaceable=unknown}

Block: 000000004564adfee3738314549f7ca35d96c4da0afc6b232183917086b6d971
```

### For More Information

In this repository you can browse all souce code for bitcoin-rpc-client (https://github.com/Polve/bitcoin-rpc-client)

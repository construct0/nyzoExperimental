# Chapter 1: Verifier

Welcome to the `nyzoVerifier` tutorial! We're starting our journey by exploring the most fundamental concept in the Nyzo network: the **Verifier**.

Imagine a group of highly diligent accountants working together on a shared, public ledger. Each accountant checks every entry, votes on the correct version of the ledger, and keeps their own copy up-to-date. In the Nyzo world, these accountants are called **Verifiers**.

## What is a Verifier?

A **Verifier** is the core software program that actively participates in maintaining the Nyzo blockchain. Think of it as the "brain" and "worker" of the Nyzo network running on a computer.

Here's what a Verifier does:

1.  **Validates Transactions:** Just like an accountant checks if a transaction slip is valid (correct signatures, sufficient funds), a Verifier checks incoming [Transaction](04_transaction_.md)s to make sure they follow the rules of the Nyzo network.
2.  **Creates New Blocks:** After validating transactions, Verifiers bundle them together into a new "page" for the ledger. This page is called a [Block](05_block_.md).
3.  **Votes on Blocks:** Since many Verifiers might create a new block around the same time, they need to agree on which *one* block is the next official page in the ledger. They do this through a voting process, which we'll cover in the [Consensus (Voting)](07_consensus__voting__.md) chapter.
4.  **Maintains the Ledger:** Each Verifier keeps a full copy of the entire blockchain (the shared ledger) and constantly updates it with the newly agreed-upon blocks.

## Identity: The Keys to the Office

How does the network know *which* Verifier is which? Each Verifier has a unique digital identity, similar to how an accountant has a unique license number and a personal signature. This identity consists of two parts:

1.  **Public Key (Identifier):** This is like the Verifier's public license number or office address. Everyone in the network can see it, and it's used to identify the Verifier.
2.  **Private Key (Seed):** This is like the Verifier's secret signature stamp or the key to their private office. It's kept completely secret and is used by the Verifier to "sign" the blocks they create and the messages they send, proving they actually came from them.

In Nyzo, the private key is often called the "seed". From this secret seed, the public key (identifier) can be mathematically generated. But importantly, you *cannot* figure out the secret seed just by looking at the public identifier!

Let's see how the `nyzoVerifier` software handles this identity. The main program logic is in `Verifier.java`. When a verifier starts, it needs its private seed.

```java
// Simplified snippet from src/main/java/co/nyzo/verifier/Verifier.java

public class Verifier {

    // This holds the secret private key (seed). It's loaded from a file.
    private static byte[] privateSeed = null;

    // ... other variables ...

    // This method loads the seed from a file or creates a new one if it doesn't exist.
    private static synchronized void loadPrivateSeed() {
        // Simplified: Tries to read "verifier_private_seed" file
        // If the file exists and contains a valid seed, load it into privateSeed.
        // If not, generate a new random seed and save it to the file.
        // ... file reading/writing logic ...
        if (privateSeed == null /* or invalid */) {
            // privateSeed = KeyUtil.generateSeed(); // Generate a new secret key
            // ... save the new seed to the file ...
        }
        // ... also saves user-friendly versions (NyzoStrings) to "verifier_info" file ...
    }

    // This method uses the loaded privateSeed to generate the public identifier.
    public static byte[] getIdentifier() {
        // KeyUtil computes the public identifier from the private seed.
        return KeyUtil.identifierForSeed(privateSeed);
    }

    // This method uses the privateSeed to sign data (like blocks or messages).
    public static byte[] sign(byte[] bytesToSign) {
        // SignatureUtil uses the private seed to create a digital signature.
        return SignatureUtil.signBytes(bytesToSign, privateSeed);
    }

    // ... main method and other functions ...
}
```

*   **`loadPrivateSeed()`:** This function is crucial. It looks for a file named `verifier_private_seed` in the Nyzo data directory. If it finds a valid seed there, it uses it. If not, it creates a brand new random seed and saves it. This seed *must* be kept secret!
*   **`getIdentifier()`:** This function takes the secret `privateSeed` and calculates the corresponding public identifier. This identifier is what other Verifiers in the network will see.
*   **`sign()`:** When the Verifier needs to prove its identity (like when creating a block or casting a vote), it uses this function along with its `privateSeed` to create a unique digital signature. Others can use the Verifier's public identifier to check that the signature is valid *without* needing to know the secret seed.

These public and private keys are often represented in a user-friendly format called [NyzoString](10_nyzostring_.md), which helps prevent errors when copying or sharing them.

## Running the Verifier Software

The `Verifier.java` file isn't just about identity; it's the main entry point for running the entire verifier program. When you start the `nyzoVerifier`, the `main` method in `Verifier.java` kicks things off.

```java
// Simplified snippet from src/main/java/co/nyzo/verifier/Verifier.java

public class Verifier {

    // ... variables and identity methods ...

    public static void main(String[] args) {
        // This is where the program starts.
        // ... argument handling ...
        RunMode.setRunMode(RunMode.Verifier); // Tell the software it's running as a Verifier
        BlockManager.initialize(); // Set up the component that manages blocks
        start(); // Call the main startup sequence
    }

    public static void start() {
        // 1. Load the secret key (privateSeed)
        loadPrivateSeed();
        // ... log the verifier's public identifier ...

        // 2. Start listening for messages from other verifiers
        MeshListener.start(); // Related to Node communication

        // 3. Load the very first block (Genesis Block)
        loadGenesisBlock();

        // 4. Load the verifier's nickname (just for display)
        loadNickname();

        // ... start other background tasks ...

        // 5. Connect to the network by contacting trusted entry points
        // ... get list of trustedEntryPoints ...
        // ... send mesh requests and node join messages to them (using NodeManager)...

        // 6. Make sure we have the latest blockchain state
        ChainInitializationManager.initializeFrozenEdge(trustedEntryPoints);

        // ... wait until connected to enough peers in the network ...

        // 7. Start the main loop!
        // ... create and start a new Thread that runs verifierMain() ...
        System.out.println("ready to start thread for main verifier loop");
    }

    // ... other methods like verifierMain() ...
}
```

The `start()` method orchestrates the setup: loading identity, starting network communication ([Node](02_node_.md) related), getting the initial [Block](05_block_.md), and connecting to other Verifiers via the [NodeManager](02_node_.md). Finally, it launches the `verifierMain()` loop in a separate thread.

## The Verifier's Daily Grind: The Main Loop

Once `start()` is done, the `verifierMain()` method runs continuously, performing the core duties of a Verifier.

```java
// Simplified snippet from src/main/java/co/nyzo/verifier/Verifier.java

public class Verifier {

    // ... other methods ...

    private static void verifierMain() {
        while (/* program should keep running */ !UpdateUtil.shouldTerminate()) {
            // Wait until previous tasks are done
            MessageQueue.blockThisThreadUntilClear();

            try {
                // Only do work if connected to the network
                if (NodeManager.connectedToMesh()) {

                    // === Core Verifier Tasks ===

                    // 1. Try to create the next block if it's time
                    // extendBlock(...) calls createNextBlock(...)
                    // createNextBlock gets Transactions, checks balances, builds the block

                    // 2. Send the new block to others if ready
                    // Message.broadcast(new Message(MessageType.NewBlock9, ...));

                    // 3. Vote on blocks proposed by others (managed by UnfrozenBlockManager)
                    // UnfrozenBlockManager.updateVote();
                    // UnfrozenBlockManager.attemptToFreezeBlock(); // Try to finalize the next block

                    // 4. Ask for missing blocks or votes if needed
                    // BlockVoteManager.requestMissingFrozenBlocks();
                    // UnfrozenBlockManager.requestMissingBlocks();

                    // 5. Perform regular maintenance
                    // NodeManager.reloadNodeJoinQueue(); // Update list of known verifiers
                    // TransactionPool.updateFrozenEdge(); // Clean old transactions
                    // ... other cleanup tasks ...
                }

            } catch (Exception reportOnly) {
                // Log errors but keep running
            }

            // Pause briefly to avoid using 100% CPU
            ThreadUtil.sleep(300L); // Sleep for 300 milliseconds
        }
    }
    // ... methods for creating blocks like createNextBlock() ...
}
```

Inside this loop, the Verifier constantly:
*   Checks if it's time to create a new [Block](05_block_.md) (`extendBlock`, `createNextBlock`).
*   Shares any new block it created.
*   Processes blocks and votes received from others to reach [Consensus (Voting)](07_consensus__voting__.md).
*   Requests any missing information from the network.
*   Performs housekeeping tasks like managing its list of known peers ([NodeManager](02_node_.md)) and cleaning up old [Transaction](04_transaction_.md) data.

## Verifier vs. Node: Software vs. Address

It's easy to confuse a Verifier with a [Node](02_node_.md), but they are distinct concepts:

*   **Verifier:** The *software entity* identified by its public/private keys, responsible for validating, voting, and block creation. It's the "accountant".
*   **Node:** The *network endpoint* – essentially the IP address and port number – through which a Verifier communicates with other Verifiers. It's the "accountant's office address and phone number".

The `Verifier.java` class represents the Verifier entity and orchestrates its actions. The `Node.java` class represents the network information *about* a Verifier (its own or others). The [NodeManager](02_node_.md) keeps track of all the Nodes the Verifier knows about.

```java
// Snippet from: src/main/java/co/nyzo/verifier/Node.java

public class Node implements MessageObject {

    private byte[] identifier; // The Verifier's public key (who lives here?)
    private byte[] ipAddress;  // The network IP address (where is the office?)
    private int portTcp;       // The TCP port number (which phone line?)
    // ... other details like UDP port, status ...

    // Constructor to create a Node object
    public Node(byte[] identifier, byte[] ipAddress, int portTcp, int portUdp) {
        this.identifier = identifier; // Store the Verifier's ID
        this.ipAddress = ipAddress;   // Store the IP address
        this.portTcp = portTcp;       // Store the TCP port
        // ... initialize other fields ...
    }

    // Method to get the Verifier's identifier associated with this Node
    public byte[] getIdentifier() {
        return identifier;
    }
    // ... other methods to get IP, port, etc. ...
}
```

As you can see, a `Node` object contains the `identifier` of the Verifier operating at that specific network address (`ipAddress` and `portTcp`).

## Conclusion

You've now learned about the **Verifier**, the fundamental participant in the Nyzo network. It's like a digital accountant responsible for validating transactions, creating blocks, voting on the correct blockchain state, and maintaining its own copy of the ledger. Each Verifier has a unique identity based on a public/private key pair (identifier/seed) and runs as a software program, coordinated mainly by the `Verifier.java` class.

Verifiers need a way to talk to each other over the network. This brings us to our next topic: the network endpoint itself.

Let's move on to [Chapter 2: Node](02_node_.md) to understand how Verifiers find and communicate with each other.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)
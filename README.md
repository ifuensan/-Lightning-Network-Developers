
# Stage 1 - Setting up a local cluster
## Introduction

In this stage of the tutorial, we will learn how to set up a local cluster of nodes Alice, Bob, and Charlie, have them talk to each other, set up channels, and route payments between one another. We will also establish a baseline understanding of the different components that must work together as part of developing on lnd.

This tutorial assumes you have completed installation of Go, btcd, and lnd on simnet. If not, please refer to the installation instructions. Note that for the purposes of this tutorial it is not necessary to sync testnet, and the last section of the installation instructions you’ll need to complete is “Installing btcd.”

The schema will be the following. Keep in mind that you can easily extend this network to include additional nodes David, Eve, etc. by simply running more local lnd instances.

   (1)                        (1)                         (1)
+ ----- +                   + --- +                   + ------- +
| Alice | <--- channel ---> | Bob | <--- channel ---> | Charlie |
+ ----- +                   + --- +                   + ------- +
    |                          |                           |
    |                          |                           |
    + - - - -  - - - - - - - - + - - - - - - - - - - - - - +
                               |
                      + --------------- +
                      | BTC/LTC network | <--- (2)
                      + --------------- +

## Understanding the components
### LND

lnd is the main component that we will interact with. lnd stands for Lightning Network Daemon, and handles channel opening/closing, routing and sending payments, and managing all the Lightning Network state that is separate from the underlying Bitcoin network itself.

Running an lnd node means that it is listening for payments, watching the blockchain, etc. By default it is awaiting user input.

lncli is the command line client used to interact with your lnd nodes. Typically, each lnd node will be running in its own terminal window, so that you can see its log outputs. lncli commands are thus run from a different terminal window.

### BTCD

btcd represents the gateway that lnd nodes will use to interact with the Bitcoin / Litecoin network. lnd needs btcd for creating on-chain addresses or transactions, watching the blockchain for updates, and opening/closing channels. In our current schema, all three of the nodes are connected to the same btcd instance. In a more realistic scenario, each of the lnd nodes will be connected to their own instances of btcd or equivalent.

We will also be using simnet instead of testnet. Simnet is a development/test network run locally that allows us to generate blocks at will, so we can avoid the time-consuming process of waiting for blocks to arrive for any on-chain functionality.

## Setting up our environment

Developing on lnd can be quite complex since there are many more moving pieces, so to simplify that process, we will walk through a recommended workflow.

Running btcd

Let’s start by running btcd, if you don’t have it up already. Open up a new terminal window, ensure you have your $GOPATH set, and run:

btcd --txindex --simnet --rpcuser=kek --rpcpass=kek

(Note: this tutorial requires opening quite a few terminal windows. It may be convenient to use multiplexers such as tmux or screen if you’re familiar with them.)

Breaking down the components:

    --txindex is required so that the lnd client is able to query historical transactions from btcd.
    --simnet specifies that we are using the simnet network. This can be changed to --testnet, or omitted entirely to connect to the actual Bitcoin / Litecoin network.
    --rpcuser and rpcpass sets a default password for authenticating to the btcd instance.

Starting lnd (Alice’s node)

Now, let’s set up the three lnd nodes. To keep things as clean and separate as possible, open up a new terminal window, ensure you have $GOPATH set and $GOPATH/bin in your PATH, and create a new directory under $GOPATH called dev that will represent our development space. We will create separate folders to store the state for alice, bob, and charlie, and run all of our lnd nodes on different localhost ports instead of using Docker to make our networking a bit easier.
```
# Create our development space
cd $GOPATH
mkdir dev
cd dev

# Create folders for each of our nodes
mkdir alice bob charlie
```
The directory structure should now look like this:
```
$ tree $GOPATH -L 2

├── bin
│   └── ...
├── dev
│   ├── alice
│   ├── bob
│   └── charlie
├── pkg
│   └── ...
├── rpc
│   └── ...
└── src
    └── ...
```

Start up the Alice node from within the alice directory:
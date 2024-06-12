[Español](#etapa-1-configurando-un-cluster-local)
# Stage 1 - Setting up a local cluster
## Introduction

In this stage of the tutorial, we will learn how to set up a local cluster of nodes Alice, Bob, and Charlie, have them talk to each other, set up channels, and route payments between one another. We will also establish a baseline understanding of the different components that must work together as part of developing on lnd.

This tutorial assumes you have completed installation of Go, btcd, and lnd on simnet. If not, please refer to the installation instructions. Note that for the purposes of this tutorial it is not necessary to sync testnet, and the last section of the installation instructions you’ll need to complete is “Installing btcd.”

The schema will be the following. Keep in mind that you can easily extend this network to include additional nodes David, Eve, etc. by simply running more local lnd instances.

```
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

```

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

### Running btcd

Let’s start by running btcd, if you don’t have it up already. Open up a new terminal window, ensure you have your $GOPATH set, and run:

btcd --txindex --simnet --rpcuser=kek --rpcpass=kek

(Note: this tutorial requires opening quite a few terminal windows. It may be convenient to use multiplexers such as tmux or screen if you’re familiar with them.)

Breaking down the components:

    --txindex is required so that the lnd client is able to query historical transactions from btcd.
    --simnet specifies that we are using the simnet network. This can be changed to --testnet, or omitted entirely to connect to the actual Bitcoin / Litecoin network.
    --rpcuser and rpcpass sets a default password for authenticating to the btcd instance.

### Starting lnd (Alice’s node)

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
```
cd $GOPATH/dev/alice
alice$ lnd --rpclisten=localhost:10001 --listen=localhost:10011 --restlisten=localhost:8001 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek 
```

The Alice node should now be running and displaying output ending with a line beginning with “Waiting for wallet encryption password.”

Breaking down the components:

`--rpclisten`: The host:port to listen for the RPC server. This is the primary way an application will communicate with lnd
`--listen`: The host:port to listen on for incoming P2P connections. This is at the networking level, and is distinct from the Lightning channel networks and Bitcoin/Litcoin network itself.
`--restlisten`: The host:port exposing a REST api for interacting with lnd over HTTP. For example, you can get Alice’s channel balance by making a GET request to localhost:8001/v1/channels. This is not needed for this tutorial, but you can see some examples here.
`--datadir`: The directory that lnd’s data will be stored inside
`--logdir`: The directory to log output.
`--debuglevel`: The logging level for all subsystems. Can be set to trace, debug, info, warn, error, critical.
`--bitcoin.simnet`: Specifies whether to use simnet or testnet
`--bitcoin.active`: Specifies that bitcoin is active. Can also include `--litecoin.active` to activate Litecoin.
`--bitcoin.node=btcd`: Use the btcd full node to interface with the blockchain. Note that when using Litecoin, the option is `--litecoin.node=btcd`.
`--btcd.rpcuser` and `--btcd.rpcpass`: The username and password for the btcd instance. Note that when using Litecoin, the options are `--ltcd.rpcuser` and `--ltcd.rpcpass`.

## Starting Bob’s node and Charlie’s node
Just as we did with Alice, start up the Bob node from within the bob directory, and the Charlie node from within the charlie directory. Doing so will configure the datadir and logdir to be in separate locations so that there is never a conflict.

Keep in mind that for each additional terminal window you set, you will need to set `$GOPATH` and include `$GOPATH/bin` in your PATH. Consider creating a setup script that includes the following lines:

```
export GOPATH=~/gocode # if you exactly followed the install guide
export PATH=$PATH:$GOPATH/bin
```
and run it every time you start a new terminal window working on lnd.

Run Bob and Charlie:
```
# In a new terminal window
cd $GOPATH/dev/bob
bob$ lnd --rpclisten=localhost:10002 --listen=localhost:10012 --restlisten=localhost:8002 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek 

# In another terminal window
cd $GOPATH/dev/charlie
charlie$ lnd --rpclisten=localhost:10003 --listen=localhost:10013 --restlisten=localhost:8003 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek
```
## Configuring lnd.conf
To skip having to type out a bunch of flags on the command line every time, we can instead modify our `lnd.conf`, and the arguments specified therein will be loaded into lnd automatically. Any additional configuration added as a command line argument will be applied after reading from lnd.conf, and will overwrite the lnd.conf option if applicable.

On MacOS, `lnd.conf` is located at: `/Users/[username]/Library/Application\ Support/Lnd/lnd.conf`
On Linux: `~/.lnd/lnd.conf`
Here is an example `lnd.conf` that can save us from re-specifying a bunch of command line options:

```
[Application Options]
datadir=data
logdir=log
debuglevel=info

[Bitcoin]
bitcoin.simnet=1
bitcoin.active=1
bitcoin.node=btcd

[btcd]
btcd.rpcuser=kek
btcd.rpcpass=kek
```

Now, when we start nodes, we only have to type
```
alice$ lnd --rpclisten=localhost:10001 --listen=localhost:10011 --restlisten=localhost:8001
bob$ lnd --rpclisten=localhost:10002 --listen=localhost:10012 --restlisten=localhost:8002
charlie$ lnd --rpclisten=localhost:10003 --listen=localhost:10013 --restlisten=localhost:8003
```
etc.

### Working with lncli and authentication
Now that we have our lnd nodes up and running, let’s interact with them! To control lnd we will need to use lncli, the command line interface.

lnd uses macaroons for authentication to the rpc server. lncli typically looks for an admin.macaroon file in the Lnd home directory, but since we changed the location of our application data, we have to set `--macaroonpath` in the following command. To disable macaroons, pass the `--no-macaroons` flag into both lncli and lnd.

lnd allows you to encrypt your wallet with a passphrase and optionally encrypt your cipher seed passphrase as well. This can be turned off by passing `--noencryptwallet` into `lnd` or `lnd.conf`. We recommend going through this process at least once to familiarize yourself with the security and authentication features around lnd.

We will test our rpc connection to the Alice node. Notice that in the following command we specify the `--rpcserver` here, which corresponds to `--rpcport=10001` that we set when starting the Alice lnd node.

Open up a new terminal window, set `$GOPATH` and include `$GOPATH/bin` in your `PATH` as usual. Let’s create Alice’s wallet and set her passphrase:

```
cd $GOPATH/dev/alice
alice$ lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
```

You’ll be asked to input and confirm a wallet password for Alice, which must be longer than 8 characters. You also have the option to add a passphrase to your cipher seed. For now, just skip this step by entering “n” when prompted about whether you have an existing mnemonic, and pressing enter to proceed without the passphrase.

You can now request some basic information as follows:

```
alice$ lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon getinfo
```
lncli just made an RPC call to the Alice lnd node. This is a good way to test if your nodes are up and running and lncli is functioning properly. Note that in future sessions you may need to call lncli unlock to unlock the node with the password you just set.

Open up new terminal windows and do the same for Bob and Charlie. alice$ or bob$ denotes running the command from the Alice or Bob lncli window respectively.

```
# In a new terminal window, setting $GOPATH, etc.
cd $GOPATH/dev/bob
bob$ lncli --rpcserver=localhost:10002 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
# Note that you'll have to enter an 8+ character password and "n" for the mnemonic.

# In a new terminal window:
cd $GOPATH/dev/charlie
charlie$ lncli --rpcserver=localhost:10003 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
# Note that you'll have to enter an 8+ character password and "n" for the mnemonic.
```
To avoid typing the `--rpcserver=localhost:1000X` and `--macaroonpath` flag every time, we can set some aliases. Add the following to your `.bashrc`:

```
alias lncli-alice="lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
alias lncli-bob="lncli --rpcserver=localhost:10002 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
alias lncli-charlie="lncli --rpcserver=localhost:10003 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
```

To make sure this was applied to all of your current terminal windows, rerun your `.bashrc` file:

```
alice$ source ~/.bashrc
bob$ source ~/.bashrc
charlie$ source ~/.bashrc
```
For simplicity, the rest of the tutorial will assume that this step was complete.

### lncli options
To see all the commands available for lncli, simply type lncli --help or lncli -h.

## Setting up Bitcoin addresses
Let’s create a new Bitcoin address for Alice. This will be the address that stores Alice’s on-chain balance. np2wkh specifes the type of address and stands for Pay to Nested Witness Key Hash.

```
alice$ lncli-alice newaddress np2wkh
{
    "address": <ALICE_ADDRESS>
}
```
And for Bob and Charlie:
```
bob$ lncli-bob newaddress np2wkh
{
    "address": <BOB_ADDRESS>
}
charlie$ lncli-charlie newaddress np2wkh
{
    "address": <CHARLIE_ADDRESS>
}
```
## Funding Alice
That’s a lot of configuration! At this point, we’ve generated onchain addresses for Alice, Bob, and Charlie. Now, we will get some practice working with btcd and fund these addresses with some simnet Bitcoin.

Quit btcd and re-run it, setting Alice as the recipient of all mining rewards:

`btcd --simnet --txindex --rpcuser=kek --rpcpass=kek --miningaddr=<ALICE_ADDRESS>`
Generate 400 blocks, so that Alice gets the reward. We need at least 100 blocks because coinbase funds can’t be spent until after 100 confirmations, and we need about 300 to activate segwit. In a new window with $GOPATH and $PATH set:

`alice$ btcctl --simnet --rpcuser=kek --rpcpass=kek generate 400`
Check that segwit is active:

`btcctl --simnet --rpcuser=kek --rpcpass=kek getblockchaininfo | grep -A 1 segwit`
Check Alice’s wallet balance.

`alice$ lncli-alice walletbalance`
It’s no fun if only Alice has money. Let’s give some to Charlie as well.

```
# Quit the btcd process that was previously started with Alice's mining address,
# and then restart it with:
btcd --txindex --simnet --rpcuser=kek --rpcpass=kek --miningaddr=<CHARLIE_ADDRESS>

# Generate more blocks
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 100

# Check Charlie's balance
charlie$ lncli-charlie walletbalance
```
### Creating the P2P Network
Now that Alice and Charlie have some simnet Bitcoin, let’s start connecting them together.

Connect Alice to Bob:

```
# Get Bob's identity pubkey:
bob$ lncli-bob getinfo
{
--->"identity_pubkey": <BOB_PUBKEY>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 450,
    "block_hash": "2a84b7a2c3be81536ef92cf382e37ab77f7cfbcf229f7d553bb2abff3e86231c",
    "synced_to_chain": true,
    "testnet": false,
    "chains": [
        "bitcoin"
    ],
    "uris": [
    ],
    "best_header_timestamp": "1533350134",
    "version":  "0.4.2-beta commit=7a5a824d179c6ef16bd78bcb7a4763fda5f3f498"
}

# Connect Alice and Bob together
alice$ lncli-alice connect <BOB_PUBKEY>@localhost:10012
{

}
```
Notice that `localhost:10012` corresponds to the `--listen=localhost:10012` flag we set when starting the Bob lnd node.

Let’s check that Alice and Bob are now aware of each other.
```
# Check that Alice has added Bob as a peer:
alice$ lncli-alice listpeers
{
    "peers": [
        {
            "pub_key": <BOB_PUBKEY>,
            "address": "127.0.0.1:10012",
            "bytes_sent": "7",
            "bytes_recv": "7",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "0"
        }
    ]
}

# Check that Bob has added Alice as a peer:
bob$ lncli-bob listpeers
{
    "peers": [
        {
            "pub_key": <ALICE_PUBKEY>,
            "address": "127.0.0.1:60104",
            "bytes_sent": "318",
            "bytes_recv": "318",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "5788"
        }
    ]
}
```
Finish up the P2P network by connecting Bob to Charlie:

`charlie$ lncli-charlie connect <BOB_PUBKEY>@localhost:10012`

### Setting up Lightning Network
Before we can send payment, we will need to set up payment channels from Alice to Bob, and Bob to Charlie.

First, let’s open the Alice<–>Bob channel.

`alice$ lncli-alice openchannel --node_key=<BOB_PUBKEY> --local_amt=1000000
--local_amt` specifies the amount of money that Alice will commit to the channel. To see the full list of options, you can try lncli openchannel `--help`.
We now need to mine six blocks so that the channel is considered valid:

`btcctl --simnet --rpcuser=kek --rpcpass=kek generate 6`
Check that Alice<–>Bob channel was created:

```
alice$ lncli-alice listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": <BOB_PUBKEY>,
            "channel_point": "2622b779a8acca471a738b0796cd62e4457b79b33265cbfa687aadccc329023a:0",
            "chan_id": "495879744192512",
            "capacity": "1000000",
            "local_balance": "991312",
            "remote_balance": "0",
            "commit_fee": "8688",
            "commit_weight": "600",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",
            "pending_htlcs": [
            ],
            "csv_delay": 144,
            "private": false
        }
    ]
}
```

Sending single hop payments
Finally, to the exciting part - sending payments! Let’s send a payment from Alice to Bob.

First, Bob will need to generate an invoice:

```
bob$ lncli-bob addinvoice --amt=10000
{
        "r_hash": "<a_random_rhash_value>",
        "pay_req": "<encoded_invoice>",
        "add_index": 1
}
```
Send the payment from Alice to Bob:

```
alice$ lncli-alice sendpayment --pay_req=<encoded_invoice>
{
	"payment_error": "",
	"payment_preimage": "baf6929fc95b3824fb774a4b75f6c8a1ad3aaef04efbf26cc064904729a21e28",
	"payment_route": {
		"total_time_lock": 650,
		"total_amt": 10000,
		"hops": [
			{
				"chan_id": 495879744192512,
				"chan_capacity": 1000000,
				"amt_to_forward": 10000,
                "expiry": 650,
                "amt_to_forward_msat": 10000000
			}
		],
		"total_amt_msat": 10000000
	}
}

# Check that Alice's channel balance was decremented accordingly:
alice$ lncli-alice listchannels

# Check that Bob's channel was credited with the payment amount:
bob$ lncli-bob listchannels
```
## Multi-hop payments
Now that we know how to send single-hop payments, sending multi hop payments is not that much more difficult. Let’s set up a channel from Bob<–>Charlie:

```
charlie$ lncli-charlie openchannel --node_key=<BOB_PUBKEY> --local_amt=800000 --push_amt=200000

# Mine the channel funding tx
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 6
```

Note that this time, we supplied the `--push_amt` argument, which specifies the amount of money we want the other party to have at the first channel state.

Let’s make a payment from Alice to Charlie by routing through Bob:
```
charlie$ lncli-charlie addinvoice --amt=10000
alice$ lncli-alice sendpayment --pay_req=<encoded_invoice>

# Check that Charlie's channel was credited with the payment amount (e.g. that
# the `remote_balance` has been decremented by 10000)
charlie$ lncli-charlie listchannels
```

### Closing channels
For practice, let’s try closing a channel. We will reopen it in the next stage of the tutorial.

```
alice$ lncli-alice listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
       ---->"channel_point": "3511ae8a52c97d957eaf65f828504e68d0991f0276adff94c6ba91c7f6cd4275:0",
            "chan_id": "1337006139441152",
            "capacity": "1005000",
            "local_balance": "990000",
            "remote_balance": "10000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2"
        }
    ]
}
```
The Channel point consists of two numbers separated by a colon, which uniquely identifies the channel. The first number is funding_txid and the second number is output_index.

```
# Close the Alice<-->Bob channel from Alice's side.
alice$ lncli-alice closechannel --funding_txid=<funding_txid> --output_index=<output_index>

# Mine a block including the channel close transaction to close the channel:
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 1

# Check that Bob's on-chain balance was credited by his settled amount in the
# channel. Recall that Bob previously had no on-chain Bitcoin:
bob$ lncli-bob walletbalance
{
    "total_balance": "20001",
    "confirmed_balance": "20001",
    "unconfirmed_balance": "0"
}
```
At this point, you’ve learned how to work with btcd, btcctl, lnd, and lncli. In Stage 2, we will learn how to set up and interact with lnd using a web GUI client.

In the future, you can try running through this workflow on testnet instead of simnet, where you can interact with and send payments through the testnet Lightning Faucet node. For more information, see the “Connect to faucet lightning node” section in the [Docker guide](https://dev.lightning.community/guides/docker/) or check out the [Lightning Network faucet repository](https://github.com/lightninglabs/lightning-faucet).

### Navigation
[Proceed to Stage 2 - Web Client](https://dev.lightning.community/tutorial/02-web-client)
[Return to main tutorial page](https://dev.lightning.community/tutorial/)

### Questions
Join the #dev-help channel on our [Community Slack](https://join.slack.com/t/lightningcommunity/shared_invite/enQtMzQ0OTQyNjE5NjU1LWRiMGNmOTZiNzU0MTVmYzc1ZGFkZTUyNzUwOGJjMjYwNWRkNWQzZWE3MTkwZjdjZGE5ZGNiNGVkMzI2MDU4ZTE)
Join IRC: [Irc](https://webchat.freenode.net/?channels=lnd)

# Etapa 1: Configurando un cluster local
## Introducción

En esta etapa del tutorial, aprenderemos cómo configurar un grupo local de nodos Alice, Bob y Charlie, hacer que se comuniquen entre sí, configurar canales y enrutar pagos entre sí. También estableceremos una comprensión básica de los diferentes componentes que deben trabajar juntos como parte del desarrollo de lnd.

Este tutorial asume que ha completado la instalación de Go, btcd y lnd en simnet. De lo contrario, consulte las instrucciones de instalación. Tenga en cuenta que para los fines de este tutorial no es necesario sincronizar testnet y la última sección de las instrucciones de instalación que deberá completar es 'Instalación de btcd'.

El esquema será el siguiente. Tenga en cuenta que puede ampliar fácilmente esta red para incluir nodos adicionales David, Eve, etc. simplemente ejecutando más instancias lnd locales.

```
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

```

## Comprender los componentes
### LND

lnd es el componente principal con el que interactuaremos. lnd significa Lightning Network Daemon y maneja la apertura/cierre de canales, enrutamiento y envío de pagos, y administra todo el estado de Lightning Network que está separado de la propia red Bitcoin subyacente.

Ejecutar un nodo lnd significa que está escuchando pagos, observando la cadena de bloques, etc. De forma predeterminada, está esperando la entrada del usuario.

lncli es el cliente de línea de comando que se utiliza para interactuar con sus nodos lnd. Normalmente, cada nodo lnd se ejecutará en su propia ventana de terminal, de modo que pueda ver las salidas de sus registros. Por lo tanto, los comandos lncli se ejecutan desde una ventana de terminal diferente.
### BTCD

btcd represents the gateway that lnd nodes will use to interact with the Bitcoin / Litecoin network. lnd needs btcd for creating on-chain addresses or transactions, watching the blockchain for updates, and opening/closing channels. In our current schema, all three of the nodes are connected to the same btcd instance. In a more realistic scenario, each of the lnd nodes will be connected to their own instances of btcd or equivalent.

We will also be using simnet instead of testnet. Simnet is a development/test network run locally that allows us to generate blocks at will, so we can avoid the time-consuming process of waiting for blocks to arrive for any on-chain functionality.

## Configurando nuestro entorno

Desarrollar en lnd puede ser bastante complejo ya que hay muchas más piezas móviles, por lo que para simplificar ese proceso, vamos a repasar un flujo de trabajo recomendado.

### Ejecutando btcd

Comencemos ejecutando btcd, si aún no lo tienes en funcionamiento. Abre una nueva ventana de terminal, asegúrate de tener configurado tu $GOPATH, y ejecuta:

```
btcd --txindex --simnet --rpcuser=kek --rpcpass=kek
```

(Nota: este tutorial requiere abrir bastantes ventanas de terminal. Puede ser conveniente usar multiplexores como tmux o screen si estás familiarizado con ellos.)

Desglosando los componentes:

    `--txindex` es necesario para que el cliente lnd pueda consultar transacciones históricas desde btcd.
    `--simnet` especifica que estamos usando la red simnet. Esto puede cambiarse a `--testnet`, o omitirse por completo para conectarse a la red real de Bitcoin / Litecoin.
    `--rpcuser` y `--rpcpass` establecen una contraseña predeterminada para autenticarse en la instancia de btcd.


### Iniciando lnd (nodo de Alice)

Ahora, configuraremos los tres nodos lnd. Para mantener las cosas lo más limpias y separadas posible, abre una nueva ventana de terminal, asegúrate de tener $GOPATH configurado y $GOPATH/bin en tu PATH, y crea un nuevo directorio bajo $GOPATH llamado dev que representará nuestro espacio de desarrollo. Crearemos carpetas separadas para almacenar el estado de alice, bob y charlie, y ejecutaremos todos nuestros nodos lnd en diferentes puertos de localhost en lugar de usar Docker para facilitar un poco nuestra red.
```bash
# Crear nuestro espacio de desarrollo
cd $GOPATH
mkdir dev
cd dev

# Crear carpetas para cada uno de nuestros nodos
mkdir alice bob charlie
```

La estructura del directorio debería verse así:

```bash
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

Inicia el nodo de Alice desde dentro del directorio alice:
```bash
cd $GOPATH/dev/alice
alice$ lnd --rpclisten=localhost:10001 --listen=localhost:10011 --restlisten=localhost:8001 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek 
```

El nodo de Alice debería estar funcionando ahora y mostrando una salida que termina con una línea que comienza con "Esperando la contraseña de cifrado de la billetera".

Desglosando los componentes:

`--rpclisten`: El host:puerto para escuchar el servidor RPC. Esta es la forma principal en que una aplicación se comunicará con lnd
`--listen`: El host:puerto para escuchar las conexiones P2P entrantes. Esto es a nivel de red, y es distinto de las redes de canales Lightning y la red de Bitcoin/Litcoin en sí.
`--restlisten`: El host:puerto que expone una API REST para interactuar con lnd a través de HTTP. Por ejemplo, puedes obtener el balance del canal de Alice haciendo una solicitud GET a localhost:8001/v1/channels. Esto no es necesario para este tutorial, pero puedes ver algunos ejemplos aquí.
`--datadir`: El directorio en el que se almacenarán los datos de lnd
`--logdir`: El directorio para registrar la salida.
`--debuglevel`: El nivel de registro para todos los subsistemas. Puede establecerse en trace, debug, info, warn, error, critical.
`--bitcoin.simnet`: Especifica si usar simnet o testnet
`--bitcoin.active`: Especifica que bitcoin está activo. También puede incluir `--litecoin.active` para activar Litecoin.
`--bitcoin.node=btcd`: Usa el nodo completo btcd para interactuar con la blockchain. Ten en cuenta que cuando se usa Litecoin, la opción es `--litecoin.node=btcd`.
`--btcd.rpcuser` y `--btcd.rpcpass`: El nombre de usuario y la contraseña para la instancia btcd. Ten en cuenta que cuando se usa Litecoin, las opciones son `--ltcd.rpcuser` y `--ltcd.rpcpass`.

## Iniciando el nodo de Bob y el nodo de Charlie

Al igual que hicimos con Alice, inicia el nodo de Bob desde dentro del directorio bob, y el nodo de Charlie desde dentro del directorio charlie. Al hacerlo, se configurará el datadir y logdir en ubicaciones separadas para que nunca haya un conflicto.

Ten en cuenta que para cada ventana de terminal adicional que configures, necesitarás establecer `$GOPATH` e incluir `$GOPATH/bin` en tu PATH. Considera la posibilidad de crear un script de configuración que incluya las siguientes líneas:

```bash
export GOPATH=~/gocode # si seguiste exactamente la guía de instalación
export PATH=$PATH:$GOPATH/bin
```
y ejecútalo cada vez que inicies una nueva ventana de terminal trabajando en lnd.

Ejecuta Bob y Charlie:
```bash
# En una nueva ventana de terminal
cd $GOPATH/dev/bob
bob$ lnd --rpclisten=localhost:10002 --listen=localhost:10012 --restlisten=localhost:8002 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek 

# En otra ventana de terminal
cd $GOPATH/dev/charlie
charlie$ lnd --rpclisten=localhost:10003 --listen=localhost:10013 --restlisten=localhost:8003 --datadir=data --logdir=log --debuglevel=info --bitcoin.simnet --bitcoin.active --bitcoin.node=btcd --btcd.rpcuser=kek --btcd.rpcpass=kek
```

## Configurando lnd.conf
Para evitar tener que escribir un montón de flags en la línea de comandos cada vez, podemos modificar nuestro `lnd.conf`, y los argumentos especificados allí se cargarán en lnd automáticamente. Cualquier configuración adicional agregada como un argumento de línea de comandos se aplicará después de leer desde lnd.conf, y sobrescribirá la opción lnd.conf si corresponde.

En MacOS, `lnd.conf` se encuentra en: `/Users/[nombre de usuario]/Library/Application\ Support/Lnd/lnd.conf`
En Linux: `~/.lnd/lnd.conf`
Aquí hay un ejemplo de `lnd.conf` que puede ahorrarnos de volver a especificar un montón de opciones de línea de comandos:

```
[Application Options]
datadir=data
logdir=log
debuglevel=info

[Bitcoin]
bitcoin.simnet=1
bitcoin.active=1
bitcoin.node=btcd

[btcd]
btcd.rpcuser=kek
btcd.rpcpass=kek
```

Ahora, cuando iniciamos los nodos, solo tenemos que escribir
```
alice$ lnd --rpclisten=localhost:10001 --listen=localhost:10011 --restlisten=localhost:8001
bob$ lnd --rpclisten=localhost:10002 --listen=localhost:10012 --restlisten=localhost:8002
charlie$ lnd --rpclisten=localhost:10003 --listen=localhost:10013 --restlisten=localhost:8003
```
etc.

### Trabajando con lncli y autenticación
Ahora que tenemos nuestros nodos lnd en funcionamiento, ¡interactuemos con ellos! Para controlar lnd necesitaremos usar lncli, la interfaz de línea de comandos.

lnd utiliza macaroons para la autenticación en el servidor rpc. lncli normalmente busca un archivo admin.macaroon en el directorio principal de Lnd, pero como cambiamos la ubicación de nuestros datos de aplicación, tenemos que establecer `--macaroonpath` en el siguiente comando. Para desactivar los macaroons, pasa la bandera `--no-macaroons` tanto a lncli como a lnd.

lnd te permite cifrar tu billetera con una frase de contraseña y opcionalmente cifrar tu frase de contraseña de semilla de cifrado también. Esto se puede desactivar pasando `--noencryptwallet` a `lnd` o `lnd.conf`. Recomendamos pasar por este proceso al menos una vez para familiarizarte con las características de seguridad y autenticación alrededor de lnd.

Probaremos nuestra conexión rpc al nodo Alice. Observa que en el siguiente comando especificamos el `--rpcserver` aquí, que corresponde a `--rpcport=10001` que establecimos al iniciar el nodo lnd de Alice.

Abre una nueva ventana de terminal, establece `$GOPATH` e incluye `$GOPATH/bin` en tu `PATH` como de costumbre. Creemos la billetera de Alice y establezcamos su frase de contraseña:

```
cd $GOPATH/dev/alice
alice$ lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
```

Se te pedirá que ingreses y confirmes una contraseña de billetera para Alice, que debe tener más de 8 caracteres. También tienes la opción de agregar una frase de contraseña a tu semilla de cifrado. Por ahora, simplemente omite este paso ingresando "n" cuando se te pregunte si tienes una mnemotecnia existente, y presiona enter para continuar sin la frase de contraseña.

Ahora puedes solicitar información básica de la siguiente manera:

```
alice$ lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon getinfo
```
lncli acaba de hacer una llamada RPC al nodo lnd de Alice. Esta es una buena manera de probar si tus nodos están en funcionamiento y lncli está funcionando correctamente. Ten en cuenta que en futuras sesiones es posible que necesites llamar a lncli unlock para desbloquear el nodo con la contraseña que acabas de establecer.

Abre nuevas ventanas de terminal y haz lo mismo para Bob y Charlie. alice$ o bob$ denota la ejecución del comando desde la ventana lncli de Alice o Bob respectivamente.

```
# En una nueva ventana de terminal, estableciendo $GOPATH, etc.
cd $GOPATH/dev/bob
bob$ lncli --rpcserver=localhost:10002 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
# Ten en cuenta que tendrás que introducir una contraseña de 8+ caracteres y "n" para la mnemotecnia.

# En una nueva ventana de terminal:
cd $GOPATH/dev/charlie
charlie$ lncli --rpcserver=localhost:10003 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon create
# Ten en cuenta que tendrás que introducir una contraseña de 8+ caracteres y "n" para la mnemotecnia.
```
Para evitar escribir la bandera `--rpcserver=localhost:1000X` y `--macaroonpath` cada vez, podemos establecer algunos alias. Añade lo siguiente a tu `.bashrc`:

```
alias lncli-alice="lncli --rpcserver=localhost:10001 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
alias lncli-bob="lncli --rpcserver=localhost:10002 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
alias lncli-charlie="lncli --rpcserver=localhost:10003 --macaroonpath=data/chain/bitcoin/simnet/admin.macaroon"
```

Para asegurarte de que esto se aplicó a todas tus ventanas de terminal actuales, vuelve a ejecutar tu archivo `.bashrc`:

```
alice$ source ~/.bashrc
bob$ source ~/.bashrc
charlie$ source ~/.bashrc
```
Por simplicidad, el resto del tutorial asumirá que este paso se completó.

### Opciones de lncli
Para ver todos los comandos disponibles para lncli, simplemente escribe lncli --help o lncli -h.

## Configurando direcciones de Bitcoin
Creemos una nueva dirección de Bitcoin para Alice. Esta será la dirección que almacena el saldo en cadena de Alice. np2wkh especifica el tipo de dirección y significa Pago a Nested Witness Key Hash.

```
alice$ lncli-alice newaddress np2wkh
{
    "address": <DIRECCION_ALICE>
}
```
Y para Bob y Charlie:
```
bob$ lncli-bob newaddress np2wkh
{
    "address": <DIRECCION_BOB>
}
charlie$ lncli-charlie newaddress np2wkh
{
    "address": <DIRECCION_CHARLIE>
}
```
## Financiando a Alice
¡Eso es mucha configuración! En este punto, hemos generado direcciones en cadena para Alice, Bob y Charlie. Ahora, practicaremos trabajar con btcd y financiaremos estas direcciones con algunos Bitcoin de simnet.

Cierra btcd y vuelve a ejecutarlo, estableciendo a Alice como la receptora de todas las recompensas de minería:

`btcd --simnet --txindex --rpcuser=kek --rpcpass=kek --miningaddr=<DIRECCION_ALICE>`
Genera 400 bloques, para que Alice obtenga la recompensa. Necesitamos al menos 100 bloques porque los fondos de coinbase no se pueden gastar hasta después de 100 confirmaciones, y necesitamos alrededor de 300 para activar segwit. En una nueva ventana con $GOPATH y $PATH establecidos:

`alice$ btcctl --simnet --rpcuser=kek --rpcpass=kek generate 400`
Verifica que segwit está activo:

`btcctl --simnet --rpcuser=kek --rpcpass=kek getblockchaininfo | grep -A 1 segwit`
Verifica el saldo de la billetera de Alice.

`alice$ lncli-alice walletbalance`
No es divertido si solo Alice tiene dinero. Vamos a darle algo a Charlie también.

```
# Cierra el proceso btcd que se inició anteriormente con la dirección de minería de Alice,
# y luego reinícialo con:
btcd --txindex --simnet

 --

rpcuser=kek --rpcpass=kek --miningaddr=<DIRECCION_CHARLIE>

# Genera más bloques
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 100

# Verifica el saldo de Charlie
charlie$ lncli-charlie walletbalance
```
### Creando la Red P2P
Ahora que Alice y Charlie tienen algunos Bitcoin de simnet, comencemos a conectarlos.

Conecta a Alice con Bob:

```
# Obtén la clave pública de identidad de Bob:
bob$ lncli-bob getinfo
{
--->"identity_pubkey": <CLAVE_PUBLICA_BOB>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 450,
    "block_hash": "2a84b7a2c3be81536ef92cf382e37ab77f7cfbcf229f7d553bb2abff3e86231c",
    "synced_to_chain": true,
    "testnet": false,
    "chains": [
        "bitcoin"
    ],
    "uris": [
    ],
    "best_header_timestamp": "1533350134",
    "version":  "0.4.2-beta commit=7a5a824d179c6ef16bd78bcb7a4763fda5f3f498"
}

# Conecta a Alice y Bob
alice$ lncli-alice connect <CLAVE_PUBLICA_BOB>@localhost:10012
{

}
```
Observa que `localhost:10012` corresponde a la bandera `--listen=localhost:10012` que establecimos al iniciar el nodo lnd de Bob.

Verifiquemos que Alice y Bob ahora se conocen entre sí.
```
# Verifica que Alice ha agregado a Bob como un par:
alice$ lncli-alice listpeers
{
    "peers": [
        {
            "pub_key": <CLAVE_PUBLICA_BOB>,
            "address": "127.0.0.1:10012",
            "bytes_sent": "7",
            "bytes_recv": "7",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "0"
        }
    ]
}

# Verifica que Bob ha agregado a Alice como un par:
bob$ lncli-bob listpeers
{
    "peers": [
        {
            "pub_key": <CLAVE_PUBLICA_ALICE>,
            "address": "127.0.0.1:60104",
            "bytes_sent": "318",
            "bytes_recv": "318",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "5788"
        }
    ]
}
```
Termina la red P2P conectando a Bob con Charlie:

`charlie$ lncli-charlie connect <CLAVE_PUBLICA_BOB>@localhost:10012`

### Configurando la Red Lightning
Antes de poder enviar un pago, necesitaremos configurar los canales de pago de Alice a Bob, y de Bob a Charlie.

Primero, abramos el canal Alice<–>Bob.

`alice$ lncli-alice openchannel --node_key=<BOB_PUBKEY> --local_amt=1000000
--local_amt` especifica la cantidad de dinero que Alice comprometerá al canal. Para ver la lista completa de opciones, puedes probar lncli openchannel `--help`.
Ahora necesitamos minar seis bloques para que el canal se considere válido:

`btcctl --simnet --rpcuser=kek --rpcpass=kek generate 6`
Verifica que se creó el canal Alice<–>Bob:

```
alice$ lncli-alice listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": <BOB_PUBKEY>,
            "channel_point": "2622b779a8acca471a738b0796cd62e4457b79b33265cbfa687aadccc329023a:0",
            "chan_id": "495879744192512",
            "capacity": "1000000",
            "local_balance": "991312",
            "remote_balance": "0",
            "commit_fee": "8688",
            "commit_weight": "600",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",
            "pending_htlcs": [
            ],
            "csv_delay": 144,
            "private": false
        }
    ]
}
```

Enviando pagos de un solo salto
Finalmente, a la parte emocionante: ¡enviar pagos! Enviemos un pago de Alice a Bob.

Primero, Bob necesitará generar una factura:

```
bob$ lncli-bob addinvoice --amt=10000
{
        "r_hash": "<a_random_rhash_value>",
        "pay_req": "<encoded_invoice>",
        "add_index": 1
}
```
Envía el pago de Alice a Bob:

```
alice$ lncli-alice sendpayment --pay_req=<encoded_invoice>
{
	"payment_error": "",
	"payment_preimage": "baf6929fc95b3824fb774a4b75f6c8a1ad3aaef04efbf26cc064904729a21e28",
	"payment_route": {
		"total_time_lock": 650,
		"total_amt": 10000,
		"hops": [
			{
				"chan_id": 495879744192512,
				"chan_capacity": 1000000,
				"amt_to_forward": 10000,
                "expiry": 650,
                "amt_to_forward_msat": 10000000
			}
		],
		"total_amt_msat": 10000000
	}
}

# Verifica que el saldo del canal de Alice se decrementó en consecuencia:
alice$ lncli-alice listchannels

# Verifica que el canal de Bob fue acreditado con la cantidad del pago:
bob$ lncli-bob listchannels
```

## Multi-hop payments
Now that we know how to send single-hop payments, sending multi hop payments is not that much more difficult. Let’s set up a channel from Bob<–>Charlie:

```
charlie$ lncli-charlie openchannel --node_key=<BOB_PUBKEY> --local_amt=800000 --push_amt=200000

# Mine the channel funding tx
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 6
```

Note that this time, we supplied the `--push_amt` argument, which specifies the amount of money we want the other party to have at the first channel state.

Let’s make a payment from Alice to Charlie by routing through Bob:
```
charlie$ lncli-charlie addinvoice --amt=10000
alice$ lncli-alice sendpayment --pay_req=<encoded_invoice>

# Check that Charlie's channel was credited with the payment amount (e.g. that
# the `remote_balance` has been decremented by 10000)
charlie$ lncli-charlie listchannels
```

### Closing channels
For practice, let’s try closing a channel. We will reopen it in the next stage of the tutorial.

```
alice$ lncli-alice listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
       ---->"channel_point": "3511ae8a52c97d957eaf65f828504e68d0991f0276adff94c6ba91c7f6cd4275:0",
            "chan_id": "1337006139441152",
            "capacity": "1005000",
            "local_balance": "990000",
            "remote_balance": "10000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2"
        }
    ]
}
```
The Channel point consists of two numbers separated by a colon, which uniquely identifies the channel. The first number is funding_txid and the second number is output_index.

```
# Close the Alice<-->Bob channel from Alice's side.
alice$ lncli-alice closechannel --funding_txid=<funding_txid> --output_index=<output_index>

# Mine a block including the channel close transaction to close the channel:
btcctl --simnet --rpcuser=kek --rpcpass=kek generate 1

# Check that Bob's on-chain balance was credited by his settled amount in the
# channel. Recall that Bob previously had no on-chain Bitcoin:
bob$ lncli-bob walletbalance
{
    "total_balance": "20001",
    "confirmed_balance": "20001",
    "unconfirmed_balance": "0"
}
```
At this point, you’ve learned how to work with btcd, btcctl, lnd, and lncli. In Stage 2, we will learn how to set up and interact with lnd using a web GUI client.

In the future, you can try running through this workflow on testnet instead of simnet, where you can interact with and send payments through the testnet Lightning Faucet node. For more information, see the “Connect to faucet lightning node” section in the [Docker guide](https://dev.lightning.community/guides/docker/) or check out the [Lightning Network faucet repository](https://github.com/lightninglabs/lightning-faucet).

### Navigation
[Proceed to Stage 2 - Web Client](https://dev.lightning.community/tutorial/02-web-client)
[Return to main tutorial page](https://dev.lightning.community/tutorial/)

### Questions
Join the #dev-help channel on our [Community Slack](https://join.slack.com/t/lightningcommunity/shared_invite/enQtMzQ0OTQyNjE5NjU1LWRiMGNmOTZiNzU0MTVmYzc1ZGFkZTUyNzUwOGJjMjYwNWRkNWQzZWE3MTkwZjdjZGE5ZGNiNGVkMzI2MDU4ZTE)
Join IRC: [Irc](https://webchat.freenode.net/?channels=lnd)

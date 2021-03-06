# How Dat Works

Note this is about Dat 1.0 and later. For historical info about earlier incarnations of Dat (Alpha, Beta) check out [this post](http://dat-data.com/blog/2016-01-19-brief-history-of-dat).

When someone starts downloading data with the [Dat command-line tool](https://github.com/maxogden/dat), here's what happens:

## Phase 1: Source discovery

Dat links look like this: `dat://c3fcbcdcf03360529b47df32ccfb9bc1d7f64aaaa41cca43ca9ac7f6778db8da`. The link itself is a fingerprint of the data that is being shared. The first thing that happens when you go to download data using one of these links is you ask various discovery networks if they can tell you where to find sources that have a copy of the data you need.

The discovery step itself is a simple query: you supply a Dat link, and receive back the IP and port of all the known data sources online that have a copy of that data you are looking for. You can then connect to them and begin exchanging data. By introducing this discovery phase we are able to create a network where data can be discovered even if the original data source disappears.

The discovery protocols we use are [DNS name servers](https://en.wikipedia.org/wiki/Name_server), [Multicast DNS](https://en.wikipedia.org/wiki/Multicast_DNS) and the [Kademlia Mainline Distributed Hash Table](https://en.wikipedia.org/wiki/Mainline_DHT) (DHT). Each one has pros and cons, so we combine all three to increase the speed and reliability of discovering data sources.

We run a DNS name server at `discovery.publicbits.org` on port 53 that Dat clients can use (in addition to specifying their own if they need to), and are able to re-use existing DHT bootstrap servers on the Internet (but are planning on running our own for added redundancy). These discovery servers are the only centralized infrastructure we need for Dat to work over the Internet, and they are redundant, interchangeable and anyone can run their own. Every data source that has a copy of the data also advertises themselves across these discovery networks.

The discovery logic itself is handled by a module that we wrote called [discovery-channel](http://npmjs.org/discovery-channel), which wraps other modules we wrote to implement DNS and DHT logic into a single interface. We can give the Dat link we want to download to discovery-channel and we will get back all the sources it finds across the various discovery networks.

## Phase 2: Source connections

Up until this point we have just done searches to find who has the data we need. Now that we know who should talk to, we have to connect to them. We currently use TCP sockets are our primary transport protocol, and layer on our own file sharing protocol on top. We also have experimental support for [UTP](https://en.wikipedia.org/wiki/Micro_Transport_Protocol) which is designed to *not* take up all available bandwidth on a network (e.g. so that other people sharing your wifi can still use the Internet). We also are working on WebRTC support so we can incorporate Browser and Electron clients for some really open web use cases.

When we get the IP and port for a potential source we try to connect using all available protocols (currently TCP and sometimes UTP) and hope one works. If one connects first, we abort the other ones. If none connect, we try again until we decide that source is offline or unavailable to use and we stop trying to connect to them. Sources we are able to connect to go into a list of known good source, so that if our Internet connection goes down we can use that list to reconnect to our good sources again quickly.

If we get a lot of potential sources we pick a handful at random to try and connect to and keep the rest around as additional sources to use later in case we decide we need more sources. A lot of these are parameters that we can tune for different scenarios later, but have started with some best guesses as defaults.

The connection logic is implemented in a module called [discovery-swarm](https://www.npmjs.com/package/discovery-swarm). This builds on discovery-channel and adds connection establishment, management and statistics. You can see stats like how many sources are currently connected, how many good and bad behaving sources you've talked to, and it automatically handles connecting and reconnecting to sources for you. Our experimental UTP support is implemented in the module [utp-native](https://www.npmjs.com/package/utp-native), which you can manually install if you want to try it out with Dat.

## Phase 3: Data exchange

So now we have found data sources, have connected to them, but we havent yet figured out if they *actually* have the data we need. This is where our file transfer protocol [Hyperdrive](https://www.npmjs.com/package/hyperdrive) comes in. You can read a much longer description of how hyperdrive works in the [Hyperdrive Specification](https://github.com/mafintosh/hyperdrive/blob/master/SPECIFICATION.md).

The short version of how Hyperdrive works is that it breaks file contents up in to pieces, hashes each piece and then constructs a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) out of all of the pieces. This ultimately gives us the Dat link, which is the top level hash of the Merkle tree.

We use a technique called Rabin fingerprinting to break files up into pieces. Rabin fingerprints are a specific strategy for what is called Content Defined Chunking. Here's an example:

![cdc diagram](meta/cdc.png)

When two peers connect to each other and begin speaking the Hyperdrive protocol they can efficiently determine if they have chunks the other one wants, and begin exchanging those chunks directly. Hyperdrive gives us the flexibility to have random access to any portion of a file while still verifying the other side isnt sending us bad data. We can also download different sections of files in parallel across all of the sources simultaneously, which increases overall download speed dramatically.

## Phase 4: Data archiving

So now that you've discovered, connected, and downloaded a copy of some data you can stick around for a while and serve up copies of the data to other who come along and want to download it.

The first phase, source discovery, is actually an ongoing process. When you first search for data sources you only get the sources available at the time you did your search, so we make sure to perform discovery searches as often is practically possible to make sure new sources can be found and connected to.

Every user of Dat is a source as long as they have 1 or more chunks of data. Just like with other decentralized file sharing protocols you will notice Dat may start uploading data before it finishes downloading.

If the original source who shared the data goes offline it's OK, as long as other sources are available. As part of mission as a not-for-profit we will be working with various institutions to ensure there are always sources available to accept new copies of data and stay online to serve those copies for important datasets such as scientific research data, open government data etc.

Because Dat is built on a foundation of strong cryptographic data integrity and content addressable storage it gives us the possibility of implementing some really interesting version control techniques in the future. In that scenario archival data sources could choose to offer more disk space and archive every version of a Dat repository, whereas normal Dat users might only download and share one version that they happen to be interested in.

## Implementations

This covered a lot of ground. If you want to go deeper and see the implementations we are using in the [Dat command-line tool](https://github.com/maxogden/dat), here you go:

- [dat](https://www.npmjs.com/package/dat) - the main command line tool that uses all of the below
- [discovery-channel](https://www.npmjs.com/package/discovery-channel) - discover data sources
- [discovery-swarm](https://www.npmjs.com/package/discovery-swarm) - discover and connect to sources
- [hyperdrive](https://www.npmjs.com/package/hyperdrive) - exchange sets of files with many sources
- [hypercore](https://www.npmjs.com/package/hypercore) - exchange lwo level binary blocks with many sources
- [bittorrent-dht](https://www.npmjs.com/package/bittorrent-dht) - use the Kademlia Mainline DHT to discover sources
- [dns-discovery](https://www.npmjs.com/package/dns-discovery) - use DNS name servers and Multicast DNS to discover sources
- [utp-native](https://www.npmjs.com/package/utp-native) - UTP protocol implementation
- [rabin](https://www.npmjs.com/package/rabin) - Rabin fingerprinter stream
- [merkle-tree-stream](https://www.npmjs.com/package/merkle-tree-stream) - Used to construct Merkle trees from chunks

# dweb-transports  API
Mitra Ardron, Internet Archive,  mitra@mitra.biz

This doc provides a concise API specification for the Dweb Javascript Transports Libraries. 

It was last revised (to match the code) on 23 April 2018. 

If you find any discrepancies please add an issue here.

See [Dweb document index](./DOCUMENTINDEX.md) for a list of the repos that make up the Internet Archive's Dweb project, and an index of other documents. 
See [Writing Shims](./WRITINGSHIMS.md) for guidelines for adding a new Transport and adding shims to the Archive

## General API notes and conventions
We were using a naming convention that anything starting “p_” returns a promise so you know to "await" it if you want a result, 
this is being phased out as many functions now take a callback or promise.

Ideally functions should take a String, Buffer or where applicable Object as parameters with automatic conversion. 
And anything that takes a URL should take either a string or parsed URL object. 

Note that example_block.html collects this from the URL and passes it to the library, 
which is intended to be a good way to see what is happening.

Note: I am gradually (March2018) changing the API to take an opts {} dict. This process is incomplete, but I’m happy to see it accelerated if there is any code built on this, just let mitra@archive.org know.

## Overview

The Transport layer provides a layer that is intended to be independent of the underlying storage/transport mechanism.

There are a set of classes:
* *Transport*: superclass of each supported transport,
* *TransportHTTP*: Connects to generic http servers, and handles contenthash: through a known gateway
* *TransportIPFS*: Connects to IPFS, currently (April 2018) via WebSocketsStar (WSS)
* *TransportYJS*: Implements shared lists, and dictionaries. Uses IPFS for transport
* *TransportWEBTORRENT*: Integrates to Feross's WebTorrent library
* *TransportGUN*: Integrates to the Gun DB
* *Transports*: manages the list of conencted transports, and directs api calls to them. 

Calls are generally made through the Transports class which knows how to route them to underlying connections.

Documents are retrieved by a list of URLs, where in each URL, the left side helps identify the transport, and the right side can be the internal format of the underlying transport BUT must be identifiable e.g. ipfs:/ipfs/12345abc or https://dweb.me/contenthash/Qm..., this format will evolve if a standard URL for the decentralized space is defined. 

The actual urls used might change as the Decentralized Web universe reaches consensus (for example dweb:/ipfs/Q123 is an alternative seen in some places)

This spec will in the future probably add a variation that sends events as the block is retrieved to allow for streaming.

## Transport Class

Fields|&nbsp;
:---|---
options | Holds options passed to constructor
name|Short name of transport e.g. “HTTP”, “IPFS”
supportURLs|Array of url prefixes supported e.g. [‘ipfs’,’http’]
supportFunctions|Array of functions supported on those urls, current (April 2018) full list would be: `['fetch', 'store', 'add', 'list', 'reverse', 'newlisturls', "get", "set", "keys", "getall", "delete", "newtable", "newdatabase", "listmonitor"]`
status|Numeric indication of transport status: Started(0); Failed(1); Starting(2); Loaded(3)

### Setup of a transport
Transport setup is split into 3 parts, this allows the Transports class to do the first phase on all the transports synchronously, 
then asynchronously (or at a later point) try and connect to all of them in parallel. 

##### static setup0 (options)
First part of setup, create obj, add to Transports but dont attempt to connect, typically called instead of p_setup if want to parallelize connections.  In almost all cases this will call the constructor of the subclass
Should be synchronous and leave `status=STATUS_LOADED`
```
options        Object fields including those needed by transport layer
Resolves to    Instance of subclass of Transport
```
Default options should be set in each transport, but can be overwritten, 
for example to overwrite the options for HTTP call it with  
`options={ http: { urlbase: “https://dweb.me:443/” } }`

##### async p_setup1 (, cb) 
Setup the resource and open any P2P connections etc required to be done just once. 
Asynchronous and should leave `status=STATUS_STARTING` until it resolves, or `STATUS_FAILED` if fails.
```
cb (t)=>void   If set, will be called back as status changes (so could be multiple times) 
Resolves to    the Transport instance
```

##### async p_setup2 (, cb) 
Works like p_setup1 but runs after p_setup1 has completed for all transports. 
This allows for example YJS to wait for IPFS to be connected in TransportIPFS.setup1() 
and then connect itself using the IPFS object. 
```
cb (t)=>void   If set, will be called back as status changes (so could be multiple times) 
Resolves to    the Transport instance
```

##### async p_setup(options, cb) 
A deprecated utility to simply setup0 then p_setup1 then p_setup2 to allow a transport to be started
in one step, normally `Transports.p_setup` should be called instead.

#### togglePaused(cb)
Switch the state of the transport between STATUS_CONNECTED and STATUS_PAUSED, 
in the paused state it will not be used for transport but, in some cases, will still do background tasks like serving files. 
```
cb(transport)=>void a callback called after this is run, may be used for example to change the UI
```

##### async p_status ()
Check the status of the underlying transport. This may update the "status" field from the underlying transport.
```
returns:    a numeric code for the status of a transport.        
```

Code|Name|Means
---|---|---
0|STATUS_CONNECTED|Connected and can be used
1|STATUS_FAILED|Setup process failed, can rerun if required
2|STATUS_STARTING|Part way through the setup process
3|STATUS_LOADED|Code loaded but havent tried to connect
4|STATUS_PAUSED|It was launched, probably connected, but now paused so will be ignored by validfor()

##### async stop(refreshstatus, cb)
Stop the transport,
```
refreshstatus (optional)    callback(transport instance) to the UI to update status on display
cb(err, this)
```
### Transport: General storage and retrieval of objects
##### p_rawstore(data)
Store a opaque blob of data onto the decentralised transport.
```
data         string|Buffer data to store - no assumptions made to size or content
Resolves to  url of data stored
```

##### p_rawfetch(url, {timeoutMS, start, end, relay})
Fetch some bytes based on a url, no assumption is made about the data in terms of size or structure.

Where required by the underlying transport it should retrieve a number if its "blocks" and concatenate them.

There may also be need for a streaming version of this call, at this point undefined.
```
url          string url of object being retrieved in form returned by link or p_rawstore
timeoutMS    Max time to wait on transports that support it (IPFS for fetch)
start,end    Inclusive byte range wanted (must be supported, uses a "slice" on output if transport ignores it.
relay        If first transport fails, try and retrieve on 2nd, then store on 1st, and so on.
Resolves to  string The object being fetched, (note currently (April 2018) returned as a string, may refactor to return Buffer)
throws:      TransportError if url invalid - note this happens immediately, not as a catch in the promise
```

### Transport: Handling lists
##### p_rawadd(url, sig)
Store a new list item, it should be stored so that it can be retrieved either by "signedby" (using p_rawlist) 
or by "url"  (with p_rawreverse). 

The underlying transport can, but does not need to verify the signature, 
an invalid item on a list should be rejected by higher layers.
```
url        String identifying list to add sig to.
sig        Signature data structure (see below - contains url, date, signedby, signature)
    date        - date of signing in ISO format,
    urls        - array of urls for the object being signed
    signature   - verifiable signature of date+urls
    signedby    - url of data structure (typically CommonList) holding public key used for the signature
```
##### seed({directoryPath, fileRelativePath, ipfsHash, urlToFile, torrentRelativePath}, cb)
Seed the file to any transports that can handle it. 
```
ipfsHash:               When passed as a parameter, its checked against whatever IPFS calculates. 
                        Its reported, but not an error if it doesn't match. (the cases are complex, for example the file might have been updated).
urlFile:                The URL where that file is available, this is to enable transports (e.g. IPFS) that just map an internal id to a URL.
directoryPath:          Absolute path to the directory, for transports that think in terms of directories (e.g. WebTorrent) 
                        this is the unit corresponding to a torrent, and should be where the torrent file will be found or should be built
fileRelativePath:       Path (relative to directoryPath) to the file to be seeded. 
torrentRelativePath:    Path within directory to torrent file if present. 
```

##### p_rawlist(url)
Fetch all the objects in a list, these are identified by the url of the public key used for signing.
Note this is the 'signedby' parameter of the p_rawadd call, not the 'url' parameter. 
List items may have other data (e.g. reference ids of underlying transport)
```
url         String with the url that identifies the list
            (this is the 'signedby' parameter of the p_rawadd call, not the 'url' parameter
Resolves to Array: An array of objects as stored on the list. Each of which is …..
```
Each item of the list is a dict: {"url": url, "date": date, "signature": signature, "signedby": signedby}

##### p_rawreverse (url)
Similar to p_rawlist, but return the list item of all the places where the object url has been listed.
(not supported by most transports)
```
url         String with the url that identifies the object put on a list
            This is the “url” parameter of p_rawadd
Resolves to Array objects as stored on the list (see p_rawlist for format)
```

##### listmonitor (url, cb, { current}) 
Setup a callback called whenever an item is added to a list, typically it would be called immediately after a p_rawlist to get any more items not returned by p_rawlist.
```
url        Identifier of list (as used by p_rawlist and "signedby" parameter of p_rawadd
cb(obj)    function(obj)  Callback for each new item added to the list
current    true to send existing members as well as new
           obj is same format as p_rawlist or p_rawreverse
```

##### async p_newlisturls(cl) 
Obtain a pair of URLs for a new list. The transport can use information in the cl to generate this or create something random (the former is encouraged since it means repeat tests might not generate new lists). Possession of the publicurl should be sufficient to read the list, the privateurl should be required to read (for some transports they will be identical, and higher layers should check for example that a signature is signed.  
```
cl          CommonList instance that can be used as a seed for the URL
Returns     [privateurl, publicurl]
```

### Transport: Support for KeyValueTable
##### async p_newdatabase(pubkey) {
Create a new database based on some existing object
```
pubkey:     Something that is, or has a pubkey, by default support Dweb.PublicPrivate, KeyPair 
            or an array of strings as in the output of keypair.publicexport()
returns:    {publicurl, privateurl} which may be the same if there is no write authentication
```

##### async p_newtable(pubkey, table) {
Create a new table,
```
pubkey:     Is or has a pubkey (see p_newdatabase)
table:      String representing the table - unique to the database
returns:    {privateurl, publicurl} which may be the same if there is no write authentication
```

##### async p_set(url, keyvalues, value) 
Set one or more keys in a table.
```
url:            URL of the table
keyvalues:      String representing a single key OR dictionary of keys
value:          String or other object to be stored (its not defined yet what objects should be supported, e.g. any object ?
```

##### async p_get(url, keys)
Get one or more keys from a table
```
url:            URL of the table
keys:           Array of keys
returns:        Dictionary of values found (undefined if not found)
```

##### async p_delete(url, keys)
Delete one or more keys from a table
```
url:            URL of the table
keys:           Array of keys
```

##### async p_keys(url)
Return a list of keys in a table (suitable for iterating through)
```
url:            URL of the table
returns:            Array of strings
```

##### async p_getall(url)
Return a dictionary representing the table
```
url:            URL of the table
returns:            Dictionary of Key:Value pairs, note take care if this could be large.
```

### Transports - other functions
##### static async p_f_createReadStream(url, {wanturl, preferredTransports=[] })
Provide a function of the form needed by <VIDEO> tag and renderMedia library etc
```
url     Urls of stream
wanturl True if want the URL of the stream (for service workers)
preferredTransports: preferred order to select stream transports (usually determined by application)
returns f(opts) => stream returning bytes from opts.start || start of file to opts.end-1 || end of file
```

##### static createReadStream(urls, opts, cb)
Different interface, more suitable when just want a stream, now.
```
urls:   Url or [urls] of the stream
opts{
    start, end:   First and last byte wanted (default to 0...last)
    preferredTransports: preferred order to select stream transports (usually determined by application)
}
returns open readable stream from the net via cb or promise
```

##### supports(url, funcl)
Determines if the Transport supports url’s of this form. For example TransportIPFS supports URLs starting ipfs:
```
url       identifier of resource to fetch or list
Returns   True if this Transport supports that type of URL
throw     TransportError if invalid URL
```

##### validFor(url, func, opts)
True if the url and/or function is supported and the Transport is connected appropriately.

##### p_info() and info(cb)
Return a JSON with info about the server via promise or callback

## Transports class
The Transports Class manages multiple transports 

##### Properties
```
_transports         List of transports loaded (internal)
namingcb            If set will be called cb(urls) => urls  to convert to urls from names. 
_transportclasses   All classes whose code is loaded e.g. {HTTP: TransportHTTP, IPFS: TransportIPFS}
_optionspaused      Saves paused option for setup

```

##### static _connected()
```
returns Array of transports that are connected (i.e. status=STATUS_CONNECTED)
```

##### static async p_connectedNames(cb)
```
returns via Promise or cb: Array of names transports that are connected (i.e. status=STATUS_CONNECTED)
```
##### static async p_connectedNamesParm()
```
resolves to: part of URL string for transports e.g. 'transport=HTTP&transport=IPFS"
```

##### static async p_statuses()
```
resolves to: a dictionary of statuses of transports e.g. { TransportHTTP: STATUS_CONNECTED }
```

##### static validFor(urls, func, options) {
Finds an array or Transports that are STARTED and can support this URL. 
```
urls:       Array of urls
func:       Function to check support for: fetch, store, add, list, listmonitor, reverse
            - see supportFunctions on each Transport class
options     checks supportFeatures
Returns:    Array of pairs of url & transport instance [ [ u1, t1], [u1, t2], [u2, t1]]
```

##### static async p_urlsValidFor(urls, func, options) {
Async version of validFor for serviceworker and TransportsProxy


##### static http()
```
returns instance of TransportHTTP if connected
```

##### static ipfs()
```
returns instance of TransportIPFS if connected
```

##### static webtorrent()
```
returns instance of TransportWEBTORRENT if connected
```

##### static gun()
```
returns instance of TransportGUN if connected
```

##### static async p_resolveNames(urls)
See Naming below
```
urls    mix of urls and names
names   if namingcb is set, will convert any names to URLS (this requires higher level libraries)
```

##### static async resolveNamesWith(cb)
Called by higher level libraries that provide name resolution function. 
See Naming below
```
cb(urls) => urls    Provide callback function 
```
#### togglePaused(name, cb)
Switch the state of a named transport between STATUS_CONNECTED and STATUS_PAUSED, 
in the paused state it will not be used for transport but, in some cases, will still do background tasks like serving files. 
```
cb(transport)=>void a callback called after this is run, may be used for example to change the UI
```

##### static addtransport(t)
```
t:        Add a Transport instance to _transports
```

##### static setup0(transports, options, cb)
Calls setup0 for each transport based on its short name. Specially handles ‘LOCAL’ as a transport pointing at a local http server (for testing).
```
transports      Array of short names of transports e.g. [‘IPFS’,’HTTP’,’GUN’]
options         Passed to setup0 on each transport
cb              Callback to be called each time status changes
Returns:        Array of transport instances
```

##### static async p_setup1(, cb)
Call p_setup1 on all transports that were created in setup0().  Completes when all setups are complete. 

##### static async p_setup2(, cb)
Call p_setup2 on all transports that were created in setup0().  Completes when all setups are complete. 

##### static async refreshstatus(t)
Set the class of t.statuselement (if set) to transportstatus0..transportstatus4 depending on its status.
```
t   Instance of transport
```

##### static connect({options}, cb) 
```
options {
    transports  Array of abbreviations of transports e.g. ["HTTP","IPFS"] as provided in URL
    defaulttransports   Array of abbreviations of transports to use if transports is unset
    paused          Array of abbreviations of transports that should be paused (not started)
    statuselement   HTML element to build status display under
    ...             Entire options is passed to each setup0 and will include options for each Transport
    cb              If found called - no parameters else a promise is returned that resolves to undefined
```    
}

##### static async p_connect({options})
Main connection process for a browser based application, 
```
options - see connect() above
```

##### static async p_urlsFrom(url)
Utility to convert to urls form wanted for Transports functions, e.g. from user input
```
url:    Array of urls, or string representing url or representing array of urls
return: Array of strings representing url
```

##### static async p_httpfetchurl(url)
Return URLS suitable for caller to pass to fetch.
```
url:    Array of urls, or string representing url or representing array of urls
return: Array of strings representing url
```

### Multi transport calls 
Each of the following is attempted across multiple transports
For parameters refer to underlying Transport call

Call|Returns|Behavior
---|---|---
static async p_rawstore(data)|[urls]|Tries all and combines results
static async p_rawfetch(urls, {timeoutMS, start, end, relay})|data|See note
static async p_rawlist(urls)|[sigs]|Tries all and combines results
static async p_rawadd(urls, sig)||Tries on all urls, error if none succeed
static listmonitor(urls, cb, { current})||Tries on all urls (so note cb may be called multiple times)
static p_newlisturls(cl)|[urls]|Tries all and combines results
static async p_f_createReadStream(urls, options)|f(opts)=>stream|Returns first success
static async p_get(urls, keys)|currently (April 2018) returns on first success, TODO - will combine results and relay across transports
static async p_set(urls, keyvalues, value)|Tries all, error if none succeed
static async p_delete(urls, keys)|Tries all, error if none succeed
static async p_keys(urls|[keys]|currently (April 2018) returns on first success, TODO - will combine results and relay across transports
static async p_getall(urls)|dict|currently (April 2018) returns on first success, TODO - will combine results and relay across transports
static async p_newdatabase(pubkey)|{privateurls: [urls], publicurls: [urls]}|Tries all and combines results
static async p_newtable(pubkey, table)|{privateurls: [urls], publicurls: [urls]}|Tries all and combines results
static async p_connection(urls)||Tries all parallel
static monitor(urls, cb, { current})||Tries all sequentially

##### static async p_rawfetch(urls, {timeoutMS, start, end, relay})
FOR NEW CODE USE `fetch` instead of p_rawfetch

Tries to fetch on all valid transports until successful. See Transport.p_rawfetch
```
timeoutMS:   Max time to wait on transports that support it (IPFS for fetch)
start,end    Inclusive byte range wanted - passed to 
relay        If first transport fails, try and retrieve on 2nd, then store on 1st, and so on.
```

##### fetch(urls,  {timeoutMS, start, end, relay}, cb)
As for p_rawfetch but returns either via callback or Promise

## httptools
A utility class to support HTTP with or without TransportHTTP
e.g. `httptools.http().p_httpfetch("http://foo.com/bar", {method: 'GET'} )`

##### Common parameters
```
httpurl     HTTP or HTTPS url
wantstream  True if want a stream returned
retries     Number of times to retry on failure at network layer (i.e. 404 doesnt trigger a retry)
noCache     Add Cache-Control: no-cache header
```

##### p_httpfetch(httpurl, init, {wantstream=false, retries=undefined})
Fetch a url.

```
init:       {headers}
returns:    Depends on mime type;
    If application/json returns a Object, 
    If text/* returns text
    Oherwise    Buffer
```

##### p_GET(httpurl, {start, end, noCache, retries=12}, cb)
Shortcut to do a HTTP/POST get, sets `mode: cors, redirect: follow, keepalive: true, cache: default`
```
start:      First byte to retrieve
end:        Last byte to retrieve (undefined means end of file)
returns     (via Promise or cb) from p_httpfetch
```
Note that it passes start and end as the Range header, most servers support it, 
but it does not (yet) explicitly check the result. 

##### p_POST(httpurl, {contenttype, data, retries=0})
Shortcut to do a HTTP/HTTPS POST. sets same options as p_GET
```
data:           Data to send to fetch, typically the body, 
contenttype:    Currently not passed as header{Content-type} because fetch appears to ignore it.
returns     (via Promise or cb) from p_httpfetch
```

## TransportHTTP
A subclass of Transport for handling HTTP connections - both directly and for contenthash: urls.

It looks at `options { http }` for its options.

Option|Default|Meaning
------|-------|-------
urlbase|https://dweb.me:443|Connect to dweb.me for contenthash urls
supportURLS|http:*, https:*, contenthash:*}| (TODO: may in the future support `dweb:/contenthash/*`)
supportFunctions|fetch, store, add, list, reverse, newlisturls, get, set, keys, getall, delete, newtable, newdatabase|
supportFeatures|fetch.range, noCache|it will fetch a range of bytes if specified in {start, end} to p_rawfetch()


## TransportIPFS
A subclass of Transport for handling IPFS connections

The code works, however there are a number of key IPFS issues <TODO document here> that mean that not 
every IPFS `CID` will be resolvable, since:
* IPFS javascript doesnt yet support CIDv1 so can't recognize the CIDs starting with "z"
* WSS only can find blocks known about by peers connected to the same star. 
* urlstore on the archive.org gateway is not advertising the files in a way that WSS finds them. 
* Protocol Labs (IPFS)) have been unable to get WSS to connect to our gateway machine 

Other parts of this code work but just note that usage should be aware that problems may be to do
with your specific configuration and use cases.

It looks at `options { ipfs }` for its options.

Option|Default|Meaning
------|-------|-------
repo|/tmp/dweb_ipfsv2700|Local file system (only relevant if running under Node)
config Addresses |Swarm: ['/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star']}|Connects currently (April 2018) to ipfs.io, this may change.
EXPERIMENTAL pubsub|true|Requires pubsub for list monitoring etc

supportURLS = `ipfs:*` (TODO: may in the future support `dweb:/ipfs/*`)
SupportFunctions (note YJS uses IPFS and supports some other functions): 
    `fetch, store`
SupportFeatures: 
    fetch.range Not supported (currently May 2019))
    noCache:    Not actually supported, but immutable
    
Currently there is code for p_f_createReadStream. It works but because IPFS cannot return an error even if it 
cannot open the stream, IPFS is usually set as the last choice transport for streams.

## TransportYJS
A subclass of Transport for handling YJS connections.
Uses IPFS as its transport (other transports are possible with YJS and may be worth experimenting with)

supportURLS = `yjs:*` (TODO: may in the future support `dweb:/yjs/*`)
supportFunctions (note YJS uses IPFS and supports some other functions): 
    `fetch, add, list, listmonitor, newlisturls, connection, get, set, getall, keys, newdatabase, newtable, monitor`
supportFeatures: 
    fetch.range Not supported (currently May 2019)
    noCache:    Not supported (currently May 2019)
    
## TransportWEBTORRENT
A subclass of Transport for handling WEBTORRENT connections (similar to, with interworking with BitTorrent)

Note that currently (April 2018) this won't work properly in the ServiceWorker (SW) because SW dont support WebRTC
When used with a SW, it will attempt to retrieve from the http backup URL that is in many magnet links.
In the SW it will also generate errors about trackers because the only reason to use trackers is to get the WebRTC links.

supportURLS = `magnet:*` (TODO: may in the future support `dweb:/magnet/*`)

supportFunctions:
    `fetch`, `createReadStream`

supportFeatures: 
    fetch.range Not supported (currently May 2019)
    noCache:    Not actually supported, but immutable

## TransportGUN
A subclass of Transport for handling GUN connections (decentralized database)

supportURLS = `gun:*` (TODO: may in the future support `dweb:/gun/*`)

supportFunctions = `add`, `list`, `listmonitor`, `newlisturls`, `connection`, `get`, `set`, `getall`, `keys`, `newdatabase`, `newtable`, `monitor`
supportFeatures: 
    noCache:    Not supported (and cache flushing is hard)

## TransportWOLK 
A subclass of Transport for handling the WOLK transport layer (decentralized, block chain based, incentivised storage)

supportURLs = ['wolk'];

supportFunctions = [ 'fetch',  'connection', 'get', 'set',  ]; // 'store' - requires chunkdata; 'createReadStream' not implemented

## Naming
Independently from the transport, the Transport library can resolve names if provided an appropriate callback. 
See p_resolveNames(urls) and resolveNamesWith(cb)

In practice this means that an application should do.
```
require('@internetarchive/dweb-transports)
```
When setup this way, then calls to most functions that take an array of urls will first try and expand names.

The format of names currently (April 2018) is under development but its likely to be something like 
`dweb:/arc/archive.org/details/foo` 
to allow smooth integration with existing HTTP urls that are moving to decentralization. 


# Earthstar

![](docs/earthstar-logo.png)


*Early alpha - do not use for important data yet*

## An offline-first, distributed, syncable document database for use in p2p software

Implementations so far:
* Typescript (node, browsers) - this repo

Related tools:
* [earthstar-cli](https://github.com/cinnamon-bun/earthstar-cli)
* [earthstar-pub](https://github.com/cinnamon-bun/earthstar-pub)

### Data model

An Earthstar workspace holds mutable documents with unique paths, similar to leveldb or CouchDb.

There are no transactions.  The unit of atomic change is writing a value to one path.  Causal order is not preserved for edits across multiple paths.

![](docs/earthstar-data-model.png)

### Scope

Each user will have their own instance of a Earthstar database, maybe in their browser or embedded in an Electron app.  There might also be some on cloud servers.  These databases can sync with each other across HTTP or duplex stream connections.

Each database instance can hold a subset of the entire data, specified by a query such as "paths starting with a certain substring", "paths written after a certain timestamp", etc.

A "workspace" is a collection of data which is sync'd across database instances.  It's similar to a Slack workspace -- a collection of related people and data.  To join a workspace you need to know its unique ID, which is a random string.

### Security

Each author is identified by a public key.

Data updates are signed by the author, so it's safe for untrusted peers to help host and propagate data.  (Soon) you will also be able to encrypt data so untrusted peers can host it without reading it.

### Editing data; multiple users and devices; conflicts

A single author can use multiple devices at the same time.  You "log into" a device by providing your public and private key, like a username and password.

Data is mutable.  Authors can update paths with new values.  The old data is thrown away and doesn't take up space.  You can also delete paths (soon); this replaces them with a tombstone value which is kept around.

There is a write permission system to allow only certain authors to write to certain paths.  Other paths can be written by anyone.

Soon, there will also be a way to write immutable messages, for use cases where you need those.

Conflicts from a single author (on multiple devices) are resolved by timestamp and the old data is discarded.  Conflicts from multiple authors are resolved by timestamp by default, but the old data is kept to allow for manual conflict resolution if needed.

## Security properties

Malicious peers can:
* Withold updates to individual documents during a sync
* See the data (until encryption is implemented, soon)
* Know which peers they're connecting to

Malicious peers cannot:
* Alter documents from another author
* Forge new documents signed by another author

Because it allows partial sync and mutation of data, Earthstar doesn't cryptographically guarantee that you have a complete dataset with no gaps from a certain author.  Authors could chose to use sequential paths or embed a blockchain of hash backlinks into their messages, and you could verify these in your app (outside of Earthstar).

## Comparisons

Like leveldb, but:
* multi-author
* it syncs

Like CouchDb, but:
* with signatures and encryption, so untrusted nodes can help move data around.

Like Scuttlebutt, but:
* one author can use multiple devices
* data is mutable
* uses a key-value data model instead of event-sourcing, to reduce the need for custom indexing
* supports partial replication
* will sync over HTTP, so it's easy to host in places like Glitch.com
* has less strict data guarantees -- you can't tell if you have someone's entire dataset or not

Like DAT, but:
* multi-author, multi-device
* simpler

Like IPFS, but:
* multi-author, multi-device
* mutable

Sometimes immutability is needed, like if you're running a package registry or want an audit log.  For social-network style use cases, though, immutability can be a privacy liability instead of a desirable guarantee.

## Use cases

Earthstar can be used for invite-only tools where you have all the data (think Slack) and large open-world tools where you only replicate part of the entire dataset (think SSB).  Apps tell Earthstar what paths and what authors to replicate.

These styles of app could be built on top of Earthstar:
* wikis
* chat: Slack, IRC
* social: Facebook, Scuttlebutt, Discourse, LiveJournal, Forums
* microblogging: Twitter, Mastodon
* productivity: Trello, GitHub Issues, Asana, Todo lists
* office: Google sheets
* art: Collaborative drawing or music tools, multi-user Twine for making interactive fiction games

## What doesn't it do?

Earthstar doesn't handle multi-user text editing like Google Docs.  This is a different challenge.  You could probably use a CRDT like Automerge and rely on Earthstar to move and sync the patches it produces.

It's not real-time yet - changes propagate over a few minutes.  This will improve but might not ever get below 1s latency.

The cryptography is not audited or bulletproof yet.

Earthstar is not focused on anonymity -- more on autonomy.  You could use it over Tor or something.

Earthstar doesn't provide transactions or causality guarantees across paths.

Moderation and blocking support is not built in, but apps can build it on top of Earthstar.

## Example

The API for a Store:
```ts
// A Store is all about a single workspace.
// Workspaces are separate universes of data
// that don't sync with each other.
// You also choose which feed formats you want to support
// by supplying validators (one for each format).
constructor(validators : IValidator[], workspace : string)

// onChange is called whenever any data changes.
// it doesn't yet send any details about the changes.
// subscribe with onChange.subscribe(...cb...);
onChange: Emitter<undefined>;

// look up a path and get the corresponding value...
getValue(path: string): string | undefined;
// or get the whole document, which is an object with more details (author, timestamp...)
getDocument(path: string): Document | undefined;

// query with a variety of options - filter by paths and authors, etc
documents(query?: QueryOpts): Document[];
paths(query?: QueryOpts): string[];
values(query?: QueryOpts): string[];

// list all authors who have written
authors(): EncodedKey[];

// write a document to the database, which will be signed by your author key.
set(keypair: Keypair, docToSet: DocToSet): boolean;

// try to import an doc from another Store.
ingestDocument(doc: Document): boolean;

// basic sync algorithm.  a faster one will be made later.
sync(otherStore: IStorage, opts?: SyncOpts): SyncResults;
```

Usage example:
```ts
// Create a database for a particular workspace, 'gardening-pals'
// We've chosen to use the main 'es.2' feed format so we supply the matching validator
let es = new StoreMemory([ValidatorEs1], 'gardening-pals');

// Make up some authors for testing
// a keypair is { public: '@aaa', secret: 'xxx' }.
// they are encoded using base58btc
let keypair1 = generateKeypair();
let keypair2 = generateKeypair();
let author1 = keypair1.public;
let author2 = keypair2.public;

// It's a key-value store.
es.set('wiki/Strawberry', 'Tasty', author1, keypair1.secret);
es.getValue('wiki/Strawberry') // --> 'Tasty'

// One author can use multiple devices with no problems.
// Conflicts are resolved by timestamp.
// Here the same author overwrites their previous value,
// which is forgotten from the database.
es.set('wiki/Strawberry', 'Tasty!!', author1, keypair1.secret);
es.getValue('wiki/Strawberry') // --> 'Tasty!!'

// Multiple authors can overwrite each other (also by timestamp).
es.set('wiki/Strawberry', 'Yum', author2, keypair2.secret);
es.getValue('wiki/Strawberry') // --> 'Yum'

// We keep the one most recent value from each author, in case
// you need to do better conflict resolution later.
// To see the old values, use a query:
es.values({ path='wiki/Strawberry', includeHistory: true })
    // --> ['Yum', 'Tasty!!']  // newest first

// Get more context about a document, besides just the value.
es.getDocument('wiki/Strawberry')
/* {
    format: 'es.2',
    workspace: 'gardening-pals',
    path: 'wiki/Strawberry',
    value: 'Yum.',
    author: 'aaa',
    timestamp: 1503982037239,  // it's microseconds: Date.now()*1000
    signature: 'xxxxxxxx.sig.ed25519',
} */

// WRITE PERMISSIONS
//
// Paths can specify which authors are allowed to write to them.
// Author names that occur prefixed by '~' in a path can write to that path.
//
// Examples:
// One author write permission:
//   '~@aaa/about'  -- only @aaa can write here.
//   '~@aaa/follows/@bbb'  -- only @aaa can write here
// Public:
//   '@aaa/wall'  -- no tilde, so anyone can write here
//   'wiki/kittens'  -- no tilde, so anyone can write here
// Multiple authors:
//   'whiteboard/~@aaa~@bbb'  -- both @aaa and @bbb can write here
//
es.set(keypair1, '~' + author1 + '/about', '{name: ""}');

// Coming soon, the workspace can also have members with
// read or read-write permissions in general.

// Values are just strings.
// If you want JSON, you have to parse/stringify it yourself.
// If you want to store binary data, you should base64-encode it.
// To delete a message, maybe set it to "null" if you expect it to be JSON.
// This may improve later.

// You can do leveldb style queries.
es.path()
es.path({ lowPath: 'abc', limit: 100 })
es.path({ pathPrefix: 'wiki/' })

// You can sync to another Store
let es2 = new StoreMemory();
es.sync(es2);
// Now es and es2 are identical.

// Get notified when anything changes.
let unsub = es.onChange.subscribe(yourCallback);

// Later, you can turn off your subscription
unsub();

//------------------------------
// Upcoming features

// Soon you can provide some queries and only sync docs matching
// one or more of those queries.
es.sync(es2, [
    { prefix: 'about/' },
    { prefix: 'wiki/' },
]);

// Soon you can control who can join, read, and write in a workspace, but details TBD.
```
----
## Details

### Classes
* Store -- responsible for holding and querying the data.
* Validator -- for a specific feed format, checks validity of incoming messages and signs outgoing messages.

### Planned features:
* Workspaces - Like a Slack workspace / Discord server / scuttleverse.  Control who joins, block people, invitations, etc.
* Encryption - Wrap docs in an encrypted envelope.  They will still sync without being understood by middle-people.
* Metadata - Each doc can have its own small k-v metadata, which can help us query for things.  The Store will be responsible for indexing it.  The goal is simple general purpose indexing so apps don't need their own indexes.
* Namespaces? - Within a workspace, like top-level folders, to more easily control what to sync and keep different apps from stepping on each others' data.

### Docs table schema
```
{
    schema: 'es.1'  // for future updates
    workspace: utf-8 string, no '\n'
    path: utf-8 string, no '\n'
    value: utf-8 string
    timestamp: int in microseconds (milliseconds * 1000)
    author pubkey: ascii string, no '\n'
    signature by author: ascii string, no '\n'
}
primary key: (path, author) -- one doc per path per author.
```

### Feed formats
* The main feed format is called `es`.  It's versioned, like `es.1`.
* Feed formats are responsible for:
    * Validating incoming messages
    * Checking signatures
    * Signing outgoing messages
    * Setting rules about paths, for example enforcing url-safe characters
    * Encoding and decoding author pubkeys between raw buffers and a string format such as `'@'+base58(key)`
* There will eventually be a `ssb` feed format, which encapsulates SSB messages as key-value pairs.  It will be able to validate existing `ssb` signatures.  It will not enforce the hash chain backlinks because Earthstar can do partial syncs or sync in any order.

### All about paths
* typically you will think of them like file paths or URL paths
* requirements:
    * printable ASCII characters only
    * tilde `~` has special meaning for giving author write permissions
* recommendations:
    * preferred to limit to url-safe characters `'()-._~$&+,/:;=?@` so browsers won't percent-escape them
    * be aware using `/` will affect links in the browser (e.g. your content within a wiki) - you may need to set a <base> path.
* max length: TODO.  has to be pretty long to allow several authors to have write permissions.

### Path permissions
Paths can specify which authors are allowed to write to them.
Only authors that occur prefixed by a tilde `~` in a path can write to that path.

Public:
* `@aaa.ed25519/wall`  -- no tilde, so anyone can write here
* `wiki/kittens`  -- no tilde, so anyone can write here

One author write permission:
* `~@aaa.ed25519/about`  -- tilde means only @aaa can write here.
* `~@aaa.ed25519/follows/@bbb.ed25519`  -- only @aaa can write here
* `about/~@aaa.ed25519`  -- tilde does not have to be at the beginning of the path

Multiple authors:
* `whiteboard/~@aaa.ed25519~@bbb.ed25519`  -- both @aaa and @bbb can write here; nobody else can

### Encoding of crypto keys, signatures, etc
* For now, keys and signatures are encoded using base58btc.
* goals: URL safe, filename safe, no punctuation for easy doubleclicking, not case sensitive, widely supported by stdlibs, easy to implement, short enough for url location component (63 chars), sigils make sense, easy to regex, works as a link in markdown, works well in HTML
* base64url
    * https://github.com/commenthol/url-safe-base64#readme
* base58 and base58check - used by Bitcoin and IPFS
    * multibase - https://github.com/multiformats/multibase
* base32
* https://tools.ietf.org/html/rfc4648 - base 16, 32, 64
* sigils
    * author: `'@' + baseXX(pubkey)
    * signature: `baseXX(sig)`
    * pointer to a path: ?
    * pointer to a specific path version: `hash(message)`?
    * blobs are not used in this system.  Put your blobs under the path `blobs/xxxx` or something.

### Signatures
Document hashes are used within signatures and to link to specific versions of a doc.

There's a simple canonical way to hash a doc: mostly you just concat all the fields in a predefined order:
```
    sha256([
        doc.schema,
        doc.workspace,
        doc.path,
        sha256(doc.value),
        '' + doc.timestamp,
        doc.author,
    ].join('\n')).hexDigest();
```
None of those fields are allowed to contain newlines (except value, which is hashed for that reason) so newlines are safe to use as a delimiter.

To sign a doc, you sign its hash with the author's secret key.

Note that docs only ever have to get transformed INTO this representation just before hashing -- we never need to convert from this representation BACK into a real doc.

There is no canonical "feed encoding" - only the canonical way to hash an doc, above.  Databases and network code can represent the doc in any way they want.

The hash and signature specification may change as the schema evolves beyond `es.1`.  For example there may be another schema `ssb.1` which wraps and embeds SSB messages and knows how to validate their signatures.

### To validate incoming docs:
* check all the fields exist, have correct data types and no `\n`
* check signature
* check valid path string (no `\n`, etc)
* check write permission of path by author
* timestamp must be not too far in the future (10 minutes?)
    * skip individual docs when timestamp is in the future
    * maybe next time we sync that doc will be allowed by then

### To ingest to database:
* compare to existing newest doc for the same path
* check if it's newer by timestamp
* if older, ignore it
* if timestamps are equal, keep the one with lowest signature to break ties
* replace old value from same author, same path
* keep one old value from each different author in a path
    * (to help revert vandalism of publicly writable paths)
```
INSERT OR REPLACE INTO docs
WHERE path=:path
AND author=:author
```

Read a path:
```
SELECT * FROM docs
WHERE path=:path
ORDER BY timestamp DESC, signature ASC
LIMIT 1
```
* return undefined if not found

### Write a document:
* remember we're using microseconds, Date.now()*1000
* set timestamp to `max(now, path's highest existing timestamp + 1)` so we are the winner
* sign
* ingest
* TODO: consider using same timestamp to indicate batch writes (transactions)

When the app wants to delete a path:
* write it the usual way using `"null"` as the value
* this acts as a tombstone
* this tombstone value may be app-dependent, it's not part of the core Earthstar spec (TODO?)
* tombstones will be returned from queries just like other values

### Sync over duplex streams:
Here's a very simple but inefficient algorithm to start with:
```
    sort paths by (path, timestamp DESC, signature ASC)
    filter by my interest query
    while true:
        both sides send next 100 docs to each other as (path, timestamp, signature)
        both sides figure out what the other needs and send it in full
        both sides send 'ok'
```

### Sync over HTTP when only one peer is publicly available:
Here's a very simple but inefficient algorithm to start with:
```
    the client side is in charge and does these actions:
    sort paths by (path, timestamp DESC, signature ASC)
    filter by my interest query
    GET /interest-query   the server's interest query
    while true:
        GET /doc-versions?after='foo/bar'&limit=100 from server
        compare with my own docs
        pull things I need
            POST /batch-get-docs       paths=[foo, foo/a, foo/b, ...]
        push things they need:
            POST /batch-ingest-docs    docs=[ {...}, {...} ]
```

Note that everything happens in batches instead of infinite streams.  The code avoids using streams.

### HTTP API
Coming soon:
```
HTTP replication:

GET /interest-query   the server's interest query
    return a list of query objects -- what does the server want?
GET /doc-versions [after='...'] [limit=100] [before='...']
    return array of docs with just path, version, first N bytes of signature
POST /batch-get-docs       paths=[foo, foo/a, foo/b, ...]
    return complete docs
POST /batch-ingest-docs    docs=[ {...}, {...} ]
    ask server to write these docs
    return 'ok' or error
```

# TODO

### Encoding of values:
* Currently values are just utf8 strings
* Do we want to make it easier to hold other things, and encode/decode for the user?
* Maybe a prefix:
```
b:....  binary as base64
u:....  utf-8
j:....  json
d:      tombstone for deleted docs
```

### Path shorthand
* Use these special values in a path
* Signatures are based on the shorthand path
* Upon ingestion to the db, the values get replaced like a template string
* Database lookups use the full expanded path
* Across the network we only need to send the shorthand path
```
(@) for my own author pubkey, to save space
(hash) for the hash of this message, to make immutable docs
```
* `(hash)` lets you create docs that can never be overwritten, because you can never make another doc with the same hash.
    * ...if we make it use a character that's forbidden in regular paths

### Path overlays?
* It would be easy make an overlay view of different path prefixes ("folders") within a Store
* Or overlay two Store instances
* For example you could combine `~@a/wiki` and `~@b/wiki` into one namespace

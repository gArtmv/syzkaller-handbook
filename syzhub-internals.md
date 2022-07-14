# Syzhub internals

Syz-hub is a point of corpus exchange for a distributed fuzzing. It shares interesting programs between different OSes and devices. The term 'different OSes' means not only Windows and Linux but also different versions of Android or different configurations of devices - like Pixel 6 and Pixel 7 because they share some components, but differ in drivers.

Syz-Hub allows to combine in a fuzzing cluster devices spread all over the world. To join the cluster their syz-manager must provide in it's config a valid IP address, client name and key. The same name/key can be reused by mulitple instances of syz-manager. Manager name will be displayed in a syz-hub web-interface. To access it just type an external IP-address of a syz-hub host. Make sure the host allows HTTP protocol.

Beware of the following Syz-Hub behavior:
1. A syz-manager ran fuzzing for 100 hours and produced 15 000 new syscals (corpus, fuzzing programs)
2. This corpus was shared to syz-hub
3. The syz-manager changed it's config to enable fuzzing of only certain syscalls and restarted
4. All corpus not relevant to current configuration will be deleted locally.
5. Then all corpus produced by this syz-manager previously is also deleted from syz-hub.


## Code structure
Syz-Hub is implemented in three files (excluding tests) and all of them are located in syz-hub <a href="https://github.com/google/syzkaller/tree/master/syz-hub">directory:</a>
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/state/state.go">state/state.go</a> - code for working with corpus.db
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/http.go">http.go</a> - runs http server for web-interface in a separate thread. 
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/hub.go">hub.go</a>

Refers to syzkaller packages:
- db
- auth
- rpctype

## Configuration

Syz-hub has only a single argument `-config` which accepts a path to a config file. The configuration for syz-hub looks like this:

```json
{
        "http": ":80",
        "rpc":  ":55555",
        "workdir": "/syzkaller/workdir",
        "clients": [
                {"name": "manager1", "key": "abrakadabra"},
                {"name": "manager2", "key": "pseudo-random"},
                {"name": "manager3", "key": "some-hashed-key"}
        ]
}
```

`"clients"` section provides authorization keys. The same key can be reused by multiple syz-managers. They will be distinguished by their manager name specified in a config.
`"workdir"` is a directory with corpus database. 

## Workdir structure

```text
workdir/
├─ mali0_corpus.db
├─ corpus.db
├─ repro.db
├─ manager/
│  ├─ manager1/
│  │  ├─ corpus.db
│  │  ├─ domain
│  │  ├─ repro.seq
│  │  ├─ seq
│  ├─ manager2/
│  │  ├─ ............
```
- `mali0_corpus.db`
- `corpus.db`
- `repro.db` - has the same data-format as corpus.db
- `manager`
- `manager#\corpus.db`
- `manager#\domain`
- `manager#\repro.seq`
- `manager#\seq`


## Startup sequence

Main function is located in <a href="https://github.com/google/syzkaller/blob/master/syz-hub/hub.go">hub.go</a>. The code loades provided config file, creates a workdir as specified in config and initializes http and RPC servers.

At startup syz-hub visits `manager` folder and loads corpus stored for each manager. After this setup it calls `purgeCorpus` which does the following:
1. Iterate over all managers and note unique signatures of corpus stored in their corpus.db instances.
2. Inspect it's own corpus.db records and delete the ones not found in manager's databases.

`purgeCorpus` operation is surrounded by log messages:
```bash
purging corpus...
done, 10 programs
```

## What happens when a new manager is connected?

`Hub` is the central type for handling connections. It has the folowing interface:

```text
Hub
- Connect(a *rpctype.HubConnectArgs, r *int) error
- Sync(a *rpctype.HubSyncArgs, r *rpctype.HubSyncRes) error
- verifyKey(key, expectedKey string) error 
- checkManager(client, key, manager string) (string, error)
```

Hub checks syz-manager eligibility when it connects and when it synchronizes with it. `checkManager` uses `hub_client` (a setting in syz-manager config) to find a key and then validates the provided key. `verifyKey` uses two methods:
- char by char comparison
- using OAuth magic (TODO: What is it?) which assumes experation of a ticket
If the key has matched then its time to assign a name to the connected syz-manager. If the name was empty then `hub_client` is used. Otherwise manager name must start with `client_hub`, for example: andrey1 and andrey1-p21 - client_hub and manager name respectively. 

The connection is handled via `Connect` which accepts syz-manager info as specified in config (`name`, `hub_client`, `hub_key`). It locks hub instance and at this moment you should see in output:
```bash
2022/07/13 22:00:10 connect from andrey1-p21: domain=linux/ fresh=true calls=89 corpus=0
```
The data you see here (domain, fresh, calls, corpus) is provided by the connected syz-manager. 
The Hub instance immediately registers the new manager in the `state` instance which represents all manager as the following type:

```go
// Manager represents one syz-manager instance.
type Manager struct {
	name          string           // hub_client or name as decided by checkManager
	Domain        string           // Domain provided by mgr ('linux' for example)
	corpusSeq     uint64           // Number of corpus?
	reproSeq      uint64           // Number of repro?
	corpusFile    string           // workdir/mgr#/corpus.db
	corpusSeqFile string           // Where corpusSeq is stored ('workdir/mgr#/seq' file)
	reproSeqFile  string           // Where reproSeq is stored ('rworkdir/mgr#/epro.seq' file)
	domainFile    string           // Where domain info stored (workdir/mgr#/domain)
	ownRepros     map[string]bool
	Connected     time.Time         // time.Now()
	Added         int
	Deleted       int
	New           int
	SentRepros    int
	RecvRepros    int
	Calls         map[string]struct{} // Calls received from syz-manager
	Corpus        *db.DB            // DB manager to handle corpus.db
}
```

The input received from manager comes as an array of bytes. It can be rejected due to a bad format or due to a number of calls bigger than allowed.
Then State instance calculates signature of the received input as a hash and stores the hash only(!) in a local copy of the manager's DB. If the hash is not yet known to Syz-Hub then it is stored in the main corpus.db. So only unique inputs are present in the main corpus.db.

## How synchronization happens?

See `Sync` method inside <a href="https://github.com/google/syzkaller/blob/master/syz-hub/hub.go">hub.go</a>. It has two arguments 'a' (input from manager) and 'r' (results sent back). During each Sync Hub receives corpus to add and delete as well as request for repros.

```text
Input data:
- manager info (name, key, hub_client)
- corpus to add
- corpus to delete
- repros


Ouput data:
- Inputs or Progs
- More
- repros
```

At the end of the sync you should a log message like:
```text
2022/07/14 09:20:52 sync from andrey1-p21: recv: add=29 del=4 repros=0; send: progs=5 repros=0 pending=0
```
- `recv` - how many data was received from a manager    
    - `add` - corpus to add
    - `del` - corpus to delete
    - `repro` - repro corpus to add
- `send` - how many data was sent back as a result:
    - `progs`
    - `repros`
    - `pending`

Each Sync is followed by checking credentials of the manager. Then Hub acquires a lock and starts processing. Database update is delegated to the `State` instance. 
```go
hub.st.Sync(name, a.Add, a.Del)
```
First it checks whether the manager is known and connected. Otherwise you'll see 'unconnected manager manager-name'. Then it processes deletions. All signatures provided in delete list are removed from corresponding manager corpus.db copy and `purgeCorpus` is called which then removes them from main corpus.db
Then corpus to add is processed. Now Manager instance has updated Added, Deleted and New fields.

Now we are back to Hub instance. 
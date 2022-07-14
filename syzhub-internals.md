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

At startup syz-hub visits `manager` folder and loads corpus stored for each manager. After this setup it calls `pureCorpus` which does the following (see `state.Make`):
1. Iterate over all managers and note unique signatures of corpus stored in their corpus.db instances.
2. Inspect it's own corpus.db records and delete the ones not found in manager's databases.

`purgeCorpus` operation is surrounded by log messages:
```bash
purging corpus...
done, 10 programs
```


## Handshake protocol


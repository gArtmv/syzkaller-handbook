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
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/state/state.go">state/state.go</a>
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/http.go">http.go</a>
- <a href="https://github.com/google/syzkaller/blob/master/syz-hub/hub.go">hub.go</a>

## Configuration

The configuration for syz-hub looks like this:

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
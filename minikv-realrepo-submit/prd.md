# MiniKV Public Packet

## Overview

Build `minikv.py`, an in-memory data structure store library. It provides a single class for storing and manipulating strings, lists, sets, and hashes, with type enforcement and optional persistence.

This task is designed around the distinction between single-method correctness and system-level correctness. Individual methods should work on their own, but the library is only complete if type enforcement, data structure semantics, and cross-method state consistency hold across multi-step workflows.

The implementation language is Python 3.11. Place `minikv.py` at the root of your solution directory. Use only the Python standard library.

Public API:

```python
from minikv import QueueServer

server = QueueServer(use_gevent=False)
server.kv_set('name', 'Alice')
print(server.kv_get('name'))       # → 'Alice'
server.kv_incr('counter')
print(server.kv_get('counter'))    # → 1
server.lpush('items', 'a', 'b')
server.sadd('tags', 'python')
server.hset('user:1', 'name', 'Alice')
```

The benchmark does not inspect private implementation details.

## Feature Set

The product has six feature modules:

1. String key-value operations — kv_set, kv_get, kv_delete, kv_exists, kv_incr, kv_decr.
2. Bulk string operations — kv_mset, kv_mget.
3. List operations — lpush, rpush, lpop, rpop, lrange.
4. Set operations — sadd, srem, smembers, scard.
5. Hash operations — hset, hget, hdel, hgetall.
6. Key management and persistence — expire, kv_flush, save_to_disk, restore_from_disk.

These modules are intentionally state-dependent. Values set by kv_set are later read by kv_get. List elements pushed by lpush appear in lrange output. Type enforcement prevents operations on mismatched key types. State is mutable — methods like smembers and hgetall return references to internal data structures.

## Global Invariants

- A key's type is determined by the first operation that created it: kv_set with string → KV type; kv_set with list → QUEUE type; kv_set with dict → HASH type; lpush/rpush → QUEUE type; sadd → SET type; hset → HASH type.
- Type-restricted operations must raise `CommandError` on keys of the wrong type, preserving the key's original value and type.
- kv_delete removes the key entirely; the name can be reused with a different type.
- lpush inserts at the head; rpush appends at the tail. lrange uses Python slice semantics (start:end).
- sadd deduplicates members. smembers returns the actual internal set object (mutable).
- hset stores one field-value pair. hgetall returns the actual internal dict object (mutable).
- kv_incr auto-creates at 0. kv_incr/kv_decr on non-integer values raises CommandError.
- expire sets an expiry time. Expired keys are lazily removed.
- kv_flush clears all keys. save_to_disk persists state; restore_from_disk restores it.
- State is private to each QueueServer instance.

## Class: QueueServer

### Constructor

`QueueServer(use_gevent=False)`

Create a new data store instance. The `use_gevent` parameter can be ignored for this task — always pass `False`.

### String Operations

#### `kv_set(key, value)`
Store a value. Returns `1`. The type is auto-detected: dict → HASH, list → QUEUE, set → SET, anything else → KV (string). Overwrites any previous value and type.

#### `kv_get(key)`
Return the value, or `None` if the key does not exist or has expired.

#### `kv_delete(key)`
Remove the key. Returns `1` if deleted, `0` if not found.

#### `kv_exists(key)`
Return `1` if the key exists (not expired), `0` otherwise.

#### `kv_incr(key)`
Increment the integer value by 1. Creates at 0 if key does not exist. Raises `CommandError` on non-numeric values (anything that is not `int` or `float`). Returns the new value.

#### `kv_decr(key)`
Decrement by 1. Same semantics as kv_incr.

### Bulk Operations

#### `kv_mset(__data=None, **kwargs)`
Set multiple string keys from a dict and/or keyword arguments. Returns the count of keys set.

#### `kv_mget(*keys)`
Return a list of values. Missing keys appear as `None`.

### List Operations

#### `lpush(key, *values)`
Prepend values to the head of a list. Creates the list if it does not exist. Returns the number of values pushed.

#### `rpush(key, *values)`
Append values to the tail. Creates the list if it does not exist. Returns the number of values pushed.

#### `lpop(key)`
Remove and return the first element, or `None` if empty.

#### `rpop(key)`
Remove and return the last element, or `None` if empty.

#### `lrange(key, start, end=None)`
Return a list slice using Python slice semantics: `[start:end]`. `None` for end means "to the end". Raises `CommandError` on non-list keys.

### Set Operations

#### `sadd(key, *members)`
Add members to a set. Creates it if it does not exist. Returns the new cardinality.

#### `srem(key, *members)`
Remove members. Returns the count of members actually removed.

#### `smembers(key)`
Return the internal set object. If the key does not exist, creates an empty set first. Raises `CommandError` on non-set keys.

#### `scard(key)`
Return the cardinality. If the key does not exist, creates an empty set and returns `0`.

### Hash Operations

#### `hset(key, field, value)`
Set a field. Creates the hash if it does not exist. Returns `1`.

#### `hget(key, field)`
Return the field value, or `None` if not found.

#### `hdel(key, field)`
Delete a field. Returns `1` if deleted, `0` if not found.

#### `hgetall(key)`
Return the internal dict object. If the key does not exist, creates an empty hash first. Raises `CommandError` on non-hash keys.

### Key Management

#### `expire(key, nseconds)`
Set a timeout in seconds. Returns `None`.

#### `kv_flush()`
Remove all keys. Returns the number of keys removed.

#### `save_to_disk(filename)`
Persist the entire database state to a file using pickle. Returns `True`.

#### `restore_from_disk(filename)`
Restore database state from a pickle file. Returns `True` if restored, `False` if the file does not exist.

## Error Behavior

- `CommandError` is raised for type mismatches (e.g., lpush on a string key, hset on a list key, sadd on a hash key).
- `CommandError` is raised for kv_incr/kv_decr on non-integer string values.
- Error messages are descriptive strings (exact text not part of public API).

## Non-Goals

- No network server or client protocol. Only the Python class API.
- No RESP protocol, gevent integration, or connection handling.
- No sorted sets, streams, pub/sub, transactions, or Lua scripting.
- No scheduled task queue.

## Evaluation Style

Hidden tests are split into two scores:

- Unit tests exercise one feature module at a time using short Python snippets.
- System tests exercise interactions across at least two modules.

System tests are labeled by dimension: `cross_feature_dataflow`, `state_accumulation`, `global_invariant`, `error_atomicity`, `operation_order_sensitivity`, `boundary_crossing`.

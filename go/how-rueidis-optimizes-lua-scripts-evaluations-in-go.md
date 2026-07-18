---
title: How Rueidis Optimizes Lua Script Evaluations in Go
date: 2026-07-15
---

# How Rueidis Optimizes Lua Script Evaluations in Go

I recently needed to perform multiple Redis operations atomically for a hot endpoint. 

This endpoint receives thousands of requests concurrently all trying to achieve the same thing: decrement one value, increment another and enforce a condition before either operation succeeds.

This is where Redis lua scripts come in handy.

## Go Library

I've always used the Ruiedis library (https://github.com/redis/rueidis) for it's featureful Redis client implementation in Go

So I started by doing the typical code like this:
```go
script := rueidis.NewLuaScript(`
local remaining = tonumber(redis.call("GET", KEYS[1]) or "0") 
if remaining <= 0 then 
  return -1
end

redis.call('DECR', KEYS[1])
return redis.call('INCR', KEYS[2])
`)

err := script.Exec(ctx, client, []string{"key1", "key2"}).Error()
```

But then I wondered, is this going to send the lua script to the Redis server every time the function is invoked using `EVAL`?

Repeatedly sending the entire script would add unnecessary overhead, especially at this scale.

## Does Rueidis Send the Entire Script Every Time?

```go
// Check if SHA-1 is already loaded.
s.sha1Mu.RLock()
scriptSha1 = s.sha1
s.sha1Mu.RUnlock()
```

After finding the Exec() function for lua scripts, I saw that Rueidis starts by checking if a hash exists on the Lua struct.

We can see that a Read-Write Mutex is used to maintain concurrency access by the use of RLock() and RUnlock(). 

By obtaining the read lock using RLock(), we can safely read the sha1 value inside the struct.

This is important for applications that run code in parallel, otherwise we may end up with race conditions.

```go
if scriptSha1 == "" {
  s.sha1Mu.Lock()
  if s.sha1 == "" { // the double check
    result := c.Do(ctx, c.B().ScriptLoad().Script(s.script).Build().ToRetryable())
    if shaStr, err := result.ToString(); err == nil {
      s.sha1 = shaStr
    } else {
      s.sha1Mu.Unlock()
      return newErrResult(result.Error())
    }
  }
  scriptSha1 = s.sha1
  s.sha1Mu.Unlock()
}
```

The next few lines are very revealing on the optimization that Rueidis does.

It starts by obtaining a write lock. Once obtained, uses the command `SCRIPT LOAD` to load the script into the Redis server script cache. This command returns a hash of the script for future invokes.

We can then safely store it inside the Lua struct because we have the write lock.

```go
resp = c.Do(ctx, s.mayRetryable(c.B().Evalsha().Sha1(scriptSha1).Numkeys(int64(len(keys))).Key(keys...).Arg(args...).Build()))
```

Lastly, the library uses the `EVALSHA` command to evaluate the script with keys and arguments.

## Takeaway

Rueidis makes evaluations of lua scripts efficient by automatically handling loading of scripts for you.

1. It uses `SCRIPT LOAD` when the script hasn't been loaded before.
2. It uses `EVALSHA` after the script has been loaded on the Redis server.
3. It falls back to `EVAL` when Redis no longer has the script.
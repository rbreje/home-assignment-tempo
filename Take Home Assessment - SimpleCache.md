## Code Review

You are reviewing the following code submitted as part of a task to implement an item cache in a highly concurrent application. The anticipated load includes: thousands of reads per second, hundreds of writes per second, tens of concurrent threads.
Your objective is to identify and explain the issues in the implementation that must be addressed before deploying the code to production. Please provide a clear explanation of each issue and its potential impact on production behaviour.

```kotlin
import java.util.concurrent.ConcurrentHashMap

class SimpleCache<K, V> {
    private val cache = ConcurrentHashMap<K, CacheEntry<V>>()
    private val ttlMs = 60000 // 1 minute
    
    data class CacheEntry<V>(val value: V, val timestamp: Long)
    
    fun put(key: K, value: V) {
        cache[key] = CacheEntry(value, System.currentTimeMillis())
    }
    
    fun get(key: K): V? {
        val entry = cache[key]
        if (entry != null) {
            if (System.currentTimeMillis() - entry.timestamp < ttlMs) {
                return entry.value
            }
        }
        return null
    }
    
    fun size(): Int {
        return cache.size
    }
}
```

## Raul's Notes

### Issue 1 - Missing cleanup

Based on the current implementation, the old or expired entries are never removed from the map. Hence, the cache will grow indefinitely and the application will throw an OutOfMemory exception in no time due to the hundreds of additions expected per second. Depending on how the fault tolerance was handled here, the users might have a slower experience with the app or they might not be even able to use it at all.

### Issue 2 - Getter age checking

There is a high probability to retrieve an already outdated value because between checking the timestamp and returning the value, another thread might add a new entry with the same key. Hence, the getter doesn't provide a predictable behaviour. This might not be a big problem in a low intensive traffic, but it's for sure a huge problem when expecting thousands of reads and hundreds of writes. The data will not reflect the reality and the user experience is going to be affected.

### Issue 3 - Misleading cache size

The size method will not reflect the actual cache size because of the not removed stale values, and this gives always unexpected behaviour. However, if by any reasons the size of the cache is handled outside of its class, there might be nothing added anymore to it. This won't be a problem in case of a correct purging mechanism.
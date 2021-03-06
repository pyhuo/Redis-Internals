# streams![image title](http://www.zpoint.xyz:8080/count/tag.svg?url=github%2FRedis-Internals%2Fstreams)

# contents

* [prerequisites](#prerequisites)
* [related file](#related-file)
* [overview](#overview)
* [internal](#internal)
    * [xadd](#xadd)
    * [xdel](#xdel)
    * [xrange](#xrange)
    * [xread](#xread)
    * [consumer groups](#consumer-groups)
* [read more](#read-more)

# prerequisites

* [rax](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/rax/rax.md)(a redis implementation [radix tree](https://redis.io/topics/streams-intro))
* [redis listpack implementation](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/listpack/listpack.md)


# related file
* redis/src/stream.h
* redis/src/t_stream.c
* redis/src/rax.h
* redis/src/rax.c
* redis/src/listpack.h
* redis/src/listpack.c

# overview

Redis `streams` is introduced in version >= 5.0, it's a MQ like structure, it models a log data structure in a more abstract way, For more overview detail please refer to [Introduction to Redis Streams](https://redis.io/topics/streams-intro)

The block in red in the basic layout of `streams` structure

![streams](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/streams.png)

# internal

`stremas` is a new and special type, the `object encoding` returns `unknown` but it actually has a type named `OBJ_STREAM`

```shell script
127.0.0.1:6379> xadd mystream * key1 128
"1576480551233-0"
127.0.0.1:6379> object encoding mystream
"unknown"

```

Together with [rax](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/rax/rax.md) and [listpack](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/listpack/listpack.md) in [prerequisites](#prerequisites), there're totally 3 parts in `mystream`

The first part is `robj`, every redis object will have this basic part to indicate that the actual type, encoding and location of the target object stored in `ptr` field

The second part is `rax`, it's used for storing the `stream ID`

The third part is `listpack`, every key node in the `rax` stores the according `keys` and `values` in the `listpack` structure

![mystream](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/mystream.png)

### xadd

If we add a new value to the same key again

```shell script
127.0.0.1:6379> xadd mystream * key1 val1
"1576486352510-0"

```

![mystream2](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/mystream2.png)

We can see that the `stream` structure is used for storing data, `rax` points to a radix tree which stores first `ID` as entry and the `keys` and `values` in the according `listpack` associated with key node in `rax`

`length` indicate how many elements are currently stored inside the `stream`(actually number of all key value pair inside all the `listpack` in all key node in the `rax`)

`last_id` stores the latest `ID` generated by `stream` automatically or specific by caller

`cgroups` stores the information of `consumer groups` in `rax` data structure

```c
int streamAppendItem(stream *s, robj **argv, int64_t numfields, streamID *added_id, streamID *use_id) {
    raxIterator ri;
    raxStart(&ri,s->rax);
    raxSeek(&ri,"$",NULL,0);
    /* ... */
    /* Get a reference to the tail node listpack. */
    if (raxNext(&ri)) {
        lp = ri.data;
        lp_bytes = lpBytes(lp);
    }
    /* ... */
    /* The master entry is composed like in the following example:
     *
     * +-------+---------+------------+---------+--/--+---------+---------+-+
     * | count | deleted | num-fields | field_1 | field_2 | ... | field_N |0|
     * +-------+---------+------------+---------+--/--+---------+---------+-+
     * /
    if (lp != NULL) {
        if (server.stream_node_max_bytes &&
            lp_bytes > server.stream_node_max_bytes)
        {
            lp = NULL;
        } else if (server.stream_node_max_entries) {
            int64_t count = lpGetInteger(lpFirst(lp));
            if (count > server.stream_node_max_entries) lp = NULL;
        }
    }
    /* Populate the listpack with the new entry. We use the following
     * encoding:
     *
     * +-----+--------+----------+-------+-------+-/-+-------+-------+--------+
     * |flags|entry-id|num-fields|field-1|value-1|...|field-N|value-N|lp-count|
     * +-----+--------+----------+-------+-------+-/-+-------+-------+--------+
     *
     * However if the SAMEFIELD flag is set, we have just to populate
     * the entry with the values, so it becomes:
     *
     * +-----+--------+-------+-/-+-------+--------+
     * |flags|entry-id|value-1|...|value-N|lp-count|
     * +-----+--------+-------+-/-+-------+--------+
     * ...
     * /
}

```

We can learn from the above source code that when you insert a new item into the `stream` object, it will find the last `listpack` object and keep inserting to this `listpack` until it's full(size exceed `stream-node-max-bytes`(4kb by default) or length exceed `stream-node-max-entries`(100 by default))

### xdel

If we delete the last entry

```shell script
127.0.0.1:6379> xdel mystream 1576486352510-0
(integer) 1

```

![xdel](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xdel.png)

The only difference is the first bit in the `flag` is set, and the first two entries of `listpack`, num items become one less and num deleted become one more

The following is part of the source code

```c
void streamIteratorRemoveEntry(streamIterator *si, streamID *current) {
    unsigned char *lp = si->lp;
    int64_t aux;

    /* We do not really delete the entry here. Instead we mark it as
     * deleted flagging it, and also incrementing the count of the
     * deleted entries in the listpack header.
     *
     * We start flagging: */
    int flags = lpGetInteger(si->lp_flags);
    flags |= STREAM_ITEM_FLAG_DELETED;
    lp = lpReplaceInteger(lp,&si->lp_flags,flags);

    /* Change the valid/deleted entries count in the master entry. */
    unsigned char *p = lpFirst(lp);
    aux = lpGetInteger(p);

    if (aux == 1) {
        /* If this is the last element in the listpack, we can remove the whole
         * node. */
        lpFree(lp);
        raxRemove(si->stream->rax,si->ri.key,si->ri.key_len,NULL);
    } else {
        /* In the base case we alter the counters of valid/deleted entries. */
        lp = lpReplaceInteger(lp,&p,aux-1);
        p = lpNext(lp,p); /* Seek deleted field. */
        aux = lpGetInteger(p);
        lp = lpReplaceInteger(lp,&p,aux+1);

        /* Update the listpack with the new pointer. */
        if (si->lp != lp)
            raxInsert(si->stream->rax,si->ri.key,si->ri.key_len,lp,NULL);
    }

    /* Update the number of entries counter. */
    si->stream->length--;
    /* ... */
}


```

If we insert a new entry with same `key` and `value`

```shell script
127.0.0.1:6379> xadd mystream 1576486352510-0 key1 val1
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item

```

We can see that even if the specific `ID` is deleted, you are not able to insert to the same key with the deleted `ID`

```shell script
127.0.0.1:6379> xadd mystream 1576486352510-1 key1 val1
"1576486352510-1"

```

You must insert an `ID` bigger than the top item even if the the top item is deleted, because the comparsion is made between the newly inserted `ID` and the `last_id` field, the `last_id` doesn't store information about whether it's deleted or not

```c
if (use_id && streamCompareID(use_id,&s->last_id) <= 0) return C_ERR;

```

### xrange

We now know the data structure of `streams`, the algorithm `xrange` used is quiet intuitive

![xrange](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xrange.png)

If we send a command `xrange mystream ID3 ID4` to the server, and the location between begin and end looks like the above diagram

Follow the radix tree, find the first `listpack` with `master ID` lower than `ID3`, traverse the `listpack`, for each element in `listpack`, if the current ID(`master ID` + `offset`) is lower than `ID4` and greater than `ID3`, add it to the reply

If it's the final element in the current `listpack`, go to find the next key node in the `radix tree`

![xrange2](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xrange2.png)

### xread

```shell script
127.0.0.1:6379> XREAD COUNT 5 STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) "1576480551233-0"
         2) 1) "key1"
            2) "128"
      2) 1) "1576486352510-1"
         2) 1) "key1"
            2) "val1"
127.0.0.1:6379> XREAD COUNT 5 STREAMS mystream 1576486352510-2
(nil)
127.0.0.1:6379> XREAD COUNT 5 BLOCK 5000 STREAMS mystream 1576486352510-2
(nil)
(5.04s)


```

The `xread` will find out if the current specific `ID` is less than `streams->last_id`, if so, it can return to the client immediately, if not, check the `block` parameter to see whether it should be blocked or not

The alogrithm is the same as the one used in `xrange`

### consumer groups

If we create a `consumer group`

```shell script
127.0.0.1:6379> XGROUP CREATE mystream my_group 0
OK

```

And create a user for the `consumer group`

```shell script
127.0.0.1:6379> XREADGROUP GROUP my_group user1 COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1576480551233-0"
         2) 1) "key1"
            2) "128"

```

![xgroup](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xgroup.png)

What's in the `pel` field and `consumers` field of the `streamCG` ?

![xgroup_inner](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xgroup_inner.png)

If  we consume one more item

```shell script
127.0.0.1:6379> XREADGROUP GROUP my_group user1 COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1576486352510-1"
         2) 1) "key1"
            2) "val1"

```

![xgroup_inner2](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xgroup_inner2.png)

We now can know why the `ID` in the radix tree are stored in `big endian` order, because `ID` is time based in most case, the more closer the time gap between the items I inserted, the less space will be used and also means faster access

The first 5 bytes between these two `ID` are the same, so the first node is compressed, after the first node, the two seprated node contains the two different `ID`

The radix tree stores `ID` as key, and a `streamNACK` as value

`streamNACK` represents the

> Pending (yet not acknowledged) message in a consumer group.

The defination

```c
typedef struct streamNACK {
    mstime_t delivery_time;     /* Last time this message was delivered. */
    uint64_t delivery_count;    /* Number of times this message was delivered.*/
    streamConsumer *consumer;   /* The consumer this message was delivered to
                                   in the last delivery. */
} streamNACK;

```

If we ack the first `ID` as proceeded

```shell script
127.0.0.1:6379> XACK mystream my_group 1576480551233-0
(integer) 1

```

![xgroup_after_xack](https://github.com/zpoint/Redis-Internals/blob/5.0/Object/streams/xgroup_after_xack.png)

The acked `ID` is deleted from `streamCG->pel` and `streamConsumer->pel`

We can learn that `streamCG->pel` stores all the `ID` for all the users in the group, while `streamCG->consumers` stores all users inside it, and each user with a `streamConsumer` object, the `streamConsumer` has a copy of it's own `ID` stores as radix tree


# read more
* [Introduction to Redis Streams](https://redis.io/topics/streams-intro)

---
layout: post
title:  "Using Redis for complex job queue"
date:   2021-06-19 18:30:00 +0700
categories: [python, redis, queue, complex]
---

# This is in draft!!!

**Redis** Our problem is we have a job queue, each job is a msg in that queue. 
But job can be run and not yet be run due to:
- Out of resource
- Too many job of a same type that have already run
    - Or limit number of concurrent running to 1 to keep order of job in a same type

For a job/task queue, Redis can simply done that using List data structure with L/RPUSH and L/RPOP operator which is atomic already. We will not need to worry about conflict when 2 worker trying to update the queue (push/pop) at the same time.

```python
def push(conn, queue_name, msg_dict):
    conn.rpush(f"queue:{queue_name}", json.dumps(msg_dict))

def pop(conn, queue_name, deserialze=json.loads):
    while not QUIT:
        packed = conn.blpop([queue_name], 30)
        if not packed: 
            continue
        msg = deserialize(packed[1])
        if msg:
            return msg
```

Now for the next requirement: limit number of concurrent job for each type. This is more like a search base problem rather than a queue.
When meeting a job from queue, we will dispatch from job queue. And check:
1. If the current resource enough for job to run. 
2. If not, check if should we destroy current resource and create new resource or not.
3. 
4. 

```shell
#Take the intersection of two ordered sets of zkey1 and zkey2 and save to ordered set zkey3, with weights configured as 10 and 1 (zkey3 will be created if it does not exist, and overwritten if it exists)
# zkey3 score = (zkey1 score)*10 + (zkey2 score)*1 
# For example one score: 20 = 1*10+10*1
127.0.0.1:6379> zinterstore zkey3 2 zkey1 zkey2 weights 10 1
(integer) 2
127.0.0.1:6379> zrange zkey3 0 -1 withscores
1) "one"
2) "20"
3) "two"
4) "40"
```
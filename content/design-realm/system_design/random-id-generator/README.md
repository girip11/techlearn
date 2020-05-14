## Problem Statement

Suppose you are building a social network product like Twitter, you need to store each user in your database with a user ID, which is unique and used to identify each user in the system.

In some systems, you can just keep incrementing the ID from 1, 2, 3 … N. In other systems, we may need to generate the ID as a random string. Usually, there are few requirements for ID generators:

* They cannot be arbitrarily long. Let’s say we keep it within 64 bits.
* ID is incremented by date. This gives the system a lot of flexibility, e.g. you can sort users by ID, which is same as ordering by register date.


## Possible Approaches

There are four major possibilities here:

##### UUID

* UUIDs are 128-bit hexadecimal numbers that are globally unique. 
* The chances of the same UUID getting generated twice is negligible.
* _Problem:_ Very big in size and don’t index well. When dataset increases, the index size increases as well thereby impacting query performance.


##### MongoDB index
MongoDB’s ObjectIDs are 12-byte (96-bit) hexadecimal numbers that are made up of -

* a 4-byte epoch timestamp in seconds,
* a 3-byte machine identifier,
* a 2-byte process id, and
* a 3-byte counter, starting with a random value.

_Problem:_ The size is relatively longer than what we normally have in a single MySQL auto-increment field (a 64-bit bigint value).


##### Database Ticket Servers

This is the approach where one can simply maintain a table to store just the latest generated ID and every time a node 
asks for ID they make a ‘select for update’ on this table, update the value with a incremented value and use the 
selected value as the next ID.

This approach is resilient and distributed in nature. The ID generation can be separated from the actual data store. 

_Problem:_ There is a risk of Single Point of Failure.

It is being used by [Flickr](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)

Example:

```
CREATE TABLE `Tickets64` (
  `id` bigint(20) unsigned NOT NULL auto_increment,
  `stub` char(1) NOT NULL default '',
  PRIMARY KEY  (`id`),
  UNIQUE KEY `stub` (`stub`)
) ENGINE=MyISAM
```

```
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

For high availability run two servers one for even and other one for odd auto-increment.
```
TicketServer1:
auto-increment-increment = 2
auto-increment-offset = 1

TicketServer2:
auto-increment-increment = 2
auto-increment-offset = 2
```

##### Twitter Snowflake

[Twitter snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010) is a dedicated network service for generating 64-bit unique IDs at high scale. 
- Generated IDs are roughly time sortable.

The IDs are made up of the following components:

* Epoch timestamp in millisecond precision - 41 bits (gives us 69 years with a custom epoch)
* Configured machine id - 10 bits (gives us up to 1024 machines)
* Sequence number - 12 bits (A local counter per machine that rolls over every 4096)
* The extra 1 bit is reserved for future purposes. Since the IDs use timestamp as the first component, they are time sortable.

The IDs generated by twitter snowflake fits in 64-bits and are time sortable, which is great.

[Sample code:](https://www.callicoder.com/distributed-unique-id-sequence-number-generator/)

```java
import java.net.NetworkInterface;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Enumeration;

public class SequenceGenerator {
    private static final int UNUSED_BITS = 1; // Sign bit, Unused (always set to 0)
    private static final int EPOCH_BITS = 41;
    private static final int NODE_ID_BITS = 10;
    private static final int SEQUENCE_BITS = 12;

    private static final int maxNodeId = (int)(Math.pow(2, NODE_ID_BITS) - 1);
    private static final int maxSequence = (int)(Math.pow(2, SEQUENCE_BITS) - 1);

    // Custom Epoch (January 1, 2015 Midnight UTC = 2015-01-01T00:00:00Z)
    private static final long CUSTOM_EPOCH = 1420070400000L;

    private final int nodeId;

    private volatile long lastTimestamp = -1L;
    private volatile long sequence = 0L;

    // Create SequenceGenerator with a nodeId
    public SequenceGenerator(int nodeId) {
        if(nodeId < 0 || nodeId > maxNodeId) {
            throw new IllegalArgumentException(String.format("NodeId must be between %d and %d", 0, maxNodeId));
        }
        this.nodeId = nodeId;
    }

    public synchronized long nextId() {
        long currentTimestamp = timestamp();

        if(currentTimestamp < lastTimestamp) {
            throw new IllegalStateException("Invalid System Clock!");
        }

        if (currentTimestamp == lastTimestamp) {
            sequence = (sequence + 1) & maxSequence;
            if(sequence == 0) {
                // Sequence Exhausted, wait till next millisecond.
                currentTimestamp = waitNextMillis(currentTimestamp);
            }
        } else {
            // reset sequence to start with zero for the next millisecond
            sequence = 0;
        }

        lastTimestamp = currentTimestamp;

        long id = currentTimestamp << (NODE_ID_BITS + SEQUENCE_BITS);
        id |= (nodeId << SEQUENCE_BITS);
        id |= sequence;
        return id;
    }


    // Get current timestamp in milliseconds, adjust for the custom epoch.
    private static long timestamp() {
        return Instant.now().toEpochMilli() - CUSTOM_EPOCH;
    }

    // Block and wait till next millisecond
    private long waitNextMillis(long currentTimestamp) {
        while (currentTimestamp == lastTimestamp) {
            currentTimestamp = timestamp();
        }
        return currentTimestamp;
    }
}

```

## References

* [Callicoder - Distributed Unique Id Generator](https://www.callicoder.com/distributed-unique-id-sequence-number-generator/)
* [Gainlo - Random Id Generator](http://blog.gainlo.co/index.php/2016/06/07/random-id-generator/)
* [Unique Id generation in distributed systems](https://preparingforcodinginterview.wordpress.com/2017/03/21/unique-id-generation-in-distributed-systems/)
* [Flickr Ticket Servers](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
* [Twitter Snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)
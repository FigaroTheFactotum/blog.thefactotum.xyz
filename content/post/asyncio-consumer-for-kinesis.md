---
title: "Asyncio Consumer for Kinesis"
date: 2018-10-21T00:07:01+01:00
draft: false
tags: ["distributed systems", "aws", "kinesis", "dynamodb", "python", "asynchronous"]
image: /images/kinesis-streams.png
---

One of the most famous design patterns for distributed systems is the
_unified log_. Its widespread usage must be credited to Linkedin's
engineers that described and implemented one of the most used
streaming platform: [Kafka](https://kafka.apache.org/). The only drawback
that makes people hesitant when considering Kafka for their tech stack is
the maintenance it'll need which is not an option for small teams. There are
not many alternatives to Kafka when adopting a unified logs pattern, and
the only one fully managed is [Amazon Kinesis](https://aws.amazon.com/kinesis/).

Kinesis and Kafka share many characteristics even though the former has
more limits, like max 7-day retention, max number of API calls per minute.
The table below, from a talk at the
[re:Invent 2017](https://www.youtube.com/watch?v=a3713oGB6Zk), compares
all the AWS messaging services to Kafka.

![Compare AWS Services to Kafka](/images/aws-vs-ksfka.jpg)

If you're a Java developer, you're all set, and you can use
[KCL](https://github.com/awslabs/amazon-kinesis-client) to write you stream
consumer; but if you use any other language, your life won't be so easy.
Amazon suggests using the KCL MultiLangDaemon written in Java that manages
shard balancing, locks, etc. for you and exposes an API for other
languages using standard output and input of the child processes :thinking_face:
So, if anything gets logged in the standard output, the message
will be corrupted.

For this reason, I started porting some of the KCL features to Python
taking advantage of the Asyncio framework that I described in
[Inside The Event Loop](https://thefactotum.xyz/post/inside-the-event-loop/).

## Consumer Wishlist

he consumers will behave similarly to Kafka consumer groups:

* Only one consumer can process messages from a shard, except for some
cases described later.
* When an owner switch happens, the new owner starts processing from
where the last owner stopped.
* Consumers in different groups will process the same messages.

To achieve all these features, the consumers need a way to coordinate among
themselves. Since I'm using AWS, the best way to do this is using
[DynamoDB](https://aws.amazon.com/dynamodb/) that provides
some consistency guarantees that will allow me to implement a distributed
lock.

## Step 0: Consumer Skeleton

Let's start defining the main components we'll use.

**Message** is what the producer sent to the server to be delivered and
processed by the consumers.

```python
@dataclass
class Msg:
    """
    Representation of a message delivered by the stream.

    Attributes:
        id: indentifier used to update the shard state.
        payload: the actual data send by the producer.
        stream: name of the stream that delivered this message.
        shard_id: shard where this message was delivered.
        ts_server: when the server received the message.
    """

    id: str
    payload: bytes

    # metadata
    stream: str
    shard_id: str
    partition_key: str
    ts_server: datetime
```

**Shard** is a partition of the stream consumed by a single consumer, and
its **state** containing some metadata is stored in DynamoDB.

```python
@dataclass
class ShardState:
    """
    State of a single shard.

    Attributes:
        shard_id: shard identifier.
        shard_owner: the worker that is holding this lease. `None` if no
            consumer is currently consuming from the shard.
        checkpoint: the most recent checkpoint sequence number for the shard.
            This value is unique across all shards in the stream.
        initial_checkpoint: initializer for the first checkpoint sequence
            number.
        lease_counter: used for lease versioning so that workers can detect
            that their lease has been taken by another worker.
        owner_switches_since_checkpoint: how many times this lease has changed
            workers since the last time a checkpoint was written.
        parent_shard_id: used to ensure that the parent shard is fully
            processed before processing starts on the child shards.
            This ensures that records are processed in the same order they
            were put into the stream.
        expires: when the lease will end.
    """

    shard_id: str
    shard_owner: typing.Optional[str]
    checkpoint: str
    initial_checkpoint: str
    lease_counter: int
    owner_switches_since_checkpoint: int
    parent_shard_id: typing.Optional[str]
    expires: int

    @classmethod
    def from_item(cls, item: dict):
        """
        Builds an instance of `ShardState` from the item returned
        by DynamoDB.

        Argument:
            item: data store in DynamoDB.
        """
        if item["ParentShardId"].get("NULL"):
            parent_shard_id = None
        else:
            parent_shard_id = item["ParentShardId"]["S"]

        if item["Checkpoint"].get("NULL"):
            checkpoint = None

        else:
            checkpoint = item["Checkpoint"]["S"]

        if item["ShardOwner"].get("NULL"):
            shard_owner = None

        else:
            shard_owner = item["ShardOwner"]["S"]

        return cls(
            shard_id=item["ShardId"]["S"],
            shard_owner=shard_owner,
            checkpoint=checkpoint,
            initial_checkpoint=item["InitialCheckpoint"]["S"],
            lease_counter=int(item["LeaseCounter"]["N"]),
            owner_switches_since_checkpoint=int(
                item["OwnerSwitchesSinceCheckpoint"]["N"]
            ),
            parent_shard_id=parent_shard_id,
            expires=int(item["Expires"]["N"]),
        )


class Shard:
    """
    `Shard` is used to manage, lock, and fetch messages from a
    specific Kinesis shard.

    Arguments:
        kinesis: Kinesis client.
        dynamodb: DynamoDB client.
        table: name of the DynamoDB table used as shard lock.
        stream: stream's name.
        shard_id: stream's id.
        owner: consumer identifier.
        initial_checkpoint: policy used to initialize the pointer
            in case of a new a shard lock.
        lock_duration: how long the lock is held by the consumer
            before letting other consumer try to acquire it.
        logger: shard logger.
    """

    def __init__(
        self,
        *,
        kinesis,
        dynamodb,
        table,
        stream,
        shard_id,
        owner,
        initial_checkpoint,
        lock_duration,
        logger,
    ):
        self._kinesis = kinesis
        self._dynamodb = dynamodb
        self._table = table
        self._stream = stream
        self.shard_id = shard_id
        self._shard_owner = owner
        self._initial_checkpoint = initial_checkpoint
        self._lock_duration = lock_duration
        self._shard_iterator = None

        self._state = None

        self._logger = logger.bind(shard_id=shard_id)

    async def acquire(self) -> bool:
        """
        Acquires a lock on this shard.

        Returns:
            ShardState: the state of the shared when the lock
                was acquired.
        """

    async def checkpoint(self, msg_id: str) -> bool:
        """
        Updates the checkpoint in the shard state.
        """

    async def refresh(self) -> bool:
        """
        Updates the metadata in the shard state extending the lock
        if it's still held by this consumer.
        """

    async def release(self):
        """
        Release a lock on the shard.

        When a shard is still locked by this object, it resets
        the state and allows other consumers to lock this shard.
        """

    async def fetch(self, batch_size):
        """
        Gets a batch of messages from this shard.
        """
```

A **Consumer** is a single application processing data from
the stream that belongs to a **consumer Group** which is a collection
of _consumers_  that coordinates not to compete for
processing the same messages on a best-effort basis, it means that two
consumers might process the same message twice in some cases.

```python
class Consumer(abc.ABC):
    """
    A `Consumer` is a task that processes data delivered by a stream
    and coordinate with the other consumers in the same consumer group
    to balance the load in the best way they can.

    Arguments:
        kinesis: Kinesis client.
        dynamodb: DynamoDB client.
        table: table's name.
        stream: stream's name.
        group: consumer group identifier.
        owner: consumer identifier.
        initial_checkpoint: str = "LATEST",
        lock_duration: int = 60000,
        wait_time_acquire: wait time after trying to acquire a shard
            without success.
        batch_size: maximum number of messages to be fetched from the shard
            at every stream poll.
        logger: consumer logger.
    """

    def __init__(
        self,
        *,
        kinesis,
        dynamodb,
        table: str,
        stream: str,
        group: str,
        owner: str,
        initial_checkpoint: str = "LATEST",
        lock_duration: int = 60000,
        wait_time_acquire: int = 5000,
        batch_size: int = 100,
        logger,
    ) -> None:
        self._kinesis = kinesis
        self._dynamodb = dynamodb

        self._table = table
        self._owner = owner
        self._initial_checkpoint = initial_checkpoint
        self._batch_size = batch_size
        self._lock_duration = lock_duration
        self._wait_time_acquire = wait_time_acquire
        self._stream = stream
        self._group = group

        self._shard = None
        self._shard_iterator = None

        self._logger = logger.bind(table=table, stream=stream, group=group, owner=owner)

    async def _list_shards(self):
        resp = await self._kinesis.list_shards(StreamName=self._stream)

        while True:
            for shard in resp.get("Shards", []):
                yield Shard(
                    kinesis=self._kinesis,
                    dynamodb=self._dynamodb,
                    table=self._table,
                    stream=self._stream,
                    shard_id=shard["ShardId"],
                    owner=self._owner,
                    lock_duration=self._lock_duration,
                    initial_checkpoint=self._initial_checkpoint,
                    logger=self._logger,
                )

            next_token = resp.get("NextToken")
            if next_token is None:
                break

            resp = self._kinesis.list_shards(NextToken=next_token)

    async def _acquire_shard(self):
        self._logger.info("acquiring shard")

        resp = await self._kinesis.describe_stream(StreamName=self._stream)
        if resp["StreamDescription"]["StreamStatus"] != "ACTIVE":
            raise Exception("stream is not active")

        async for shard in self._list_shards():
            self._logger.info("acquiring shard")

            locked = await shard.acquire()

            if locked:
                self._logger.info("shard acquired", shard_id=shard.shard_id)

                self._shard = shard
                return True

            else:
                self._logger.info("cannot acquire shard", shard_id=shard.shard_id)

        else:
            return False

    async def _refresh_shard(self):
        self._logger.info("refreshing shard")
        locked = await self._shard.refresh()

        if not locked:
            self._shard = None

        return locked

    async def fetch(self):
        """
        Gets a batch of message from the stream to be processed
        by this consumer.
        """
        while True:
            if self._shard is None:
                locked = await self._acquire_shard()

            else:
                locked = await self._refresh_shard()

            if locked:
                self._logger.info("shard locked", shard_id=self._shard.shard_id)
                break

            else:
                self._logger.info("no shards available")
                await asyncio.sleep(self._wait_time_acquire / 1000.)

        messages = await self._shard.fetch(self._batch_size)

        return messages

    async def checkpoint(self, msg_id: str) -> bool:
        """
        Updates the shard state to reflect the progress made by the
        consumer when processing the messages from the stream.

        Arguments:
            pointer: identifier of last processed message ID.
        """
        ok = await self._shard.checkpoint(msg_id)

        if not ok:
            self._state = None

        return ok

    async def clean(self):
        self._logger.info("releasing shard")
        if self._shard is not None:
            await self._shard.release()

    async def run(self):
        """
        Start a consumer that will process all the batches it receives
        and coordinate with the other consumers in the group.
        """
        try:
            while True:
                batch = await self.fetch()

                await self.process_batch(batch)

                if batch:
                    await self.checkpoint(batch[-1].id)

        finally:
            await self.clean()

    @abc.abstractmethod
    async def process_batch(self, batch: typing.List[Msg]):
        """
        Subclasses will define this method that will be called with every
        new batch of messages.

        Arguments:
            batch: messages fetched from the stream.
        """
```

## Step 1: Lock the Shard

The main point is stopping consumers belonging to the same group
from processing the same shard, one way to achieve this is using
a distributed lock that I'll implement using DynamoDB.

![Lock steps](/images/kinesis-consumer-lock.png)

This implementation uses the DynamoDB's feature `ConsistentRead`
to avoid fetching an out-of-date state.

When a consumer acquires a lock, it gets assigned a lease for a
configured duration, and it uses it to check if any other consumer
acquired the lock in any condition expression
`ShardOwner = :current_owner AND LeaseCounter = :current_lease`.

```python
    async def acquire(self) -> bool:
        """
        Acquires a lock on this shard.

        Returns:
            ShardState: the state of the shared when the lock
                was acquired.
        """
        self._logger.info("trying to acquire lock")

        key = {"ShardId": {"S": self.shard_id}}

        now = int(time.time() * 1000)  # current time in ms
        expires = now + self._lock_duration

        resp = await self._dynamodb.get_item(
            TableName=self._table, Key=key, ConsistentRead=True
        )

        item = resp.get("Item")
        attributes = {
            ":new_shard_owner": {"S": self._shard_owner},
            ":new_expires": {"N": str(expires)},
        }

        if item is None:
            self._logger.info("lock does not exist yet")

            update = (
                "set ShardOwner = :new_shard_owner,"
                "Expires = :new_expires,"
                "Checkpoint = :checkpoint,"
                "InitialCheckpoint = :initial_checkpoint,"
                "LeaseCounter = :lease_counter,"
                "OwnerSwitchesSinceCheckpoint = :owner_switches_since_checkpoint,"
                "ParentShardId = :parent_shard_id"
            )
            condition = "attribute_not_exists(shard_id)"

            attributes[":checkpoint"] = {"NULL": True}
            attributes[":initial_checkpoint"] = {"S": self._initial_checkpoint}
            attributes[":lease_counter"] = {"N": "0"}
            attributes[":owner_switches_since_checkpoint"] = {"N": "0"}
            attributes[":parent_shard_id"] = {"NULL": True}

        else:
            self._logger.info("checking existing lock")

            self._state = ShardState.from_item(item)

            if (
                self._shard_owner != self._state.shard_owner
                and now < self._state.expires
            ):
                self._logger.info("lock has not been released yet")
                return False

            update = (
                "SET ShardOwner = :new_shard_owner, "
                "Expires = :new_expires, "
                "LeaseCounter = LeaseCounter + :one"
            )

            if self._shard_owner != self._state.shard_owner:
                self._logger.info("trying to switch ownership")

                update += ", OwnerSwitchesSinceCheckpoint = OwnerSwitchesSinceCheckpoint + :one"

            else:
                self._logger.info("trying to extend ownership")

            condition = "ShardOwner = :current_owner AND LeaseCounter = :current_lease"

            if self._state.shard_owner is None:
                attributes[":current_owner"] = {"NULL": True}

            else:
                attributes[":current_owner"] = {"S": self._state.shard_owner}

            attributes[":current_lease"] = {"N": str(self._state.lease_counter)}
            attributes[":one"] = {"N": "1"}

        try:
            resp = await self._dynamodb.update_item(
                TableName=self._table,
                Key=key,
                UpdateExpression=update,
                ConditionExpression=condition,
                ExpressionAttributeValues=attributes,
                ReturnValues="ALL_NEW",
            )

            self._state = ShardState.from_item(resp["Attributes"])

            return True

        except self._dynamodb.exceptions.ConditionalCheckFailedException:
            self._logger.info("lock was acquired by another process")

            return False

    async def fetch(self, batch_size):
        """
        Gets a batch of messages from this shard.
        """
        if self._shard_iterator is None:
            if self._state.checkpoint is None:
                iter_args = {"ShardIteratorType": self._state.initial_checkpoint}

            else:
                iter_args = {
                    "ShardIteratorType": "AFTER_SEQUENCE_NUMBER",
                    "StartingSequenceNumber": self._state.checkpoint,
                }

            resp = await self._kinesis.get_shard_iterator(
                StreamName=self._stream, ShardId=self.shard_id, **iter_args
            )

            self._shard_iterator = resp["ShardIterator"]

        resp = await self._kinesis.get_records(
            ShardIterator=self._shard_iterator, Limit=batch_size
        )

        self._logger.info("check consumer lag", lag=resp["MillisBehindLatest"])

        messages = [
            Msg(
                id=record["SequenceNumber"],
                payload=record["Data"],
                stream=self._stream,
                shard_id=self.shard_id,
                partition_key=record["PartitionKey"],
                ts_server=record["ApproximateArrivalTimestamp"],
            )
            for record in resp.get("Records", [])
        ]

        self._shard_iterator = resp.get("NextShardIterator")
        if self._shard_iterator is None:
            # TODO close shard
            pass

        return messages
```

## Step 2: Checkpoints and Heartbeats

If a consumer crashes, we don't want the next one acquiring the same shard
to start processing messages from the beginning of the stream again.
The consumer must update the shard state's field `Checkpoint` after
consuming the batch to mitigate messages being delivered more than once.
In the case of long-running tasks the consumer might be _marked_ as lost
by other consumers after the expiration time stored in the state, this
triggers an owner switch and lead to messages being processed by two consumers.
It's possible for the consumer to extend the lease even without commit
any new checkpoint, just refreshing the shard state that acting as heartbeats
that notify other consumers is the same group it's still alive.

```python

    async def checkpoint(self, msg_id: str) -> bool:
        """
        Updates the checkpoint in the shard state.
        """
        self._logger.info("updating checkpoint", msg_id=msg_id)
        key = {"ShardId": {"S": self.shard_id}}

        try:
            attrs = {
                ":new_sequence_number": {"S": str(msg_id)},
                ":lease_counter": {"N": str(self._state.lease_counter)},
                ":current_owner": {"S": self._shard_owner},
            }

            if self._state.checkpoint is None:
                attrs[":null"] = {"NULL": True}

                condition = "ShardOwner = :current_owner AND LeaseCounter = :lease_counter AND Checkpoint = :null"

            else:
                condition = "ShardOwner = :current_owner AND LeaseCounter = :lease_counter AND Checkpoint < :new_sequence_number"

            resp = await self._dynamodb.update_item(
                TableName=self._table,
                Key=key,
                UpdateExpression="set Checkpoint = :new_sequence_number",
                ConditionExpression=condition,
                ExpressionAttributeValues=attrs,
                ReturnValues="ALL_NEW",
            )

            self._state = ShardState.from_item(resp["Attributes"])

            return True

        except self._dynamodb.exceptions.ConditionalCheckFailedException:
            self._logger.info("cannot update checkpoint in lost lock")

            self._state = None
            return False

    async def refresh(self) -> bool:
        """
        Updates the metadata in the shard state extending the lock
        if it's still held by this consumer.
        """
        locked = await self.acquire()

        return locked

    async def release(self):
        """
        Release a lock on the shard.

        When a shard is still locked by this object, it resets
        the state and allows other consumers to lock this shard.
        """
        key = {"ShardId": {"S": self.shard_id}}

        update = "SET ShardOwner = :new_shard_owner, " "Expires = :new_expires"

        condition = "ShardOwner = :current_owner AND Expires = :current_expires"

        attributes = {
            ":current_owner": {"S": self._state.shard_owner},
            ":current_expires": {"N": str(self._state.expires)},
            ":new_shard_owner": {"NULL": True},
            ":new_expires": {"N": "0"},
        }

        try:
            await self._dynamodb.update_item(
                TableName=self._table,
                Key=key,
                UpdateExpression=update,
                ConditionExpression=condition,
                ExpressionAttributeValues=attributes,
                ReturnValues="ALL_NEW",
            )

        except self._dynamodb.exceptions.ConditionalCheckFailedException:
            self._logger.info("lock was acquired by another process")
```

## Step 3: Run the Consumer

To test the code, we can create a simple producer that
pushes messages to the Kinesis strean and a consumer that prints them
when it gets them from the stream.

```python
import asyncio
import aiobotocore


AWS_ACCESS_KEY_ID = "..."
AWS_SECRET_ACCESS_KEY = "..."
AWS_REGION = "..."

AWS_SESSION_CONFIG = {
    "region_name": AWS_REGION,
    "aws_access_key_id": AWS_ACCESS_KEY_ID,
    "aws_secret_access_key": AWS_SECRET_ACCESS_KEY,
}


# kinesis config
KINESIS_STREAM_ARN = "arn:aws:kinesis:eu-central-1:XXX:stream/messages"


class Printer(Consumer):
    async def process_batch(self, batch):
        for msg in batch:
            print(msg)

        await asyncio.sleep(5)


def run():
    logging.basicConfig(level=logging.INFO)

    loop = asyncio.get_event_loop()

    # prepare aws session
    session = aiobotocore.get_session(loop=loop)

    owner = "me-" + str(os.getpid())

    kinesis = session.create_client("kinesis", **AWS_SESSION_CONFIG)
    dynamodb = session.create_client("dynamodb", **AWS_SESSION_CONFIG)

    c = Printer(
        kinesis=kinesis,
        dynamodb=dynamodb,
        table="printer_group_a",
        stream="messages",
        group="printer_group_a",
        owner=owner,
        initial_checkpoint="TRIM_HORIZON",
        lock_duration=60000,
        logger=logger,
    )

    try:
        loop.run_until_complete(c.run())

    finally:
        loop.run_until_complete(c.clean())


if __name__ == "__main__":
    run()
```

## Conclusion

KCL implementation is not great for non-Java applications,
and this is a first attempt to get a Python version of that using `asyncio`.
Some feature that I'll add in the future are:

* Consumer processing messages from more than one shard.
* Re-balance load across different consumers.


# Preparation

1. Start Confluent Platform


    confluent local services schema-registry start
    
2. Create Topics


    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic customer_consents
    
    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic customer_accounts
    
3. Register Schema


    curl -X POST -H "Content-Type:application/json" -d"{\"schema\":$(jq tostring consent-schema.avsc)}" http://localhost:8081/subjects/customer_consents-value/versions

4. Generate Data


    kafka-console-producer --topic customer_accounts --broker-list localhost:9092 \
      --property parse.key=true \
      --property key.separator=";"
    
    user_1;{"customerId": "user_1", "contactMailAddress": "email_1"}
    user_2;{"customerId": "user_2", "contactMailAddress": "email_2"}
    user_3;{"customerId": "user_3", "contactMailAddress": "email_3"}
    user_4;{"customerId": "user_4", "contactMailAddress": "email_4"}
    user_5;{"customerId": "user_5", "contactMailAddress": "email_5"}
    user_6;{"customerId": "user_6", "contactMailAddress": "email_6"}
    user_7;{"customerId": "user_7", "contactMailAddress": "email_7"}
    user_8;{"customerId": "user_8", "contactMailAddress": "email_8"}
    
    
    kafka-avro-console-producer \
     --broker-list localhost:9092 \
     --topic customer_consents  \
     --property parse.key=true \
     --property key.separator=";" \
     --property key.serializer=org.apache.kafka.common.serialization.StringSerializer \
     --property schema.registry.url=http://localhost:8081 \
     --property value.schema="$(jq tostring -r consent-schema.avsc)"
    
    user_1;{"customerId": "user_1", "hashedCustomerId": "hashforuser1", "consents": {"retargeter1": true, "retargeter2": true}}
    user_2;{"customerId": "user_2", "hashedCustomerId": "hashforuser2", "consents": {"retargeter1": false, "retargeter2": false}}
    user_3;{"customerId": "user_3", "hashedCustomerId": "hashforuser3", "consents": {"retargeter1": true, "retargeter2": true}}
    user_4;{"customerId": "user_4", "hashedCustomerId": "hashforuser4", "consents": {"retargeter1": false, "retargeter2": false}}
    user_5;{"customerId": "user_5", "hashedCustomerId": "hashforuser5", "consents": {"retargeter1": true, "retargeter2": true}}
    user_6;{"customerId": "user_6", "hashedCustomerId": "hashforuser6", "consents": {"retargeter1": false, "retargeter2": false}}
    user_7;{"customerId": "user_7", "hashedCustomerId": "hashforuser7", "consents": {"retargeter1": true, "retargeter2": true}}
    user_8;{"customerId": "user_8", "hashedCustomerId": "hashforuser8", "consents": {"retargeter1": false, "retargeter2": false}}
    user_9;{"customerId": "user_9", "hashedCustomerId": "hashforuser9", "consents": {"retargeter1": true, "retargeter2": true}}
    user_10;{"customerId": "user_10", "hashedCustomerId": "hashforuser10", "consents": {"retargeter1": false, "retargeter2": false}}
  
---

# Demo 1: Basic Stream–Table Join

I’m running the Confluent Platform locally using the Confluent CLI.
You can see that it started Zookeeper, Kafka, and the Schema Registry.

    confluent local services status

At work, we don’t have a ksqlDB cluster yet.
Fortunately, I don’t need a cluster for my workaround.

I just start ksqlDB locally via docker – as shown in the Quickstart on the ksqlDB.io site.

I have a docker-compose.yml – copied from ksqlDB.io – that points to my locally running Confluent Platform.

    docker-compose up
    docker exec -it ksqldb-cli ksql http://localhost:8088


Let’s have a look at the data.
First, we can check what topics there are.

    show topics;

We can see the customer_accounts and customer_consents topics.
We can also look at the content of the topics.

    print 'customer_accounts' from beginning;
    print 'customer_consents' from beginning;

For the join, we want to include all existing data.
I set auto.offset.reset to earliest so that it includes data from the beginning of the topics.

    SET 'auto.offset.reset' = 'earliest';
    
 Now we create the table and the stream.
 
     CREATE TABLE customer_accounts (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR)
         WITH (kafka_topic='customer_accounts', value_format='json');
     
     SHOW TABLES;
     SELECT * FROM customer_accounts EMIT CHANGES LIMIT 5;
     
     CREATE STREAM customer_consents (key_customerid VARCHAR KEY)
         WITH (kafka_topic='customer_consents', value_format='avro');
     
     SHOW STREAMS;
     SELECT * FROM customer_consents EMIT CHANGES LIMIT 5; 


As a sidenote, ksql is also useful for producing messages.
I will update the consent for user 1 now:

    INSERT INTO customer_consents VALUES('user_1','user_1','hashforuser1',MAP('retargeter1':=false, 'retargeter2':=true));

    SELECT * FROM CUSTOMER_CONSENTS EMIT CHANGES;

This also gives you some validation – ksqlDB knows the schema.

    INSERT INTO customer_consents VALUES('user_1','user_1',false,MAP('retargeter1':=false, 'retargeter2':=true));


So now we can perform the join – just like in the tutorial.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           customer_accounts.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
    FROM customer_consents
    LEFT JOIN customer_accounts ON customer_consents.key_customerid = customer_accounts.key_customerid
    EMIT CHANGES LIMIT 10;

That looks almost good.
We have a pretty full table with all the customer IDs, the hashed customer IDs, and the consent to retargeter 2.
We also see a lot of the correct email addresses. So the join worked!
BUT: we see some nulls in the contactmailaddress column.

That is not what we expected! And for a legal issue the current output is not sufficient – we have to fix it.

I quickly realized that this was a data quality issue.
The customer_accounts topic was incomplete!

    print 'customer_accounts' from beginning;
    
=> BACK TO SLIDES

---

# Demo 2: Write Messages Manually

kcat is a powerful CLI tool to consume and produce Kafka messages.
Let’s look at a simple kcat command.

    kcat -b localhost:9092 -t customer_accounts -C -c5

We call kcat with my local Kafka broker and the topic customer_accounts.
We use -C for the consumer mode and -c5 to consume the first five messages in the topic.


To use Avro, I use kcat from docker – my Arch Linux installation of kcat refuses to consume Avro

    docker run --network="host" -t edenhill/kcat:1.7.1 -b localhost:9092 -t customer_consents -r http://localhost:8081 -s value=avro -C -c5



At this point, kcat looks comparable to to bundled Kafka command line tools, but we will shortly see that it is more powerful.

---

I queried the email addresses from the database into the file contactmails_dump

    cat contactmails_dump

Now we’ll create a new topic for the mails:

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmails

Now we will produce Kafka messages from the contactmails_dump using kcat.

    kcat -b localhost:9092 -t philip.contactmails -P -K ";" -z lz4 -T -l contactmails_dump
    
We use 
-P for the producer mode, 
-K to specify the key delimiter, 
-z lz4 to use lz4 compression, 
-T so that it prints the messages in the console, 
-l to produce one message per line 

Now we create a new table from this topic in ksqlDB.

    CREATE TABLE contact_mails_dump (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR)
        WITH (kafka_topic='philip.contactmails', value_format='JSON', wrap_single_value=false);
    
    SELECT * FROM contact_mails_dump EMIT CHANGES LIMIT 10;

I used wrap_single_value=false here.
The data in the messages are simple strings – not JSON objects with a key and a value.

Now let’s try our luck again and join to the new topic.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           contact_mails_dump.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
       FROM customer_consents
       LEFT JOIN contact_mails_dump ON customer_consents.key_customerid = contact_mails_dump.key_customerid
       EMIT CHANGES LIMIT 10;
       
Great – so now the entire contactmailaddress column is nulls!
That is a bit of a letdown.

If the entire column is null, then maybe the join did not work?

For a join to work, the two topics need to be co-partitioned.
That is, the keys have to match and every key must be in the same partition for both topics.

Let’s debug the join.

---

First, let’s see if the keys are the same.
We used different methods to produce the messages and we assume that the keys are simple strings that only contain the customer ID.
Here we can use kcat again to look at the keys in detail.

    kcat -b localhost:9092 -t philip.contactmails -C -c1 -f '%k' | hexdump -C
    
    kcat -b localhost:9092 -t customer_consents -C -c1 -f '%k' | hexdump -C

    echo 75 73 65 72 5f 31 | xxd -r -p
    echo 75 73 65 72 5f 38 | xxd -r -p

    //alternative: ASCII CONVERTER: https://www.rapidtables.com/convert/number/hex-to-ascii.html

We can see that the keys contain just the ASCII characters without any other bytes.
That is what we expected!
If the keys were using Avro, for example, there would be more to the keys than just the customer IDs.

So we have verified that the keys are good.
Now lets look at the partitions.
We will use kcat again.

If we had used Avro or Json schemas, we could see that there are additional bytes.
We can quickly check the values in stead of the keys, where we use Avro.

    kcat -b localhost:9092 -t customer_accounts -C -c1 -f '%s' | hexdump -C
    kcat -b localhost:9092 -t customer_consents -C -c1 -f '%s' | hexdump -C
    
The first five bytes of the Avro value are interesting.    
They stem from the Confluent Avro format.
The first byte is always 0.
The next four bytes represent the schema ID – here, it’s 1.

These bytes would not be visible in a string representation of the message.
So it’s conceivable that one might run into a situation where keys may have the same customerId, but the join still fails.

---

Now we will look at the partitions.
We will use kcat again.

    kcat -b localhost:9092 -t philip.contactmails -C -e -f '%k,%p\n' > partitioning_contactmails.csv
    
    kcat -b localhost:9092 -t customer_consents -C -e -f '%k,%p\n' > partitioning_consents.csv

Here we use the format flag -f and specify the format: %k represents the key, %p the partition.

    ctrl + shift + e
    
    sort -t , -k 1,1 partitioning_contactmails.csv > partitioning_contactmails_sorted.csv
    sort -t , -k 1,1 partitioning_consents.csv > partitioning_consents_sorted.csv

    cat partitioning_contactmails_sorted.csv
    cat partitioning_consents_sorted.csv

We can easily see that the partitions don’t match!

After some googling, I found out that kcat does not use the same default partitioner as other Kafka client!
Ironically, is both the tool that helped me debug the co-partitioning 
as well as the tool that is responsible for breaking the co-partitioning in the first place.

But we can specify the right partitioner when we produce data.

First, we’ll create a new clean topic contactmailsWithCorrectPartitions.

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmailsWithCorrectPartitions

Now we produce the data with kcat again, but this time with the murmur2_random partitioner.

    kcat -X topic.partitioner=murmur2_random -b localhost:9092 -t philip.contactmailsWithCorrectPartitions -P -K ";" -z lz4 -T -l contactmails_dump

Let’s compare the partitions again.
I will use kcat to write keys and their partitions to a file. Then use the sort command to sort the file.

    kcat -b localhost:9092 -t philip.contactmailsWithCorrectPartitions -C -e -f '%k,%p\n' > partitioning_contactmails_correctpartitions.csv

    sort -t , -k 1,1 partitioning_contactmails_correctpartitions.csv > partitioning_contactmails_correctpartitions_sorted.csv

Now we can use the join command to make the comparison easier.

    dos2unix partitioning_contactmails_correctpartitions_sorted.csv partitioning_consents_sorted.csv
    join -t , -1 1 -2 1 partitioning_contactmails_correctpartitions_sorted.csv partitioning_consents_sorted.csv > partitioning_joined.csv

        via  https://superuser.com/a/26869
        -t ,   : ',' is the field separator
        -k 1,1 : character sort on 1st field
        -1 1   : file 1, 1st field
        -2 1   : file 2, 1st field
        >      : output to file

We can now easily see that the last two numbers are always the same. 
The key on the left lands in the same partitions for both topics!

All right, the co-partitioning is fixed now.
Let’s try the join again!

    CREATE TABLE contact_mails_dump_correct (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR)
        WITH (kafka_topic='philip.contactmailsWithCorrectPartitions', value_format='JSON', wrap_single_value=false);
    
    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           contact_mails_dump_correct.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
       FROM customer_consents
       LEFT JOIN contact_mails_dump_correct ON customer_consents.key_customerid = contact_mails_dump_correct.key_customerid
       EMIT CHANGES LIMIT 10;

Now that was anticlimactic.
The contactmailaddress column is still empty!

=> BACK TO SLIDES

---

# Demo 3: Dealing with Time

Let’s try the time travel approach.

I prepared a file contactmails_dump_with_time

    cat contactmails_dump_with_time

Let’s write the messages with the timestamps to Kafka like before.

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmailsWithTime

    kcat -X topic.partitioner=murmur2_random -b localhost:9092 -t philip.contactmailsWithTime -P -K ";" -z lz4 -T -l contactmails_dump_with_time

Now we create a new table.

    CREATE TABLE contact_mails_dump_oldtimestamp (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR, artificialMessageTime VARCHAR)
        WITH (kafka_topic='philip.contactmailsWithTime', value_format='JSON', timestamp='artificialMessageTime', timestamp_format='yyyy-MM-dd HH:mm:ss');

We use the parameters timestamp and timestamp_format to use the attribute artificalMessageTime as our rowtime in ksqlDB.
We can check that with a query:
    
    SELECT rowtime, TIMESTAMPTOSTRING(rowtime, 'yyyy-MM-dd HH:mm:ss') AS rowtime_formatted, * FROM contact_mails_dump_oldtimestamp EMIT CHANGES LIMIT 10;

The rowtime / rowtime_formatted columns match the artificialmessagetime column we added to the dump.

Now we can use that new table for our join.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           contact_mails_dump_oldtimestamp.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
       FROM customer_consents
       LEFT JOIN contact_mails_dump_oldtimestamp ON customer_consents.key_customerid = contact_mails_dump_oldtimestamp.key_customerid
       EMIT CHANGES LIMIT 10;

You can see that now the contactmailaddress column is complete.

We have now succeeded an can perform a GDPR-compliant explicit opt-out for retargeter 2.
Now the marketing department can use retargeter 2 again for retargeting and everyone is happy.


## Table-Table Join

For completeness sake, we’ll have a look at the second option: the table–table join.

We create a table from the consents instead of a stream.

    CREATE TABLE consents_table (key_customerid VARCHAR PRIMARY KEY)
    WITH (kafka_topic='customer_consents', value_format='avro');
    
Now let’s join them: 

    SELECT consents_table.key_customerid AS key_customerid,
       consents_table.customerId,
       consents_table.hashedCustomerId,
       contact_mails_dump_correct.contactMailAddress,
       consents_table.consents['retargeter2'] AS consentToRetargeter2
    FROM consents_table
    FULL OUTER JOIN contact_mails_dump_correct ON consents_table.key_customerid = contact_mails_dump_correct.key_customerid
    EMIT CHANGES;

I used a FULL OUTER JOIN here so that we also see the additional messages where the contactmailaddress is null first.
But in the end, the contactmail messages will trigger another message with complete data.

You can see that for every one of the ten customers there is a complete message somewhere in there.




# Cleanup

    docker-compose down
    
    confluent local destroy
    
    clear

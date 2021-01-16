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

Ich habe die Confluent Platform lokal laufen über die Confluent CLI.
Ihr könnt sehen, dass Zookeeper, Kafka und die Schema Registry läuft.

    confluent local services status

Bei meinem Kunden haben wir noch kein ksqlDB Cluster.
Glücklicherweise brauche ich kein Produktions-ksqlDB-Cluster für meinen Workaround.

Ich starte ksqlDB lokal mit Docker – wie im Quickstart Tutorial von ksqlDB.io.

Ich habe eine docker-compose.yml – kopiert von ksqlDB.io – die  auf meine lokale Confluent Plattform zeigt.

    docker-compose up
    docker exec -it ksqldb-cli ksql http://localhost:8088


Werfen wir einen Blick auf die Daten.
Zuerst können wir schauen, welche Topics es gibt.

    show topics;

Wir können die customer_accounts und customer_consents Topics sehen.
Wir können auch den Inhalt der Topics ausgeben lassen.

    print 'customer_accounts' from beginning;
    print 'customer_consents' from beginning;


Für den Join wollen wir alle alten Daten benutzen – nicht nur die Nachrichten ab jetzt.
Dazu setze ich auto.offset.reset auf earliert, sodass alle Topics von Anfang an verarbeitet werden.

    SET 'auto.offset.reset' = 'earliest';
    
 Wir erzeugen die Tabelle und den Stream.
 
     CREATE TABLE customer_accounts (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR)
         WITH (kafka_topic='customer_accounts', value_format='json');
     
     SHOW TABLES;
     SELECT * FROM customer_accounts EMIT CHANGES LIMIT 5;
     
     CREATE STREAM customer_consents (key_customerid VARCHAR KEY)
         WITH (kafka_topic='customer_consents', value_format='avro');
     
     SHOW STREAMS;
     SELECT * FROM customer_consents EMIT CHANGES LIMIT 5; 
     
     
Nebenbei, kann man KSQL auch benutzen um Nachrichten zu schreiben.
Als Beispiel update ich den Consent für einen Benutzer.

    INSERT INTO customer_consents VALUES('user_1','user_1','hashforuser1',MAP('retargeter1':=false, 'retargeter2':=true));

    SELECT * FROM CUSTOMER_CONSENTS EMIT CHANGES;

Das gibt uns auch eine Validierung: ksqlDB kennt das Schema.

    INSERT INTO customer_consents VALUES('user_1','user_1',false,MAP('retargeter1':=false, 'retargeter2':=true));


Jetzt können wir den Join durchführen wie im Tutorial.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           customer_accounts.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
    FROM customer_consents
    LEFT JOIN customer_accounts ON customer_consents.key_customerid = customer_accounts.key_customerid
    EMIT CHANGES LIMIT 10;

Das sieht fast gut aus.
Wir haben eine ziemlich volle Tabelle mit allen customerIds und dem Consent für Retargeter 2.
Wir sehn auch viele befüllte E-Mail-Adressen – der Join hat funktioniert!
ABER: wir sehen leider auch in paar Nulls in der contactmailaddress Spalte.

Das ist nicht was wir erwartet haben.
Da wir das für ein rechtliches Problem machen, muss das aber vollständig sein.

Ich habe dann schnell realisiert, dass die Daten nicht vollständig sind.
Die customer_accounts waren unvollständig.

    print 'customer_accounts' from beginning;
    
=> BACK TO SLIDES

---

# Demo 2: Write Messages Manually

kafkacat ist ein mächtiges CLI Tool um Kafka nachrichten zu lesen und zu schreiben.
Wir schauen einen einfachen kafkacat Befehl an.

    kafkacat -b localhost:9092 -t customer_accounts -C -c5

Wir rufen kafkacat mit dem lokalen Kafka Broker und dem Topic customer_accounts auf.
Wir benutzen -C für den Consumer-Modus und -c5 um nur die ersten fünf Nachrichten im Topic zu lesen.


kafkacat kann auch Avro-Nachrichten konsumieren.
Dazu benutze ich kafkacat mittels docker – meine Arch Linux Installation von kafkakcat weigert sich, Avro zu benutzen.

    docker run --network="host" -t edenhill/kafkacat:1.6.0 -b localhost:9092 -t customer_consents -r http://localhost:8081 -s value=avro -C -c5


Bis jetzt sieht kafkacat ähnlich aus wie der kafka-console-consumer, der in Kafka enthalten ist.
Wir werden aber gleich sehen, dass es noch mehr kann.

---

Ich habe die E-Mail-Adressen auf der Datenbank in die Datei contactmails_dump gespeichert.

    cat contactmails_dump

Wir erstellen ein neues Topic für die Addressen.

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmails

Jetzt benutzen wir kafkacat um die Zeilen aus der Datei nach Kafka zu schreiben.

    kafkacat -b localhost:9092 -t philip.contactmails -P -K ";" -z lz4 -T -l contactmails_dump
    
We use 
-P for the producer mode, 
-K to specify the key delimiter, 
-z lz4 to use lz4 compression, 
-T so that it prints the messages in the console, 
-l to produce one message per line 

Jetzt können wir aus dem neuen Topic die Tabelle in ksqlDB erzeugen.

    CREATE TABLE contact_mails_dump (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR)
        WITH (kafka_topic='philip.contactmails', value_format='JSON', wrap_single_value=false);
    
    SELECT * FROM contact_mails_dump EMIT CHANGES LIMIT 10;

Ich benutze hier wrap_single_value=false.
Die Daten in den Nachrichten sind einfache Strings – keine JSON-Objekte mit Schlüssel und Wert.

Dann lasst uns schauen, ob der Join jetzt funktioniert.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           contact_mails_dump.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
       FROM customer_consents
       LEFT JOIN contact_mails_dump ON customer_consents.key_customerid = contact_mails_dump.key_customerid
       EMIT CHANGES LIMIT 10;
       
Toll – jetzt ist die ganze Spalte leer! Das ist ärgerlich.

Wenn die ganze Spalte leer ist, vielleicht hat der Join nicht funktioniert.

Für einen Join müssen beide Topics co-partitioniert sein.
Das heißt die Schlüssel müssen identisch sein und in beiden Topics in den gleichen Partitionen liegen.

Lasst uns den Join Debuggen.

---

Erst schauen wir uns an, ob die Schlüssel in beiden Topics gleich sind.
Wir haben verschiedenen Methoden benutzt, um die Nachrichten zu schreiben.
Wir nehmen an, dass die Schlüssel einfach simple Strings sind, die nur die customerId enthalten.

Mit kafkacat können wir uns das genau anschauen.

    kafkacat -b localhost:9092 -t philip.contactmails -C -c1 -f '%k' | hexdump -C
    
    kafkacat -b localhost:9092 -t customer_consents -C -c1 -f '%k' | hexdump -C

    echo 75 73 65 72 5f 31 | xxd -r -p
    echo 75 73 65 72 5f 38 | xxd -r -p

    //alternative: ASCII CONVERTER: https://www.rapidtables.com/convert/number/hex-to-ascii.html

Wir können sehen, dass die Schlüssel nur die ASCII-Zeichen enthalten ohne zusätzliche Bytes.
Das ist, was wir erwartet haben.

Wir haben also verifiziert, dass die Keys gleich sind.

Wenn wir Avro oder Json Schemas benutzt hätten, könnten wir sehen, dass zusätzliche Bytes existieren.
Das können wir kurz anhand der Values ausprobieren. 

    kafkacat -b localhost:9092 -t customer_accounts -C -c1 -f '%s' | hexdump -C
    kafkacat -b localhost:9092 -t customer_consents -C -c1 -f '%s' | hexdump -C
    
Die ersten fünf Bytes im Avro-Value sind dabei interessant.
Sie kommen aus dem Confuent Avro Format.
Das erste Byte ist bisher immer 0.
Die nächsten vier Bytes entsprechen der Schema ID – hier 1.

Diese Bytes wären in einer String-Repräsentation der Nachricht nicht sichtbar.
Es wäre also denkbar, dass man Schlüssel mit der gleichen customerId haben könnte, die sich trotzdem nicht joinen lassen.

---

Jetzt schauen wir uns die Partitionen an.
Wir benutzen wieder Kafkacat.

    kafkacat -b localhost:9092 -t philip.contactmails -C -e -f '%k,%p\n' > partitioning_contactmails.csv
    
    kafkacat -b localhost:9092 -t customer_consents -C -e -f '%k,%p\n' > partitioning_consents.csv

Wir benutzen das "format" Flag -f und geben das Vormat vor: %k repräsentiert den key, %p die Partition.

    ctrl + shift + e
    
    sort -t , -k 1,1 partitioning_contactmails.csv > partitioning_contactmails_sorted.csv
    sort -t , -k 1,1 partitioning_consents.csv > partitioning_consents_sorted.csv

    cat partitioning_contactmails_sorted.csv
    cat partitioning_consents_sorted.csv

Wir können schnell sehen, dass die Partitionen nicht matchen!

Nach etwas googlen habe ich gefunden, dass kafkacat nicht den selben Standard-Partitioner benutzt wie andere Kafka Clients.
Ironischerweise ist kafkacat jetzt sowohl das Tool, das mir geholfen hat, das Co-Partitioning zu debuggen,
als auch das Tool, das dafür verantwortlich war, dass das Co-Partitioning gebrochen ist.

Wir können den richtigen Partitioner angeben wenn wir Daten schreiben.

Wir erstellen erst ein neues, saubeser Topic contactmailsWithCorrectPartitions.

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmailsWithCorrectPartitions

Jetzt produzieren wir die Daten erneut mit kafkacat. Dieses Mal mit murmur2_random Partitioner.

    kafkacat -X topic.partitioner=murmur2_random -b localhost:9092 -t philip.contactmailsWithCorrectPartitions -P -K ";" -z lz4 -T -l contactmails_dump

Wir vergleichen wieder die Partitionierung.
Ich benutze kafkacat um die Schlüssel und Partitionen in eine Datei zu schreiben und sortiere die Datei.

    kafkacat -b localhost:9092 -t philip.contactmailsWithCorrectPartitions -C -e -f '%k,%p\n' > partitioning_contactmails_correctpartitions.csv

    sort -t , -k 1,1 partitioning_contactmails_correctpartitions.csv > partitioning_contactmails_correctpartitions_sorted.csv

Jetzt können wir den Join-Befehl nutzem, um den Vergleich einfacher zu machen.

    dos2unix partitioning_contactmails_correctpartitions_sorted.csv partitioning_consents_sorted.csv
    join -t , -1 1 -2 1 partitioning_contactmails_correctpartitions_sorted.csv partitioning_consents_sorted.csv > partitioning_joined.csv

        via  https://superuser.com/a/26869
        -t ,   : ',' is the field separator
        -k 1,1 : character sort on 1st field
        -1 1   : file 1, 1st field
        -2 1   : file 2, 1st field
        >      : output to file

Wir können jetzt einfach sehen, dass die letzten zwei Ziffern immer gleich sind.
Der Schlüssel links landet in beiden Topics in der gleichen Partition.

So, jetzt wo das Co-Partitioning gefixt ist, können wir den Join wieder probieren.

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

Das ist enttäuschend. Die Spalte ist immer noch leer.

=> BACK TO SLIDES

---

# Demo 3: Dealing with Time

Wir probieren erst den Zeitreiseansatz.

Ich habe eine Datei vorbereitet: contactmails_dump_with_time

    cat contactmails_dump_with_time

Wir schreiben die Datei nach Kafka wie zuvor.

    kafka-topics --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 4 \
      --config "cleanup.policy=compact" \
      --topic philip.contactmailsWithTime

    kafkacat -X topic.partitioner=murmur2_random -b localhost:9092 -t philip.contactmailsWithTime -P -K ";" -z lz4 -T -l contactmails_dump_with_time

Jetzt erstellen wir eine neue Tabelle.

    CREATE TABLE contact_mails_dump_oldtimestamp (key_customerid VARCHAR PRIMARY KEY, contactMailAddress VARCHAR, artificialMessageTime VARCHAR)
        WITH (kafka_topic='philip.contactmailsWithTime', value_format='JSON', timestamp='artificialMessageTime', timestamp_format='yyyy-MM-dd HH:mm:ss');

Wir benutzen die Parameter timestamp und timestamp_format um das Attribut artificalMessageTime als unsere ksqlDB rowtime zu benutzen.
Wir können das mit einer Query prüfen.
    
    SELECT rowtime, TIMESTAMPTOSTRING(rowtime, 'yyyy-MM-dd HH:mm:ss') AS rowtime_formatted, * FROM contact_mails_dump_oldtimestamp EMIT CHANGES LIMIT 10;

Die rowtime / rowtime_formatted Spalten decken sich mit der artificalMessageTime Spalte.
Wir haben die Nachrichten für ksqlDB quasi in die Vergangenheit geschrieben.

Jetzt können wir die Tabelle für unseren Join benutzen.

    SELECT customer_consents.key_customerid AS key_customerid,
           customer_consents.customerId,
           customer_consents.hashedCustomerId,
           contact_mails_dump_oldtimestamp.contactMailAddress AS contactMailAddress,
           customer_consents.consents['retargeter2'] AS consentToRetargeter2
       FROM customer_consents
       LEFT JOIN contact_mails_dump_oldtimestamp ON customer_consents.key_customerid = contact_mails_dump_oldtimestamp.key_customerid
       EMIT CHANGES LIMIT 10;

Wir sehen, dass die contactmailaddress Spalte jetzt vollständig ist.

Wir haben die alten Nachrichten jetzt erfolgreich angereichert und können den GDPR-konformen expliziten Opt-Out für Retargeter 2 machen.
Die Marketing-Abteilung kann damit Retargeter 2 wieder benutzen und alle sind glücklick.


## Table-Table Join

Zur Vollständigkeit können wir auch die zweite Option mit Table–Table-Join ausprobieren.

Wir erstellen aus den Consents eine Tabelle statt eines Streams.

    CREATE TABLE consents_table (key_customerid VARCHAR PRIMARY KEY)
    WITH (kafka_topic='customer_consents', value_format='avro');
    
Jetzt der Join: 

    SELECT consents_table.key_customerid AS key_customerid,
       consents_table.customerId,
       consents_table.hashedCustomerId,
       contact_mails_dump_correct.contactMailAddress,
       consents_table.consents['retargeter2'] AS consentToRetargeter2
    FROM consents_table
    FULL OUTER JOIN contact_mails_dump_correct ON consents_table.key_customerid = contact_mails_dump_correct.key_customerid
    EMIT CHANGES;

Ich benutze hier einen FULL OUTER JOIN damit wir auch die zusätzlichen Nachrichten sehen, wo eine Seite erst Null ist.
Wir sehen, dass am Ende für jeden Kunden eine vollständige Nachricht existiert.


# Backup



# Cleanup

    docker-compose down
    
    confluent local destroy
    
    clear

---
layout: post
title: Mariadb binlog parse failed in timestamp(6) field
tags:
- mariadb
---

Recently, our DBA coworker asked me to help diagnose a write communication error bug in the Debezium CDC platform. This bug only occurred in a specific instance, as all other instances with the same CDC configuration worked fine. They adjusted some timeout configurations, but the issue persisted after transitioning from the snapshot phase to the binlog phase.

In the MariaDB error log, the error message displayed was write communication error, while Debezium indicated a failure to process. Initially, I checked the logs sent by the data team, which were grepped. After examining the MariaDB slow log (set the slow log time to zero to display all queries), I discovered that Debezium initiated its jobs from a cleared GTID, resulting in a failure because the database couldn't locate the binlog file and position. The data team realized they had forgotten to clear the offset stored in the Kafka topic. After cleaning and retrying, the synchronization worked, resolving the issue caused by a simple human error.

The following day, the CDC job failed again with the same error. Could there have been a network problem? We used tcpdump to capture the database's network data and retried the task. However, it failed again after some time. Upon a basic pcap analysis, we observed that the CDC job sent a FIN tcp pack to the database while the database was still sending binlog data to Debezium.

Was this a Debezium bug? Considering that GTID support was relatively recent, I needed to examine the full process log to understand Debezium's actions. During this investigation, I encountered the grepped information. Upon further inspection, the Java stack trace, which contained crucial details like `java.io.EOFException: Failed to read the next byte from position 4863 and Caused by: com.github.shyiko.mysql.binlog.event.deserialization.EventDataDeserializationException: Failed to deserialize data of EventHeaderV4{timestamp=1711594580000, eventType=WRITE_ROWS, serverid=269410578, headerLength=19, dataLength=39, nextPosition=669961355, flags=0}.`

As per the MySQL/MariaDB documentation, a binlog comprises a header and a body/data part. The v4 binlog format includes a 19-byte header part. Since we encountered a failure at the nextPosition 669961355, the binlog should be at offset `669961355 - 19 - 39`. By using `show binlog events in 'xxx.binlog' from <offset> limit 1`, we can identify the unparsed binlog. Surprisingly, it was a straightforward `INSERT INTO` operation on a seemingly "normal" table containing only int/timestamp types. How could such a simple insert operation fail to be parsed by Debezium?

To troubleshoot, we decided to locate this record and table using the insert SQL (which was not the table need to CDC, as Debezium's whitelist only functions after the binlog has been parsed, it was been effected by non-related table). By examining the timestamp field, we confirmed that after the insertion into this table, the CDC job failed immediately. Consequently, we pinpointed this table's event as the cause of the CDC job failure.

Subsequently, I attempted to parse this binlog file using mysqlbinlog. Surprisingly, mysqlbinlog failed just like Debezium! This raised the question: why could MariaDB understand this binlog while mysqlbinlog couldn't?

Considering potential parsing issues with mysqlbinlog, I explored newer tools to parse the same binlog file, but they also failed. This led me to conclude that the binlog file must have been corrupted. It appeared that show binlog events did not share the same parsing logic as mysqlbinlog.

Unfortunately, the latest version of MariaDB employs mysqlbinlog 3.5, lacking the --debug option. Considering the option of compiling a custom mysqlbinlog tool from the MariaDB source code to add debugging capabilities, I found the process time-consuming and challenging due to the intricate C++ logic involved.

Recalling the existence of various third-party tools for parsing binlogs, I decided to explore those options. After a brief search, I came across a blog post on parsing binlogs with Python: [here](https://medium.com/@mahadir.ahmad/reading-mysql-binlog-with-python-c85a8ece3b28). This resource by Mahadir Ahmad provided the solution I was seeking. Thank you !

Upon downloading the sample code, making necessary modifications, installing dependencies, and running the script with the failed binlog position offset, the script failed successfully. The Python traceback revealed that the script struggled to parse the timestamp field. Despite the binlog event being a simple insertion of a single row, the script attempted to interpret it as two update events, leading to failure when reading the bytes as uint at the end. The script inadvertently left only 1 byte for a uint, causing the parsing error. By adjusting the parse logic to treat the field as a timestamp2 type, the script successfully parsed the data. Eureka!

> In MariaDB 10.1.2, a new temporal format was introduced from MySQL 5.6, altering the operations of TIME, DATETIME, and TIMESTAMP columns at lower levels. These changes allow temporal data types to include fractional parts and negative values. To revert to the older data type format, you can disable this feature using the mysql56_temporal_format system variable.

> For tables containing TIMESTAMP values created on an older version of MariaDB or when the mysql56_temporal_format system variable was disabled, the data continues to be stored in the legacy format. To update table columns from the old format to the new format, execute an ALTER TABLE... MODIFY COLUMN statement changing the column to the same data type. This adjustment may be necessary when exporting the table's tablespace for import onto a server with mysql56_temporal_format=ON set (see MDEV-15225).

> -- https://mariadb.com/kb/en/timestamp/#internal-format

This specific table was created in MariaDB 10.0 and later upgraded to 10.3. To resolve it, we followed the documentation's recommendation: `ALTER TABLE ... MODIFY COLUMN` to the same timestamp type, effectively resolving the issue.

Mariadb itself can replica data from binlog like this cause it fix this problem in 10.1. You can read [this](https://onlyacat.com/archives/1707037303391) blog to find more details about mysql/mariadb timestamp type evolved.

PS: The MariaDB upgrade process alone cannot rectify this error; modifying the column to the same timestamp type using DDL is imperative.

PPS: Exercise caution when using grep to read logs unless you fully understand the implications.

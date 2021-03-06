# The MailerQ database

MailerQ can use a database to store configuration data. This is either a 
Mysql, PostgreSql or Sqlite database. We recommend to set up such a 
database before you run MailerQ, but this is not strictly necessary.
The Sqlite database is by far the simplest to configure - all you need 
to do is ensure that the sqlite3 library is installed on your system.

The other two database systems, Mysql and PostgreSql, take a little
more time to set up, but are not too difficult to install too. You only 
need to create the database and put the login and password in the 
MailerQ configuration file. MailerQ does the rest and creates all
tables.

Using a database is *optional*. MailerQ can also run without having a 
database connection. However, since connecting to a database is simple 
(especially a Sqlite database), you better use it with a database.


## Database settings in the config file

Only one variable has to be set in the config file to connect to a
database: the `database` variable:

````
database:           sqlite://path/to/database/file
database:           mysql://user:password@hostname/databasename
database:           postgresql://user:password@hostname/databasename
````

Sqlite is the simplest database to set up, because you just specify the 
path to a file on the MailerQ server (this file does not even have to 
exist). However, Sqlite is not the most powerful or fastest system, so 
using a Mysql or PostgreSql is better.

MailerQ automatically creates or alters missing or incomplete tables. 
If you use a MySQL or PostgreSQL database, you should ensure that the 
database already exists, and that MailerQ has enough privileges to 
create and modify tables.


## Rebuilding the database

When MailerQ starts, it first connects to the database and checks whether
all tables are in a valid state. Tables that do not exist are created,
and tables that miss columns are automatically altered and the missing
columns are added. This automatic table-checking is done every time
that MailerQ starts up.

Normally, the only time when tables are created is the very first
time that you start MailerQ, and the only time when tables are altered 
is after upgrading to a new version that uses a slightly different 
database schema.

If you want to enforce that all tables in the database are dropped and
replaced by brand new empty tables, you can start MailerQ with the 
"--purge-tables" command line option:

````
mailerq --purge-tables
````

This option tells MailerQ not to check and repair tables, but to drop
them all and create new ones.



## Multiple MailerQ instances

It is possible to run multiple MailerQ instances that all connect to the
same database. The data in the database is periodically reloaded into main 
memory by all MailerQ instances, meaning that updates that are made via 
the management console of one MailerQ instance become (after a couple of 
minutes) available in the other instances as well.


## Reading and writing to the database directly

MailerQ has a powerful [web based MTA management console](management-console "Management console"). 
This console gives you access to the database driven MailerQ configuration forms. 
It is therefore in normal operations not at all necessary to run any 
queries on the database by yourself. But if you do like to access the data
you are free to do so. As mentioned above, MailerQ reloads data from the 
database every couple of minutes, so any changes you make will automatically 
come into effect without the need to restart MailerQ.


## Database structure

All created SQL tables are straight forward. Because MailerQ is compatible with 
many different database systems, it was not possible to use obscure or vendor 
specific SQL features. The tables use well known data types, and no foreign 
keys or constraints. Booleans are stored as integers and NULL and 0 values 
have the same semantics. The value -1 is used to set something to 'unlimited'.

The following tables are created

*   [ips](documentation/database-access#ips "ips")
*   [capacity](documentation/database-access#capacities "Capacity")
*   [capacity_per_ip](documentation/database-access#capacities "Capacity per IP")
*   [dkim_keys](documentation/database-access#dkim "DKIM keys")
*   [dkim_patterns](documentation/database-access#dkim "DKIM patterns")
*   [flood_responses](documentation/database-access#floodResponses "Flood Responses")


### The "ips" table

On startup, MailerQ detects all IP addresses that are linked to the server,
and stores these IP addresses in a database table. MailerQ does not use this
table for queries or for anything, the table is only created so that other
programs, like yours, have some way to find out which IP addresses are
available for sending out emails. If you run multiple instances of MailerQ,
and you want to find out to which RabbitMQ exchange you should publish
JSON formatted messages to send them via a certain IP, you just have to
let your script connect to the database and do the lookup

| Field name | Type    | Description                                        |
|------------|---------|----------------------------------------------------|
| ip         | varchar | IP address that is available to a MailerQ instance |
| vhost      | varchar | RabbitMQ vhost                                     |
| exchange   | varchar | RabbitMQ exchange                                  |
| outbox     | varchar | RabbitMQ outbox ( = routing key )                  |


### Tables "capacity" and "capacity_per_ip"

The 'capacity' and 'capacity_per_ip' tables hold the delivery capacities per target domain. If you want to install different limits for your local outgoing IP addresses you can use the 'capacity_per_ip' table. The 'capacity' table holds the records for system wide delivery capacities per domain - no matter what local IP is used.

When MailerQ sends out an email from one of its own IP addresses, it will first check if a record exists in the 'capacity_per_ip' table by running a select query with the domain name and local IP address in the where clause. If there was no matching record in this table, it will run a subsequent query on the 'capacity' table with only the domain name in the where clause.

| Field name             | Type            | Description                                                                                                                                                                                                              |
|------------------------|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| id                     | int PRIMARY_KEY | The unique ID for the record                                                                                                                                                                                             |
| domain                 | varchar UNIQUE  | A domain name for which capacity settings are defined                                                                                                                                                                    |
| version                | tinyint         | This column is only available in the 'capacity_per_ip' table and holds the IP address version. Supported values are 4 for IPv4 and 6 for IPv6                                                                            |
| localip                | binary(16)      | This column is only available in the 'capacity_per_ip' table and holds the local IP address for which the row hold delivery capacities. The IP address is stored in [binary format](database-access#ips "binary format") |
| domain_maxmessages     | int DEFAULT -1  | Max number of messages per minute that are sent to the domain                                                                                                                                                            |
| domain_maxconnects     | int DEFAULT -1  | Max number of newly created connections per minute for this domain                                                                                                                                                       |
| domain_maxconnections  | int DEFAULT -1  | Max number of connections that can be open at the same time to the domain                                                                                                                                                |
| ip_maxmessages         | int DEFAULT -1  | Max number of messages per minute that are sent to the IP addresses linked to the domain                                                                                                                                 |
| ip_maxconnects         | int DEFAULT -1  | Max number of newly created connections per minute to the IP addresses linked to the domain                                                                                                                              |
| ip_maxconnections      | int DEFAULT -1  | Max number of connections that can be open at the same time to an IP address the domain                                                                                                                                  |
| connection_maxmessages | int DEFAULT -1  | Max number of emails to send over an opened connection before it is closed                                                                                                                                               |
| connection_maxidletime | int DEFAULT -1  | Max number of miliseconds a connection is kept idle. MailerQ can keep connections open while it is throttling the deliveries. A connection is however never kept open for a longer period than this limit                |
| secure                 | int             | Boolean type field. Defines if secure connection should be attempted. If secure connections are not supported by the receiving server, regular connections are established                                               |

#### Examples

Limit the number of message deliveries to example.org to 1000 messages per minute and make sure that there will never be more than 2 connections open at the same time:

````
insert into capacity (domain, domain_maxmessages, domain_maxconnections) values ('example.org', 1000, 2)

````

Imagine that the example.org has four different IP addresses on which it receives email, and you do not want MailerQ to make more than five new connections per minute to each of those servers (thus 4 x 5 = 20 new connections in total):

````
update capacity set ip_maxconnects = 5 where domain = 'example.org'

````

### Tables 'deliveries_domain', 'deliveries_ip' and 'deliveries_total'

The three tables that all start with the name 'deliveries' are filled by MailerQ with counters with the number of deliveries, failures and retries. The data in the tables is used by the management console to display charts. In normal situations there is no good reason to insert or update any data in these tables, but you could run queries to remove rows from the table or to retrieve data if you want to display statistics somewhere else.

Although it would have been perfectly possible to integrate the three capacity tables into one single table, we use three different tables to speed up the queries used by the charts on the management console. The tables contain the total number of deliveries, the deliveries split up per target domain and the deliveries per ip:

*   Table 'deliveries_total': counters per minute for all deliveries
*   Table 'deliveries_domain': counters per minute split up per domain
*   Table 'deliveries_ip': counters per minute splut up per remote target ip

| Field name | Type            | Description                                                                                                                                                                                           |
|------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| id         | int PRIMARY_KEY | Usual auto_increment id field                                                                                                                                                                         |
| time       | datetime        | The minute for which the row holds the counters. Because all counters are kept per minute, only timestamps with rounded seconds (set to 00) are stored in this field                                  |
| version    | tinyint         | IP address version. 4 for IPv4 and 6 for IPv6                                                                                                                                                         |
| localip    | binary(16)      | IP address from which messages were sent. This column contains [binary data](database-access#ips "binary data")                                                                                       |
| domain     | varchar         | This column is only available in the table 'deliveries_domain' and holds the domain to which the messages were sent                                                                                   |
| remoteip   | binary(16)      | This column is only available in the table 'deliveries_ip' and holds the IP address to which the messages were sent. The IP address is stored in [binary format](database-access#ips "binary format") |
| success    | int             | Number of successful deliveries                                                                                                                                                                       |
| failures   | int             | Number of failed deliveries                                                                                                                                                                           |
| retries    | int             | Number of retries                                                                                                                                                                                     |

Data in this table is used to generate statistics. Since the data is never deleted it can grow significantly pretty quick. Data can be safely removed without any other problems than loss of statistic information.

### Table 'dkim_keys'

The last table to discuss is the 'dkim_keys' table. This holds all DKIM keys that are used by MailerQ for signing the outgoing messages. Each record holds a domain name, DKIM selector, the algorithm to use and of course the private key. MailerQ reloads all DKIM keys every 10 minutes so it is advised to wait at least 10 minutes before you send out an email if you have just updated the database. DKIM keys stored via the management console are immediately active.

| Field name | Type            | Description                                                             |
|------------|-----------------|-------------------------------------------------------------------------|
| id         | int PRIMARY_KEY | Usual auto_increment id field                                           |
| domain     | varchar UNIQUE  | The domain for which this record holds the DKIM key. Must be lower case |
| selector   | varchar         | The selector for DKIM signing                                           |
| privatekey | mediumtext      | Private key used for encryption. It should be kept secret at all times  |
| algorithm  | varchar         | An algorithm used for data encryption ( accepts SHA-1 or SHA-256 )      |

## Ip addresses in binary format

MailerQ stores IP addresses in binary format. An IPv6 address takes up 16 bytes, and an IPv4 address takes up 4 bytes and is stuffed with an additional 12 zero bytes to fill up the 16 bytes that are reserved for IP addresses.

In PHP you can use the functions inet_ntop() and inet_pton() to convert IP addresses to and from binary format:

````
$binary4 = inet_pton($ipv4).pack("LLL",0,0,0);
$binary6 = inet_pton($ipv6);
$ipv4 = inet_ntop($binary4);
$ipv6 = inet_ntop($binary6);

````
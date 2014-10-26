# SSDB TIPC

## Intro

A high performance network server build on top of rocksdb.
Rocksdb is a very reliable high performance database.

See if we can do this by changing ssdb.io

## Goals

* ultra high performance
* leverage TIPC capabilities 
  * avoid usage of ip addresses, easier to manage
  * tipc addresses work over multiple backplanes
  * fast & message based
* support for fixed size data structures
  * e.g. imaging always 16 kbyte blocks need to be stored in DB, the underlying rocksdb can be optimized for it
* easy configuration by means of TOML format config file (the std in golang community today)
* interopable between languages
* master/slave replication between 2 db's
* auto propagation from slave to master (so automatic clustering support)
* consistency is very high but not as high as full paxos clustered DB (see splitbrain problem below & replication)

## Why TIPC

* tipc is a great protocol for distributed computing. I thas support for many cluster features out of the box.
* ...

## Tasks

* investigate ssdb see how much needs to be changed
* spec toml format & command line usage
* create server in C
* create client in C
* create client in python (use cython to create a .so and bind to the c client)
* benchmarks over gbit & 10 gbit infrastructure, nicely document the outcome
* testsuite & benchmark tool

## Requirements

* hot reload of tipc config (e.g. reconfig for new tipc addresses to listen upon)
* hot reload of db config (e.g. new database)
* support for multiple rocksdb's (on different paths, to be specified in toml, each get a name)
* very easy authentication scheme (in toml format), defined per DB per function e.g. user1 can only access db:mydb for methods get & set
* use redis protocol to send/receive data

* Interface
  * ping
  * set(dbname,key,binarydata)
  * multiset(dbname,[keys,...],[binarydata,...])
  * get(dbname,key)
  * multiget(dbname,[keys])
  * delete(dbname,key)
  * multidelete(dbname,[keys])
  * list(dbname,prefix,usagestat=None,olderthan=None)
  	* see below for usage stats: when query for e.g. deleteRange(...,usagestat=0) this would mean all data written for certain prefix but not read yet
  	* olderthan to specify that data needs to have certain age
  * reloadconfig() #only for tipc & additional db's
  * flushReplication(dbname="")			#make sure data is replicated (batch closed & synced), if no db specified then all db's
  * flushDB(dbname="")					#make sure data is written to disk (ssd), if no db specified then all db's
  * deleteDB(dbname)
  * getLastTransactionId(dbname)		#tells last id of last cmd done

* transactionlog & transactionId (not sure I write this correct)
  * each cmd which modifies something like set/delete/... will get per db an incremental id which we call transactionid
  * for data not replicated yet we keep list of keys,transactionId's what needs to be replicated = replicationqueue
    * no need to do this on disk !!!
    * do this per DB
  * replication queue
  	* is in mem structure which holds all data to be replicated with corresponding transactionId's
  * when new data comes in we need to check if already in replicationqueue 
    * if in replication queue, do a flushReplication cmd first, then write
  * per db we can mark if we want to write a transaction log (std not enabled for performance reasons)
    * when transanctionlog (tlog) active
      * we write per DB a file with transactionId,key,data,metadata, ... in dense format (use fastest possible compression)
      * per configurable period a new file is written (e.g. every 5 min)
  * we remember per db what last properly written (todisk) transactionId is

* Metadata
  * add 10 bytes to each saved object  (is 10 bytes enough?)
  * first byte is used as follows
    * 0 = just written never read
    * 1 = 1 time read
    * 2-20 = x time read (stops at 20)  
  * then x bytes for epoch (last time data modified)
  * then y bytes for crc
  * then z bytes for transactionid
  * remaining bytes is for future usage e.g. type of info stored, ...
  * this would allow us to see which info has been read & written when, so caching policies can be created
  * Interface
    * setMeta(dbname,key,stats,mod-epoch,...)
    * getMeta(dbname,key)
      * returns crc,stat,mod-epoch
    * multiSetMeta(dbname,key,stats,modepoch)
    * multiGetMeta(dbname,key,stats,modepoch)
  
* replication
  * batched replication
  * transactionId are important !!!
  * order of cmds needs to be kept per DB when doing the replication !
  * slave is not there in master-slave situation
    * master: keep on writing to db, remember transaction id
    * start writing transaction log !!!
    * keep on trying to find slave, if slave find let slave to catch up (starting from tlog)

* Performance SUPER HIGH
  * write sync or batched to dbbackend (rocksdb)
  * write behaviour specified per db (toml)
  * when batched: max time behind can be set, max nr of objects behind can be set (sync cmd will make sure data is put on backend)
  * when replication 
    * do this in batches (happens per DB)
    * max time can be set, max nr of objects behind can be set
  * try to avoid memory copying (path from tipc message to db or batch for db nees to be as short as possible)
  * optimizations for fixed datablocksize (specified per db in toml)

* stats
  * every X seconds basic stats are send (using statsd format) to a destination
  * stats are: 
    * iops per period per db e.g. 1000 iops for db mydb during last 10 sec
    * nrcommands per type e.g. nr sets  always counted per period specified e.g. X could be 10 sec
    * size of each db in bytes and nr items
    * size of replicationqeue in nr of items per db
  * stats are counted per db
  * whens stats send this is done in 1 message with all info per db per type
  * configuration done per db in toml

* luasupport
  * like in redis/ledis, lua support

* autopropagation slave to master
  * when the c-client cannot connect to the master, it will try to connect to the slave (use ping command)
  * if the slave is there ask the slave to promote to master, the promoted master (was slave) will take over the tipc address of master
  * the 5 states of a db
    * master
    * old master (when slave gets promoted, the old master is the one who was master originally)
    * promoted master
    * slave
    * catching up slave
  * if a slave gets promoted to master it becomes a "promoted master"
    * it will start writing transaction log onto disk for all new data untill there is an active "slave" again which has all data
    * also modified data gets written to tlog (overwrite in set)
    * keep track of all keys modified per db since promotion (modtrackingdb)
  * split brain process
  	* this happens when master which should have been demoted still accepted data, we now have new data on old master & promoted master
  	* how can we detect
  	  * "old master" tries to replicate but "promoted slave" does not accept because slave got promoted, this will tell "old master"  oeps, I am no longer master
  	  * "old master" will try to recover the situation
  	* how to recover
  	  * "old master" stops accepting clients (this should have happened automatically because of tipc)
  	  * "old master" has the replication queue which will hold all new info
  	  * "old master" will talk to new master and check against modtrackingdb, 
  	    * if no conflicts data from replication queue gets written to new master
  	      * this means all data was succesfully merged in new master
  	    * if conflicts
  	       * send message (critical), db is possibly corrupt, not certain though
  	       * the data with newest moddate gets written to db on newmaster (also kept in tlog)
   	  * old master will after recovery action shut down
  * how to add slave
    * when slave is started of master it will check transactionID per db
      * when differences, the tlogs are used to catch up
    * if no tlogs on disk for starting transactionID's
      * e.g. slave completely empty & new and master only has partial tlog
      * ask promoted slave to send all info from db
      * promoted slave keeps on writing tlog (which happened from moment slave got promoted)
      * when main db's has been copied (so walked over all keys)
      * play the transaction logs to new slave (this will make sure that all changes since promotion gets played back in right order)
      * when tlogs played back: "promoted master" becomes "master" and "catching up slave" becomes "slave"
        * the tlogs get removed if server was not keeping tlogs




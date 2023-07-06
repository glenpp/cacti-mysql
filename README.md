# MySQL Performance Graphs on Cacti via SNMP

I regularly end up doing work with MySQL and as I regularly point out in other articles, tuning can give massive performance improvements if done right. The first thing is to when tuning is understand what is going on and where the bottlenecks are. Just running on a faster box is all well and good but it may not actually lead to in proportionate performance increases if the bottlenecks lie elsewhere.

These graphs are the product of some tuning work I've been doing over the past few months and are what I use for basic tuning and looking at what impact changes to queries have.

## Tuning wisely

There is an awful lot of stuff on the web about measuring miss ratios for things query caches and blindly increasing the size of them if the ratios are not favourable. There are circumstances when this approach is valid, but care is needed when tuning to dig further and understand why miss ratios may be high.

A typical example may be when the bulk of queries are simply not cacheable (eg. the same query doesn't repeat or always will return something different). What happens is that caches fill up with non-cacheable queries leaving less space for queries that are cacheable and at the same time introducing overheads of checking caches which are no use. Possible solutions would be to redesign queries (if possible) to be more cacheable, or just explicitly stop trying to cache a query that can be cached:

```sql
SELECT SQL_NO_CACHE * WHERE parameter = 'random stuff' AND anotherparameter = 'more random stuff';
```

That can give you the performance boosts you may be looking for where just throwing in some more cache memory may have negligible effect.

This is why hit/miss ratio graphs here have a label saying things like "lower is better... sometimes" as lower may only be better under specific circumstances. They can be very misleading if you don't understand why they are not showing optimum results.

One thing that is often worth checking is memory usage. In many cases caches may have high miss ratios and still have plenty of free space in the cache. This is a sure sign that heaping more cache space on is not going to be of any use at all.

To be useful, these graphs need to be used wisely and with understanding of what is actually happening.

## Getting it working

Like with other templates I create, this one relies on SNMP to shift the data. This is a convenient, simple one-stop way of shifting data (with encryption if needed) into Cacti or other monitoring systems and allows one method to be used for all your monitoring irrespective of being local, over the LAN or in a remote locations across the planet. There are already plenty of other templates that run as local scripts or via ssh.

This one is simply an extension script for snmpd that grabs (and caches) the data when called. Assuming you put this extension script in /etc/snmp and have a cache directory for my extensions of /var/local/snmp/cache (as described in previous articles), you can add the lines from snmpd.conf.cacti-mysql to your /etc/snmp/snmpd.conf file and then restart snmpd.

The first argument for **mysql-stats** is the cache file, and from there it simply picks up and outputs the variables specified on the remainder of the line.

I will add to this as myself (or perhaps others) need more things graphed.

In order to read all these variables from MySQL we need credentials. These are stored in /etc/snmpd/mysql-stats.conf by default and you can modify the script if you need to change that. Typically this would have something along the lines of:

```
data_source: DBI:mysql::localhost
username: monitor
password: secret
```

Make sure the permissions are set right on this file - we need the snmpd user to be able to read it by nobody else.

```
$ ls -l /etc/snmp/mysql-stats.conf
-rw-r----- 1 snmp snmp 83 Dec 27 21:05 /etc/snmp/mysql-stats.conf
```

Log into MySQL and execute the following (changing "secret" as appropriate) to give the script permission to monitor:

```sql
CREATE USER 'monitor'@'localhost' IDENTIFIED BY 'secret';
GRANT PROCESS ON *.* TO 'monitor'@'localhost';
```

More privileges may be needed in the longer term as more monitoring is added, but that's enough to monitor a basic single node setup and it's always best to give as little privileges as possible.

Install the Cacti template cacti_host_template_mysql.xml in the usual way and all going well graphs should start appearing. The usual diagnostics apply if data doesn't appear.

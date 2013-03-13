ats-plugin-cache-key-genid
==========================

Apache Traffic Server (ATS) plugin to modify the URL used as the cache key by adding a generation ID tag to the hostname.
This is useful when ATS is running in reverse proxy mode and proxies several (ie hundreds or thousands) of hosts.
Each host has a generation ID (genid) that's stored in a small embedded kytocabinet database.
Without this plugin, the CacheUrl is set to the requested URL.  
For example, if the requested url is  http://example.tld/foobar.css, then the CacheUrl is http://example.tld/foobar.css.
With this plugin, the CacheUrl is set to http://example.tld.#/foobar.css, where # is an integer representing example.tld's genid.

## Why use this plugin?

The simple answer:  fast clear cache operation.  

By incrementing a hosts genid, you instantly invalidate all of that host's files.  Not really invalidate, but make it impossible to find.

When an HTTP request comes into ATS, it takes the URL and generates the CacheUrl, then looks in the cache for the key matching md5(CacheUrl).
If CacheUrl changes, the md5 hash changes, and it won't find old copies.

This method is exceptionally faster than http://localhost/delete_regex=http://example\.tld/.*.  Using ATS's delex_regex method performs a full cache scan. 
It must look at every file in the cache, determine if it matches the regular expresssion, and then delete the file.  Using ats-plugin-cache-key-genid, no cache
scan is required.  You simply increment a single value in a kytocabinet database and done.

## to compile
```bash
/opt/ats/bin/tsxs -o cache-key-genid.so -c cache-key-genid.c
```

## to compile in libkyotocabinet
```bash
gcc -shared -Wl,-E -o cache-key-genid.so cache-key-genid.lo /opt/kyotocabinet/lib/libkyotocabinet.a
```

## to put into /opt/ats/libexec/trafficserver/
```bash
sudo /opt/ats/bin/tsxs -o cache-key-genid.so -i
```

## to create the kyotocabinet database
```bash
sudo /opt/kyotocabinet/bin/kcpolymgr create -otr /opt/ats/var/trafficserver/genid.kch
# replace "ats:disk" with the user:group that runs your ATS server
sudo chown ats:disk /opt/ats/var/trafficserver/genid.kch
```

## to add/modify a record in the kyotocabinet database
```bash
sudo /opt/kyotocabinet/bin/kcpolymgr set -onl /opt/ats/var/trafficserver/genid.kch example.tld 5
```

## to get a record from the kyotocabinet database
```bash
/opt/kyotocabinet/bin/kcpolymgr get -onl /opt/ats/var/trafficserver/genid.kch joecdntest10.com 2>/dev/null
```

## Set ATS debug to ON in records.config like this (do not do this in production):
```bash
CONFIG proxy.config.diags.debug.enabled INT 1
CONFIG proxy.config.diags.debug.tags STRING cache-key-genid
```


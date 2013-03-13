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

## The high level workflow of what this plugin does

The solution is to not actually delete the files, but instead increment a counter in an embedded database local to each ATS server. 
ats-plugin-cache-key-genid modifies the cache-key to include the host's genid.

* Intercept incoming http requests
* Hook just before the cache key is set
* The cache key is effectively md5(url) or md5(http://host/path). Change it to md5(http://host.genid/path)
** Take the url
** Find the host
** Lookup the host's generation ID in an embeded, super fast, super lightweight, mostly|all in memory, key/value pair kyotocabinet database
** Make a newurl string by injecting the host's genid just after the host in the original url. ie http://foo.com/style.css becomes http://foo.com.2/style.css
* Call TSCacheUrlSet with the newurl

How do you accomplish the genid increment?  Below we give you the kytocabinet command to do so.  Presumably, you have some for of user interface where they request 
a Clear Cache operation.  You must somehow relay this to your ATS server(s) and command then to increment that host's genid.  There are many designs available for this.
You either push or pull the command.  This might be accomplished by an event bus, for example.  Passing the request from a User admin UI to the ATS servers is beyond 
the scope of this write up.

## Requires

* Apache Traffic Server 3.0 (http://trafficserver.apache.org/)
* Kyto Cabinet (http://fallabs.com/kyotocabinet/)

## to compile
Assuming the ATS provided tsxs is in your PATH:
```bash
tsxs -o cache-key-genid.so -c cache-key-genid.c
```

## to compile in libkyotocabinet
```bash
gcc -shared -Wl,-E -o cache-key-genid.so cache-key-genid.lo /opt/kyotocabinet/lib/libkyotocabinet.a
```

## to put into libexec/trafficserver/
Assuming the ATS provided tsxs is in your PATH:
```bash
sudo tsxs -o cache-key-genid.so -i
```

## to create the kyotocabinet database
Assuming the Kytocabinet provided kcpolymgr is in your PATH:
```bash
sudo kcpolymgr create -otr /your/path/to/var/trafficserver/genid.kch
# replace "ats:disk" with the user:group that runs your ATS server
sudo chown ats:disk /your/path/to/var/trafficserver/genid.kch
```

## to add/modify a record in the kyotocabinet database
Assuming the Kytocabinet provided kcpolymgr is in your PATH:
```bash
sudo kcpolymgr set -onl /your/path/to/var/trafficserver/genid.kch example.tld 5
```

## to get a record from the kyotocabinet database
Assuming the Kytocabinet provided kcpolymgr is in your PATH:
```bash
kcpolymgr get -onl /your/path/to/var/trafficserver/genid.kch example.tld 2>/dev/null
```

## Set ATS debug to ON in records.config like this (do not do this in production):
```bash
CONFIG proxy.config.diags.debug.enabled INT 1
CONFIG proxy.config.diags.debug.tags STRING cache-key-genid
```

If you turn the debug on like this, then you can tail the traffic.out file and witness the discovery of the url and the CacheUrl transformation.
You would not want to run this in production, b/c you'd be writing too much to the log file, which would slow down ATS.
It's great for dev/test/debug, however, so you know it's working well.



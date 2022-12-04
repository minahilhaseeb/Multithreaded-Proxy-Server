# Multithreaded-Proxy-Server
Multithreaded HTTP Proxy and Cache!

#Goals
- To gain experience with networking, server sockets, client sockets, and the various system calls and library functions used to build networked programs.
- To understand the importance of network protocols and how programs communicating over the network must respect them if they’re all to coordinate properly.
- To fully understand HTTP, which is the primary protocol used by the World Wide Web.
- To gain some experience with HTTP over SSL, which is a secure form of HTTP request and response exchange.
# Implementing v1: Sequential HTTP proxy
- Your final product should be a multithreaded proxy and cache that blocks access to certain domains.  As with all nontrivial programs, we’re encouraging you to work through a series of milestones instead of implementing everything in one extended, daredevil coding binge.  You’ll want to read and reread Sections 11.5 and 11.6 of your B&O textbook (or rather, Sections 5 and 6 of the networking chapter from your reader) to ensure a basic understanding of HTTP.
- For the v1 milestone, you shouldn’t worry about threads or caching.  You should transform the initial code base into a sequential but otherwise legitimate proxy.  The code you’re starting with responds to *all* HTTP requests with a placeholder status line consisting of an *"HTTP/1.0"* version string, a status code of 200, and a curt *"OK"* reason message.  The response includes an equally curt payload announcing the client’s IP address. Once you’ve configured your browser so that all HTTP traffic is directed toward the relevant port of the *myth* machine you’re working on, go ahead and launch *proxy* and start visiting any and all websites.  
#### Your proxy should at this point intercept every HTTP request and respond with this:
- For the v1 milestone, you should upgrade the starter application to be a true proxy—an intermediary that ingests requests from the client, establishes connections to the origin servers, passes the requests on to the origin servers, waits for these origin servers to respond, and then passes their responses back to the clients.  Once the v1 checkpoint has been implemented, your *proxy* application should basically be a busybody that intercepts requests and responses and passes them on to the intended servers.  For the moment, you only need to provide support for the most common HTTP methods: *GET*, *POST*, and *HEAD*.  There are other methods (*PUT* and *DELETE* come to mind), but they're rarely used by traditional web servers, so your proxy doesn't need them.  You'll eventually add support for a fourth, *CONNECT* method to support HTTP over SSL, but for the moment just focus on *GET*, *POST*, and *HEAD*.
- Each intercepted request is passed along to the origin server pretty much as is, save for three small changes. We need to modify the intercepted request URL within the first line — the request line as it's called —  so that when you forward it as part of the request, it includes only the path and not the protocol or the host. The request line of the intercepted request should look something like this: GET http://web.stanford.edu/class/cs110/index.html HTTP/1.1
- Once you’ve reached your v1 milestone, you’ll be the proud owner of a fully functional *proxy*. You should visit every web site imaginable to ensure the round-trip transactions pass through your proxy without impacting the functionality of the site (caveat: see the note below on sites that require login or are served up via HTTPS).  Of course, you can expect any sites you can visit to load very slowly, since your proxy has this much parallelism: *zero*.
-  Your *proxy* application should, in theory, run until you explicitly quit by pressing ctrl-C.  A real proxy would be polite enough to wait until the current HTTP requests have been handled.  You don’t need to worry about this at all—just terminate the proxy application without being polite and without regard for any cleanup.

# Implementing v2: Sequential* *proxy* ** *with strikesets and* *caching*
- Once you’ve built CS110 proxy v1, you’ll have constructed your first networked application. In practice, proxies are used to either block access to certain websites, cache static resources that rarely change so they can be served up more quickly, or both.
- Incorporating support for each is relatively straightforward, provided you confine your changes to the *request-handler.h* and *.cc* files.  In particular, you should just add two *private* instance variables—one of type *StrikeSet*, and a second of type *HTTPCache* to *HTTPRequestHandler*.  Once you do that, you should do this: 
- Your *HTTPRequestHandler* class would normally forward all requests to the relevant original servers without hesitation.  But, if your request handler notices the origin server matches one of the regexes in the *StrikeSet*-managed set of verboten domains, you should immediately respond to the client with a status code of 403 and a payload of *"Forbidden Content"*.  Whenever you have to respond with your own HTML documents (as opposed to ones generated by the origin servers), just go with a protocol of *"HTTP/1.0"*.
- You should update the *HTTPRequestHandler* to check the cache to see if you’ve stored a copy of a previously generated response for the same request.  The *HTTPCache* class you’ve been given can be used to see if a valid cache entry exists, repackage a cache entry into *HTTPResponse* form, examine an origin-server-provided *HTTPResponse* to see if it’s cacheable, create new cache entries, and delete expired ones.  The current implementation of *HTTPCache* can be used as is—at least for this milestone.  It uses a combination of HTTP response hashing and timestamps to name the cache entries, and the naming schemes can be gleaned from a quick gander through the *cache.cc* file.
- Your to-do item for caching? Before passing the HTTP request on to the origin server, you should check to see if a valid cache entry exist.  If it does, just return a copy of it—verbatim!—without bothering to forward the HTTP request.  If it does *not*, then you should forward the request as you would have otherwise.  If the HTTP response identifies itself as cacheable, then you should cache a copy before propagating it along to the client.  
 
# Implementing v3: Concurrent proxy with strikesets and caching*
- You’ve implemented your *HTTPRequestHandler* class to proxy, block, and cache, but you have yet to work in any multithreading magic.  For precisely the same reasons threading worked out so well with your RSS News Feed Aggregator, threading will work miracles when implanted into your *proxy*.  Virtually all of the multithreading you add will be confined to the *scheduler.h* and *scheduler.cc* files.  These two files will ultimately define and implement an über-sophisticated *HTTPProxyScheduler* class, which is responsible for maintaining a list of socket/IP-address pairs to be handled in FIFO fashion by a limited number of threads.
- The initial version of *scheduler.h**/.cc* provides the lamest scheduler ever: It just passes the buck on to the *HTTPRequestHandler*, which proxies, blocks, and caches on the main thread.  Calling it a scheduler is an insult to all other schedulers, because it doesn’t really schedule anything at all.  It just passes each socket/IP-address pair on to its *HTTPRequestHandler* underling and blocks until the underling’s *serviceRequest* method sees the full HTTP transaction through to the last byte.
- Fortunately, you built a *ThreadPool* class for Assignment 5, which is exactly what you want here.  I’ve included the *thread-pool-release.h* file in the *assign6* repositories, and I’ve updated the *Makefile* to link against my own working solution of the *ThreadPool* class.  You should leverage a single *ThreadPool* with 64 worker threads, and use that to elevate your sequential proxy to a superhuman multithreaded one.  Given a properly working *ThreadPool*, going from sequential to concurrent is actually not very much work at all.  Don’t bother trying to integrate your own *ThreadPool*.  I promise you it’s easier to just use mine.
- Your *HTTPProxyScheduler* class should encapsulate just a single *HTTPRequestHandler*, which itself already encapsulates exactly one *StrikeSet* and one *HTTPCache*.  You should stick with just one scheduler, request handler, strikeset, and cache, but because you’re introducing parallelism, you’ll need to implant more synchronization directives to avoid any and all data races.  You shouldn’t need to protect the strikeset operations, since the strikeset, once constructed, never changes.  But you need to ensure concurrent changes to the cache don’t introduce any races that might threaten the integrity of the cached HTTP responses.  In particular, if your proxy gets two competing requests for the same exact resource and you don’t protect against race conditions, you may see problems.
####Here are some basic requirements:
- You must, of course, ensure there are no race conditions—specifically, that no two threads are trying to search for, access, create, or otherwise manipulate the same cache entry at any one moment.
- You can have at most one open connection for any given request.  If two threads are trying to fetch the same document (e.g. the HTTP requests are precisely the same), then one thread must go through the *entire examine-cache/fetch-if-not-present/add-cache-entry transaction before the second thread can even look at the cache to see if it’s there*.
- You shouldn’t lock down the entire cache with a single *mutex* for all requests, because that would introduce a huge bottleneck into the mix, allow at most one open network connection at a time, and render your multithreaded application to be more or less sequential.  
- Instead, your *HTTPCache* implementation should maintain an array of 997 *mutex*es, and before you do anything on behalf of a particular request, you should hash it and acquire the *mutex* at the index equal to the hashcode modulo 997.  You should be able to inspect the initial implementation of the *HTTPCache* and figure out how to surface a hash code and use that to decide which *mutex* guards any particular request. A specific *HTTPRequest* will always map to the same *mutex*, which guarantees safety; different *HTTPRequest*s may very, very occasionally map to the same *mutex*, but we’re willing to live with that, since it happens so infrequently. 
# Implementing v4: Concurrent proxy with HTTP/SSL passthrough*
- Finally, I want you to add support for HTTP over SSL (or HTTPS). Whenever a client wants to converse in HTTP over SSL (generally because the user attempts to load a URL prefixed by *https://*), it issues a *CONNECT* request instead of a more traditional traditional *GET*, *POST*, or *HEAD*  request.  *CONNECT* requests look like this:
CONNECT www.nytimes.com:443 HTTP/1.1
Host: www.nytimes.com
<zero or more additional key-value pairs>
<blank line>
- The browser issues this specific *CONNECT* request when it would like the proxy to establish a two-way connection directly to www.nytimes.com (http://www.nytimes.com/), port 443.  Once the proxy successfully establishes the forward connection to www.nytimes.com (http://www.nytimes.com/), port 443, the proxy should reply to the client with:
HTTP/1.1 200 OK
<blank line>
- By doing so, you proxy is telling the client that it’s established its own connection with the origin server and that all bytes sent by the client to the proxy will be forwarded verbatim to the origin server, and that all bytes returned from the origin server will be forwarded back to the client.  Once the client receives this 200/OK response from the proxy, it operates as if it is directly connected to the original server and sends encrypted HTTP requests with the expectation that it will receive encrypted HTTP responses. Of course, the two aren’t truly connected to one another, because the proxy is in the middle and works to present the illusion they are.  Your proxy shouldn’t try to ingest HTTP requests and responses using its *HTTPRequest* and *HTTPResponse* classes, because these classes can’t decrypt these bytes sequences.  Only the client and origin server endpoints can.
- To illustrate, assume that the proxy exchanges encrypted bytes with the client via descriptor 17 and with the origin server via descriptor 23. Any encrypted bytes read from the client through descriptor 17 are written verbatim to descriptor 23 so they’re forwarded to the origin server. Similarly, any encrypted bytes read from the origin server via descriptor 23 are written verbatim back to the client via descriptor 17. If the proxy reads in 76 bytes from descriptor 17, then it writes those same 76 bytes, unmodified, to file descriptor 23.  Similarly, if the proxy reads 911 bytes from descriptor 23, then it writes those same 911 bytes as is to file descriptor 17.  By doing so, the proxy acts as a simple, bidirectional passthrough until the client and origin server close the connections or until no bytes are sent in either direction for five seconds.
- Your implementation should layer *iosockstream*s over these descriptors as it does for previous milestones, but you’ll want to rely on the provided *ProxyWatchset* class to monitor each of the underlying descriptors for activity.  In particular, the *ProxyWatchset* exports a *wait* method that blocks until at least one byte of data can be read from one or more registered descriptors *or* until a timeout—by default, five seconds—expires.

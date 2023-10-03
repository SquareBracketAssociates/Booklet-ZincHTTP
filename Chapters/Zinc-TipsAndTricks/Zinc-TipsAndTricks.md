## Tips and Tricks

@cha:zinc-tips

This chapter will evolve over time and collect nice approaches to various practical problems you may encounter. 

### DNS over HTTPS \(DoH\)

Firefox switched over to using 'DNS over HTTPS \(DoH\)' by default \([https://blog.mozilla.org/netpolicy/2020/02/25/the-facts-mozillas-dns-over-https-doh/](https://blog.mozilla.org/netpolicy/2020/02/25/the-facts-mozillas-dns-over-https-doh/)\).

We can do this in Pharo as well, even out of the box \(minus the interpretation of the results, but still\).

### First, what is this? 

A good description can be found at [https://developers.cloudflare.com/1.1.1.1/dns-over-https/](https://developers.cloudflare.com/1.1.1.1/dns-over-https/).
Using the Cloudflare server, we can do the following in Pharo, using the JSON wire format.

```
ZnClient new
 url: 'https://cloudflare-dns.com/dns-query';
 accept: 'application/dns-json';
 queryAt: #name put: 'pharo.org';
 queryAt: #type put: 'A';
 contentReader: [ :entity | STONJSON fromString: entity contents ];
 get.
```


The actual address can be accessed inside the returned result.

```
SocketAddress fromDottedString: (((ZnClient new
 url: 'https://cloudflare-dns.com/dns-query';
 accept: 'application/dns-json';
 queryAt: #name put: 'pharo.org';
 queryAt: #type put: 'A';
 contentReader: [ :entity | STONJSON fromString: entity contents ];
 get) at: #Answer) first at: #data).
```


If you load the following code, [https://github.com/svenvc/NeoDNS](https://github.com/svenvc/NeoDNS).
It is just as easy to use the binary UDP wire format.

```
ZnClient new
 url: 'https://cloudflare-dns.com/dns-query';
 accept: 'application/dns-message';
 contentWriter: [ :message | 
   ZnEntity with: message asByteArray type: 'application/dns-message' ];
 contentReader: [ :entity | 
   DNSMessage readFrom: entity readStream ];
 contents: (DNSMessage addressByName: 'pharo.org');
 post.
```


Again, the actual address can be accessed inside the returned object.

```
(ZnClient new
  url: 'https://cloudflare-dns.com/dns-query';
  accept: 'application/dns-message';
  contentWriter: [ :message |
    ZnEntity with: message asByteArray type: 'application/dns-message' ];
  contentReader: [ :entity |
    DNSMessage readFrom: entity readStream ];
  contents: (DNSMessage addressByName: 'pharo.org');
  post) answers first address.
```


Incidentally, a more robust answer can be got as follows:

```
NeoSimplifiedDNSClient default addressForName: 'pharo.org'.
```



### Serving static files


Is it possible a Zinc server returns static files within a specific url path \(like the ZnStaticFileServerDelegate\) and also returns other logics as shown with `map:#otherPath to: MyWebapp new` ?

The `ZnDefaultServerDelegate` can already do a lot, including the request. Consider part of the startup file of \*[http://zn.stfx.eu](http://zn.stfx.eu)* which serves its own website, together with all the default builtin responses.

So the following are all possible:
- http://zn.stfx.eu
- http://zn.stfx.eu/zn/index.html
- http://zn.stfx.eu/xn/small.html
- http://zn.stfx.eu/welcome
- http://zn.stfx.eu/dw-bench
- http://zn.stfx.eu/help


\(the last one lists all configured prefixes\).

```
(ZnServer defaultOn: 8180)
	logToTranscript;
	logLevel: 1;
	start.
```


```
(staticFileServerDelegate := ZnStaticFileServerDelegate new)
	prefixFromString: 'zn'; 
	directory: '/home/stfx/zn' asFileReference.

ZnServer default delegate prefixMap 
	at: 'zn' 
	put: [ :request | staticFileServerDelegate handleRequest: request ];
	at: 'redirect-to-zn'
	put: [ :request | ZnResponse redirect: '/zn/index.html' ];
	at: '/'
	put: 'redirect-to-zn'.
```



There is `ZnPrefixMappingDelegate` you can use.

```
server
   delegate: (ZnPrefixMappingDelegate
      map: 'static' to: staticFileDelegate; 
      map: 'app' to: myApp)
```


There is also `ZnStaticFileDecoratorDelegate` if you want to mimick another typical setup where all urls that resolve to a file get served from disk and any other will be forwarded to you app.

```
server
   delegate: (ZnStaticFileDecoratorDelegate
      decorate: myApp
      servingFilesFrom: 'static/')
```


Problems on Android
-------------------

Android Browser doesn't support WebSockets,
so this connection method is ruled out by definition.

It also seems that the browser might be faulty and doesn't properly
send POST requests.
All content of the request is sent, but somehow (too long delay?) gets
fragmented in a way that confuses the HTTP server which accepts
only the header part.

The test uses a PhoneGap packaged SimpleClient run on Android.
PhoneGap embeds the mobile application inside a WebView whose
functionality is backed by the same engine as Android Browser.
More testing exposed that the behaviour is the same when using
SimpleClient as a web application straight from the browser.

For an example see below.
`192.168.1.17:5280` is [`ejabberd`](https://github.com/esl/ejabberd)
running `mod_bosh` which uses Cowboy.
`192.168.1.25:33082` is PhoneGap packaged SimpleClient on Android.

    $ sudo ngrep -W byline -d wlan0 port 5280 
    interface: wlan0 (192.168.1.0/255.255.255.0)
    filter: (ip or ip6) and ( port 5280 )

    ####
    T 192.168.1.25:33082 -> 192.168.1.17:5280 [AP]          (block 1)
    POST /http-bind/ HTTP/1.1.
    Host: 192.168.1.17:5280.
    Accept-Encoding: gzip.
    Accept-Language: pl-PL, en-US.
    Accept-Charset: utf-8, iso-8859-1, utf-16, *;q=0.7.
    Keep-Alive: 115.
    User-Agent: Mozilla/5.0 (Linux; U; Android 2.3.3; pl-pl;
                HTC Vision Build/GRI40) AppleWebKit/533.1
                (KHTML, like Gecko) Version/4.0 Mobile Safari/533.1.
    Origin: file://.
    Connection: keep-alive.
    Accept: text/xml, text/html, application/xhtml+xml, image/png,
            text/plain, */*;q=0.8.
    Content-Type: text/xml; charset=utf-8.
    Content-Length: 269.
    .

    ##
    T 192.168.1.25:33082 -> 192.168.1.17:5280 [AP]          (block 2)
    <body content='text/xml; charset=utf-8' hold='1'
          xmlns='http://jabber.org/protocol/httpbind'
          to='localhost' wait='300' rid='694165' secure='false'
          newkey='2d5903f1a6a1dd0313f3dda832cc9f2068cd5ffb'
          xml:lang='en' ver='1.6' xmlns:xmpp='urn:xmpp:xbosh'
          xmpp:version='1.0'/>
    ##
    T 192.168.1.17:5280 -> 192.168.1.25:33082 [AP]          (block 3)
    HTTP/1.1 500 Internal Server Error.
    connection: keep-alive.
    server: Cowboy.
    date: Sat, 16 Mar 2013 21:11:05 GMT.
    content-length: 0.
    .

    ###

Block 1 and 2 from the above listing are in fact the same POST request.
However, the HTTP server, as well as `ngrep` as can be seen above,
treat these two data payloads as separate requests.
The server notices `Content-Length: 269` and assumes the request body is
present, but on trying to access that body it crashes.

It is not granted that a POST request will get fragmented, but it may and
that does happen often.
Even clearer example is [this log](https://gist.github.com/lavrin/5195628)
of `ngrep` output where the second part of POST arrived
_after_ the server had already crashed.

Issue observed on:

- Android 2.3 and 4.1
- [ejabberd commit ca7c8da64a9623cbf181bf5d32ef3ddbe439e632](https://github.com/esl/ejabberd/tree/ca7c8da64a9623cbf181bf5d32ef3ddbe439e632)
- [Cowboy commit cc507789bf1d9ea27e9fb53d06dc8a464672b7da](https://github.com/extend/cowboy/tree/cc507789bf1d9ea27e9fb53d06dc8a464672b7da)
- [SimpleClient commit 4253fadd5d805a20928855f6b8ab8e2f0f7a679f](https://github.com/lavrin/jsjac-simpleclient-pg/tree/4253fadd5d805a20928855f6b8ab8e2f0f7a679f)

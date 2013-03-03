---
layout: post
title: "Herding HTTP Requests, Or Why Your Keep-Alive Connection May Already Be Dead"
date: 2013-03-02 21:05
comments: true
categories: android
---
One of the more expensive things you can do on Android, or mobile in general, is
sending and receiving data over a mobile network connection. Holding a dedicated channel 
in particular causes high battery consumption, 
as much as [a hundred times more compared to being idle](http://en.wikipedia.org/wiki/Radio_Resource_Control#cite_ref-energyconsumption_2-0). Moreover, there is overhead
on various levels when establishing a dedicated connection: First, [it has has been shown](http://developer.sonymobile.com/2010/08/23/android-tutorial-reducing-power-consumption-of-connected-apps/)
that poor timing w.r.t. sending network requests can leave the connection in states
with high power consumption before going back to idle, even when it's not actively
used. Second, establishing an HTTP connection, especially when using secure sockets (HTTPS),
will result in a handshake being performed between the mobile client and the server,
even when multiple requests are sent in quick succession. If no proper precaution
is taken, these handshakes will occur N times when N requests are being sent, and your app
will incur the associated extra cost of network traffic and battery consumption.

To mitigate these issues, it is generally advisable to batch HTTP requests whenever
possible and send them over a [persistent HTTP connection](http://en.wikipedia.org/wiki/HTTP_persistent_connection).
This can be implemented using request queues (persisted or in memory)
which are emptied at a given time using HTTP Keep-Alive. At SoundCloud we have recently
implemented a tracking component which batches up a number of requests and flushes it
in certain intervals to minimize the aforementioned overhead. However, we stumbled
a few times when trying to make persistent connections work with HttpURLConnection, so make
sure to double check your own implementation against the next few paragraphs or
you may unwittingly send a large number of requests over separate connections,
even when Keep-Alive is enabled. So let me break it to you:

# HTTP Keep-Alive on Android does NOT just work
Fortunately for us, Android sets the Keep-Alive header by default, which a quick glance
at the header fields of a newly opened HttpURLConnection shows. However, we were
surprised to see that Android would still happily open new connections when
using code like the snippet below:

{% codeblock lang:java %}
Collection<String> requestUrls = ...;
HttpURLConnection connection = null;

for (String url : requestUrls) {
    try {
        connection = (HttpURLConnection) new URL(url).openConnection();

        connection.setConnectTimeout(CONNECT_TIMEOUT);
        connection.setReadTimeout(READ_TIMEOUT);
        connection.connect();

        final int response = connection.getResponseCode();

        if (response == 200) {
          // handle success
        } else {
          // handle error code
        }
    } catch (IOException e) {
        // ...
    }
}

if (connection != null) {
    connection.disconnect();
}
{% endcodeblock %}

Sniffing these requests with Wireshark showed that they were sent using separate
TCP streams coming from different client ports (indicating separate connections.)
Looking at the code, we're not closing the connection until we've sent the last
request, and we made sure Keep-Alive is actually set for every request, so what's the problem here?

{% img /images/posts/keep_alive_not_working.png %}

# Finding the culprit

Revisiting [Persistent Connections](http://docs.oracle.com/javase/6/docs/technotes/guides/net/http-keepalive.html)
we found this curious paragraph:

{% blockquote %}
When the application finishes reading the response body or when the application calls close() on the InputStream returned by URLConnection.getInputStream(), the JDK's HTTP protocol handler will try to clean up the connection and if successful, put the connection into a connection cache for reuse by future HTTP requests.
{% endblockquote %}

Apparently, in order to actually being able to reuse a connection, the implementation must
know where one HTTP request on the same TCP connection ends and the next one starts.
In our case, we weren't interested in the response payload since these were fire-and-forget
style requests, so we simply added a

```connection.getInputStream().close();```

as the documentation suggests. This, still, did not work for us. Again Android would not reuse
TCP sockets but open a new one for every request; we haven't found out why this is happening
so please leave me a comment if you do. From here on you're left with two options:
You either fully consume the response payload, which you may want to do anyway depending
on your mileage. This would mean reading bytes from the input stream until hitting the EOS byte.
Alternatively, if you're not interested in the server's response, you should do as
the document above suggests and send an HTTP HEAD instead:

{% codeblock lang:java %}
...
for (String url : requestUrls) {
    try {
        connection = (HttpURLConnection) new URL(url).openConnection();

        connection.setRequestMethod("HEAD");

        // as above
    } catch (IOException e) {
        // ...
    }
}
...
{% endcodeblock %}

Note that in this case, neither closing the input stream nor consuming it is required
since there will be no response payload to read from.
Returning to Wireshark, here's what the communication with the server looks like now:

{% img /images/posts/tcp_stream_keep_alive.png %}

This is what we wanted to begin with: Client and server establish a TCP stream, then sending all
six GET requests over the same connection. After doing our work, either the client or the
server will terminate the connection by starting a FIN exchange. Here it is important to
understand that calling ```connection.disconnent()``` does not guarantee triggering FIN.
From the documentation of [HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html):

{% blockquote %}
Once the response body has been read, the HttpURLConnection should be closed by calling disconnect(). 
Disconnecting releases the resources held by a connection so they may be closed or reused.
{% endblockquote %}

The emphasis here is on *may*: as you can see from the previous screenshot, it was the server
that decided to terminate the connection at some point, and not our call to ```disconnect```,
since the first FIN is sent by the destination not the source.

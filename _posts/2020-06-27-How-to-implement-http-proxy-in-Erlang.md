---
layout: post
title:  "How to implement http proxy in Erlang"
date:   2020-06-01 15:14:22 +0300
---

There are two types of proxies

- reverse proxies (hide server from the client)
- forward proxies (hide client from the server)

Reverse proxies could be easily implemented with [vegur](https://github.com/heroku/vegur)/[cowboyku](https://github.com/heroku/cowboyku) from heroku. cowboyku is a heroku fork of cowboy webserver created exactly for building reverse proxies in erlang. It is used in heroku for [http routing](https://devcenter.heroku.com/articles/http-routing) feature.

For forward proxies it's a little bit tricky. None of popular erlang webservers does not support `CONNECT` method. Let's eloborate how to do this from scratch.

First of all we need to start to listen a tcp port

{% highlight erlang %}
{ok, ListenSocket} = gen_tcp:listen(3128, [
    binary,
    {packet, http},
    {active, false},
    {reuseaddr, true}
])
{% endhighlight %}

Couple words about [options](http://erlang.org/doc/man/gen_tcp.html#type-listen_option):
  - `{packet, http}` will allow us not to worry about parsing http and [rely on Erlang stdlib](http://erlang.org/doc/man/erlang.html#decode_packet-3).
  - `{active, false}` Simple rule: don't use `{active, true}`. It's needed to avoid message-box overload.

Once we have a listen socket, we can accept connections from a client

{% highlight erlang %}
{ok, ClientSocket} = gen_tcp:accept(ListenSocket).
{% endhighlight %}

Accept is a blocking operation. It will wait for a client as long as needed.

Once we have a client socket, we can begin an exchange of bytes via tcp with the client. For this we will need some data structure to accamulate data related to request. The most natural for erlang is to use record for this purpose

{% highlight erlang %}
-record(request, {
    uri,
    headers = []
}).
{% endhighlight %}

Besides uri we also want to preserve headers. If we will want to introduce authentication to our proxy, we will need those header. Becasue credentials are passed in `Proxy-Authentication` header.


Thanks to `{packet, http}` we don't have to parse any http-traffic and we are able to receive ready http messages

{% highlight erlang %}{% raw %}
recv_loop(ClientSocket, #request{headers = Headers} = Request) ->
    case gen_tcp:recv(ClientSocket, 0) of
        {ok, {http_request, "CONNECT", Uri, _Version}} ->
            recv_loop(ClientSocket, Request#request{uri = Uri});
        {ok, {http_header, _, Name, _, Value}} ->
            recv_loop(ClientSocket, Request#request{headers = [{Name, Value} | Headers]});
        {ok, http_eoh} ->
            {ok, Request#request{headers = lists:reverse(Headers)}}
    end.
{% endraw %}{% endhighlight %}

We deliberately do not match other methods to keep only essence of the process. We are only interested in `CONNECT` method because we are building proxy now, not regular web-server.

How does `CONNECT` method work? `CONNECT` method is an appeal to server to establish tunnel connection with a target host and resend all the data it receives.

{% highlight erlang %}
{scheme, Host, Port} = Request#request.uri,
{ok, ServerSocket} = gen_tcp:connect(Host, list_to_integer(Port), [{packet, raw}, {active, false}].
{% endhighlight %}

Now we are using `{packet, raw}` because it's not really our business what traffic will go to the target host. Most likely it will be an ecrypted https-traffic. Thus it's just bytes for us.

After tunnel connection is established we should notify client that CONNECT requst is over an it can start sending the main request. Also we need to change socket mode from `http` to `raw` for client socket as well

{% highlight erlang %}
ok = inet:setopts(ClientSocket, [{packet, raw}]),
ok = gen_tcp:send(ClientSocket, <<"HTTP/1.1 200 OK\r\n\r\n">>).
{% endhighlight %}

Well, Everything is ready for proxy. But there is a little problem here. Due to we don't know the nature of traffic and protocol between client and server, we cannot know when to stop receiving data from the client and start sending to the server and vice versa. But client and server knows. So we have to do client-server transfer and server-client transfer and the same time until the sockets will be closed.

{% highlight erlang %}{% raw %}
proxy(ClientSocket, ServerSocket) ->
    spawn_link(fun() -> transfer(ClientSocket, ServerSocket) end),
    spawn_link(fun() -> transfer(ServerSocket, ClientSocket) end).

transfer(From, To) ->
    case gen_tcp:recv(From, 0) of
        {ok, Data} ->
            ok = gen_tcp:send(To, Data),
            transfer(From, To);
        {error, _Error} ->
            ok
    end.
{% endraw %}{% endhighlight %}

That is all. Erlang has everything to build forward proxies in a very straightforward and easy way. You can see a full implementation [here](https://github.com/tank-bohr/reimagined-dollop).


### Related links:

- [CONNECT-method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/CONNECT)
- [Proxy-Authenticate header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Proxy-Authenticate)
- [HTTP(S) proxy in golang in less than 100 lines of code](https://medium.com/@mlowicki/http-s-proxy-in-golang-in-less-than-100-lines-of-code-6a51c2f2c38c)

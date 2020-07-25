---
layout: post
title:  "How reactor works"
date:   2020-07-25 15:00:00 +0300
---

This post is a continuation of the [previous article]({{"2020/07/18/How-prefork-servers-work.html"|absolute_url}}) on how the prefork works.

Prefork servers have a particular feature. They handle one request at a time. There is a reason for that, a fork is an expensive operation, mostly in a sense of memory and in a sense of performance as well. That's why we had to create a **limited** amount of processes **preliminary**. Creating a fork during request processing is not an option, because it has a significant performance impact.
It leads to the request-response model when you have to establish a connection on every request. And it's not TCP-friendly, because it has some overhead:
- you have to do [TCP-handshake](https://en.wikipedia.org/wiki/Handshaking#TCP_three-way_handshake) every time
- you are loosing your increased [sliding window](https://en.wikipedia.org/wiki/Sliding_window_protocol)

That's why HTTP/1.1 keep-alive and http/2 had been invented. To utilize TCP more efficiently.
So now we have a contradiction: TCP works better with persistent connection but prefork has to work with short-living requests. Thus we should come up with something more suitable than prefork. Let's begin from the socket again.

{% highlight ruby %}
require 'socket'

socket = Socket.new(:INET, :STREAM)
socket.setsockopt(:SOCKET, :REUSEADDR, true)
sockaddr = Socket.pack_sockaddr_in(2200, '127.0.0.1')
socket.bind(sockaddr)
socket.listen(_backlog = 3)
{% endhighlight %}


But now we don't want to do a fork, because we know that it's very expensive. It means that we will have a lot of client sockets in one process. It means that [select](https://man7.org/linux/man-pages/man2/select.2.html) is our choice

{% highlight ruby %}
clients = []

while true do
  ready_to_read, _ready_to_write, _errors = select([socket] + clients)
  ready_to_read.each do |io|
    if io == socket
      client_socket, _client_addrinfo = socket.accept
      clients << client_socket
    else
      input = io.gets
      io.puts "#{input.chomp.reverse}"
    end
  end
end
{% endhighlight %}


To control the main loop we can use signal+pipe trick

{% highlight ruby %}
run = true
clients = []
to_read, to_write = IO.pipe
Signal.trap('TERM') { to_write.puts 'TERM' }

while run do
  ready_to_read, _ready_to_write, _errors = select([to_read, socket] + clients)
  ready_to_read.each do |io|
    if io == to_read
      signal = to_read.gets.chomp
      run = false if signal == 'TERM'
    elsif io == socket
      client_socket, _client_addrinfo = socket.accept
      clients << client_socket
    else
      input = io.gets
      io.puts "#{input.chomp.reverse}"
    end
  end
end
{% endhighlight %}

Now we can shut down the process gracefully by sending the `TERM` signal to it.

`select` system call is a part of a POSIX standard, so it will work almost everywhere, even on Windows platform. But it has some performance drawbacks. That's why every platform has it's own subsystem for the `select` call replacement. Linux has [epoll](https://en.wikipedia.org/wiki/Epoll) and BSD-based systems have [kqueue](https://en.wikipedia.org/wiki/Kqueue).

The last thing we need to add is a timeout, to be able to implement [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout)-alike callbacks. The last parameter in `select` is for timeout. It's exactly what we need. The following code will tick every second.

{% highlight ruby %}
while run do
  ready_to_read, _ready_to_write, _errors = select([to_read, socket] + clients, [], [], 1)
  if ready_to_read.nil?
    puts 'tick'
    next
  end
  ready_to_read.each do |io|
    if io == to_read
      signal = to_read.gets.chomp
      run = false if signal == 'TERM'
    elsif io == socket
      client_socket, _client_addrinfo = socket.accept
      clients << client_socket
    else
      input = io.gets
      io.puts "#{input.chomp.reverse}"
    end
  end
end
{% endhighlight %}

So that's it. We've implemented an essence of [event-machine](https://github.com/eventmachine/eventmachine) in 30 lines of code. Also we showed how the reactor pattern works and explain what advantages does it have against prefork model. It's also possible to take the best of two worlds. We can have mupltiple processes with multiple connections per process. In prefork implementation we've already used `select` for receiving events from sockets, all we need is just not to close a socket but keep it and add it to `select`.

### Related links:

- [The first part]({{"2020/07/18/How-prefork-servers-work.html"|absolute_url}})
- [Reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern)
- [EventMachine: fast, simple event-processing library for Ruby programs](https://github.com/eventmachine/eventmachine)
- [Final implementation](https://github.com/tank-bohr/studious-octo-funicular)


Let's imagine we want to create a new shiny service. Let's say "Reverse string As A Service" or RAAS. And for some reason, we want to do it from scratch.
First of all, we need a socket to receive connections.

{% highlight ruby %}
require 'socket'
socket = Socket.new(:INET, :STREAM)
{% endhighlight %}

Ruby standard library has very thin [wrapper](https://ruby-doc.org/stdlib-2.7.1/libdoc/socket/rdoc/index.html) around [socket](https://man7.org/linux/man-pages/man2/socket.2.html) system call. Thus it allows us to do very low-level stuff in a very high-level programming language. Which is cool!

The next step is to make a [listen](https://man7.org/linux/man-pages/man2/listen.2.html) syscall.

{% highlight ruby %}
sockaddr = Socket.pack_sockaddr_in(2200, '127.0.0.1')
socket.bind(sockaddr)
socket.listen(_backlog = 5)
{% endhighlight %}

The only parameter in listen function is an integer so-called backlog. Backlog is a very interesting parameter. It shows how long a pending connection queue would be. If two connections come simultaneously, one listening socket cannot process both of them. Thus one of them waits in the queue. So the length of this queue is a backlog. If your system can process connections very fast, you could keep a big backlog. Otherwise, you should set it not too big. Because most likely you want to refuse connection as soon as possible if your system runs out of resources. Especially if you have some balancer, like haproxy or nginx with upstream module enabled, in front of your app. In this case, your client could even don't notice the `connection refused` error because the balancer will route a request to the next application instance.

Ok, now our service is listening and ready to receive connections. What happens when a connection comes? We need to [accept](https://man7.org/linux/man-pages/man2/accept.2.html) it.

{% highlight ruby %}
client_socket, client_addrinfo = socket.accept
{% endhighlight %}

[Socket#accept](https://ruby-doc.org/stdlib-2.7.1/libdoc/socket/rdoc/Socket.html#method-i-accept) returns two data structures as a result:

- `client_socket` is a socket per se. Having this socket we can send/receive data to/from client
- `client_addrinfo` is an [Addrinfo](https://ruby-doc.org/stdlib-2.7.1/libdoc/socket/rdoc/Addrinfo.html) struct. From here, for example, you can get an ip address of the client. Could be useful.

Now is the time to do what our service was devised for. Reverse a string!

{% highlight ruby %}
input = client_socket.gets
client_socket.puts "#{input.chomp.reverse}"
{% endhighlight %}

Cool! It works. But there is a little problem our service finishes all the work after processing one request. One request is not enough. We want to be able to process thousands of them. We are expecting a high load.

The first thing we could do is a loop our business logic.

{% highlight ruby %}
while input = client_socket.gets do
  client_socket.puts "#{input.chomp.reverse}"
end
{% endhighlight %}

But then one client could use our service forever and it won't be available for other clients. So let's limit one client with only one reverse request.

{% highlight ruby %}
while (client_socket, client_addrinfo = socket.accept) do
  input = client_socket.gets
  client_socket.puts "#{input.chomp.reverse}"
  client_socket.close
end
{% endhighlight %}

Now if you need another string to reverse you have to do it within a new connection. It's ok. For example, http works the same way.

But we still cannot handle 2 simultaneous requests. It's time to do some **prefork**!

{% highlight ruby %}
# ...
socket.listen(_backlog = 5)

fork # <--- Not a 🥄

while (client_socket, client_addrinfo = socket.accept) do
  input = client_socket.gets
  client_socket.puts "#{input.chomp.reverse}"
  client_socket.close
end
{% endhighlight %}

Here is where unix magic begins. After fork invocation, there are two identical processes. They share one listen socket and accept connections from it in round-robin fashion. The first connection will go to the first process, the second connection to the second process, the third connection to the first process again, and so on. It's a built-in balancing mechanism. We don't have to do any rotation logic. Unix will do everything for us. It's the simplest possible prefork server.

Now we can process 2 simultaneous requests. But what if we want to process 3 simultaneous requests 🤔. It's a little bit tricky. We have to have 2 kinds of processes. The main one which we will fork other processes from. And several workers will process requests.

{% highlight ruby %}
#...
socket.listen(_backlog = 5)

workers_count = 3
workers_count.times do
  if fork.nil?
    # it's a child. it's a worker
    while (client_socket, client_addrinfo = socket.accept) do
      input = client_socket.gets
      client_socket.puts "#{input.chomp.reverse}"
      client_socket.close
    end
    exit
  else
    # it's a parent. it's a master
  end
end
{% endhighlight %}

But it's not enough. After the master process finished it's forking work it will finish its execution due to it has no business to do. And we will have 3 orphan worker processes. It's not good. In the parent process, we should wait for all children to finish their work. But we don't have any means to terminate the worker process yet. We need somehow to say workers they should terminate.

There are not so many ways for 2 processes to interact with each other

- some shared resource (file, database, message queue)
- network socket
- unix signals
- pipes

For our purposes pipe looks like the best choice. Let's try it out

{% highlight ruby %}
workers_count = 3
worker_pipes = []
workers_count.times do |number|
  to_read, to_write = IO.pipe
  if fork.nil?
    # it's a child. it's a worker
    $0 = "[RAAS] worker #{number}"
    run = true
    while run do
      ready, _, _ = select([to_read, socket])
      ready.each do |io|
        if io == to_read
          run = false
        elsif io == socket
          client_socket, client_addrinfo = socket.accept
          input = client_socket.gets
          client_socket.puts "#{input.chomp.reverse}"
          client_socket.close
        end
      end
    end
    exit
  else
    # it's a parent. it's a master
    worker_pipes << to_write
  end
end

# Master process context
$0 = "[RAAS] master"

puts 'Press Enter'
gets

worker_pipes.each(&:puts)
puts 'Done'
{% endhighlight %}

As you can see now we have two io objects to receive data from. So we have to use [select](https://man7.org/linux/man-pages/man2/select.2.html) in our loop. `$0` is a process name that you will see in the `ps` output. This will allow us to distinguish worker processes from the master one. There is a function in ruby with much better semantics [setproctitle](https://ruby-doc.org/core-2.7.1/Process.html#method-c-setproctitle). Also this awkward pattern

{% highlight ruby %}
if fork.nil?
  # ...
  exit
end
{% endhighlight %}

can be replaced with the neat ruby syntax: [fork](https://ruby-doc.org/core-2.7.1/Process.html#method-c-fork) with block.

As one can see it' the most direct way to implement our service. Thus it's awful in a sense of style and best engineering practices. A lot of things require refactoring. So it can be a good refactoring kata.

The last thing I wanted to mention is that we now have an almost ready [unicorn webserver](http://unicorn.bogomips.org/) in less than 50 loc. The only thing missing is the http parser. For http-parser unicorn and puma use [ragel](http://www.colm.net/open-source/ragel/) to generate c-code. Http-parser should be blazing fast.

### Related links:

- [Unicorn Unix Magic Tricks](https://www.youtube.com/watch?v=DGhlQomeqKc) (video)

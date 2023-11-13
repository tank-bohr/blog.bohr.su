---
layout: post
title:  "Ruby ractors: chasing parallelism"
date:   2021-08-07 18:00:00 +0300
---

Ractor is one of the most significant features introduced in Ruby 3.0. It's aimed to bring to ruby an actor model, get around GVL limitations and provide ruby programmers with means for parallel programming. To better understand the rationale for ractors, it's necessary to refer to the history of the ruby language.

## 2004 - YARV begins

Let's start with the YARV. YARV is an acronym for Yet Another Ruby Virtual Machine, written by Koichi Sasada. Back then, Koichi was a CS student at the University of Tokyo in Japan. The main goals for the YARV project were

- To address all the performance issues "old ruby" had and to become the Fastest Ruby Interpreter.
- Native Thread Support
- To have some fun

Ruby, before YARV, had no VM at all and worked by traversing AST and evaluated each node. Wich was very slow.

## 2005 - Koichi makes a decision about native threads support and multithreading model.

It's a key decision for our discussion. Koichi considered three different thread models.

- Model 1: User-level threads aka Green Threads
- Model 2: Native-thread with giant VM lock aka GIL (like Python)
- Model 3: Native-thread with fine-grain lock (like Java VM)

The first model was implemented by Matz in the first ruby versions and didn't show good performance. Native threads could improve the performance and scalability characteristics of the language dramatically.

So Koichi chose the second model for some reasons

- It was much easier to implement
- No need to add a synchronization code. Much easier to write C-extensions
- In 2005 multi-core machines were rare beasts. So one couldn't achieve true parallelism any way.

Thus ruby got native threads support but still couldn't run in parallel. But Koichi had a plan how to bypass this limitation. He wanted to design a Multi-VM instances mechanism (MVM). Perl thread model called interpreter threads uses a similar approach - to provide a new Perl interpreter for each thread. And the use of interpreter-based threads in Perl is officially [discouraged](https://perldoc.perl.org/threads#WARNING) (sic!).

## GIL

So what is GIL, aka GVL (Global VM Lock)? GVL is a mechanism that protects VM internals. It doesn't give thread-safety guarantees for ruby programs. It gives thread-safety guarantees for ruby VM itself.

Let me show you a simple example.

{% highlight ruby %}
array = []

10.times.map do
  Thread.new do
    100.times do
      array << nil
    end
  end
end.each(&:join)

puts array.size
{% endhighlight %}

```
$ ruby script.rb
1000
```

GVL is very MRI-specific. Neither of the other rubies implementations doesn't have such a thing.

```
$ jruby script.rb
847
```

or even

```
...
ConcurrencyError: Detected invalid array contents due to unsynchronized modifications with concurrent users
           << at org/jruby/RubyArray.java:1292
  script.rb at script.rb:6
  script.rb at script.rb:5
...
```

In JRuby `Array#push` method isn't protected by GVL. So we can easily corrupt the data. For JRuby, this code should look like

{% highlight ruby %}
require 'java'
java_import java.util.concurrent.CopyOnWriteArrayList

array = CopyOnWriteArrayList.new

10.times.map do
  Thread.new do
    100.times do
      array.add(nil)
    end
  end
end.each(&:join)

puts array.size
{% endhighlight %}

## January 30, 2009 - Ruby 1.9.1 released

With a very humble entry in the [Changelog](https://svn.ruby-lang.org/repos/ruby/tags/v1_9_1_0/NEWS)

```
=== Implementation changes
...
* YARV
    * Ruby codes are compiled into opcodes before executed.
    * Native thread
...
```


## May 14, 2010 - Evan Phoenix releases Rubinius

Rubinius is an alternative Ruby implementation Based on LLVM. The goals were

* Thread safety. Rubinius intends to be thread-safe so you could embed more
  than one interpreter in a single application.

* Clean, readable code that is easy for users to understand and extend.

* Reliable, rock-solid code. Valgrind is used to help verify correctness.

* Bring modern research in virtual machines, garbage collectors, and compilers
  to the Ruby programming language.


In other words, Evan Phoenix wanted to have a Ruby VM without GVL but with JIT and more maintainable code.

## March 30, 2012 - Evan Phoenix releases puma web-server.

The main idea behind the puma web server was to utilize the power of native threads' true parallelism of the Rubinius from the author of Rubinius.

The original README said:

```
With Rubinius 2.0, Puma will utilize all cores on your CPU with real threads,
meaning you won't have to spawn multiple processes to increase throughput. You
can expect to see a similar benefit from JRuby.

On MRI, there is a Global Interpreter Lock (GIL) that ensures only one thread
can be run at a time. But if you're doing a lot of blocking IO (such as HTTP
calls to external APIs like Twitter), Puma still improves MRI's throughput by
allowing blocking IO to be run concurrently (EventMachine-based servers such as
Thin turn off this ability, requiring you to use special libraries).
Your mileage may vary. In order to get the best throughput, it is highly
recommended that you use a Ruby implementation with real threads like Rubinius
or JRuby.
```

And puma became the most popular web server in ruby. But not because it works best with Rubinius threads. People use it with MRI and choose puma for the low memory footprint. In my opinion, it's a very important thing. We can see a clear request from the community to get rid of the GVL. But in reality, nobody cares. If we look at what people do and not what people talk about on Twitter, we will see that the GVL is not a problem. Both puma and sidekiq have fork-based analogs ([unicorn](https://github.com/defunkt/unicorn) and [resque](https://github.com/resque/resque) respectively), free of the GVL's disadvantages. But puma and sidekiq are still the most popular tools in the ruby community despite the performance issues caused by GVL, and people continue to use them with MRI.
The same thing we could see with the [refinements](https://ruby-doc.org/core-3.0.2/doc/syntax/refinements_rdoc.html). The monkey-patching abuse is considered a serious problem in language, and refinements provide a very clear and elegant way to solve it. But refinements adoption is still [quite low for some reason](https://interblah.net/why-is-nobody-using-refinements).

## September 8, 2016 - Guilds proposal on Rubykaigi

On the Rubykaigi 2016, Koichi Sasada presented his proposal for the new concurrency model for Ruby 3. It was called "Guilds," it was a continuation of the MVM idea, and it became a prototype for the ruby ractors.

The idea of guilds was to bring convenient concurrency primitive as a first-class citizen, which will free from race conditions and allow proper parallel code execution. Matz had [approved](https://bugs.ruby-lang.org/issues/17100#note-14) it became a part of [Ruby 3x3](https://blog.heroku.com/ruby-3-by-3) strategy.


## December 25, 2020 - Ruby 3.0 was released.

Guilds were finally released as a part of Ruby 3.0. Koichi had [renamed](https://bugs.ruby-lang.org/issues/17100#Name-of-Ractor-and-Guild) them to Ractors. The feature is still in experimental mode, and you'll get a warning message while using it.

```
<internal:ractor>:267: warning: Ractor is experimental, and the behavior may change in future versions of Ruby! Also there are many implementation issues.
```

## What can we do with Ractors?

First of all, we can replace all the thread usage in tools like puma or sidekiq with ractors to get the benefits of the parallel execution (each ractor has its own GVL).

The next step, in my opinion, should be building an actor-based framework like [OTP](https://en.wikipedia.org/wiki/Open_Telecom_Platform). Although ruby now has actors, it's not enough to build an OTP-alike framework due to the lack of OTP's key component, the [links](https://learnyousomeerlang.com/errors-and-processes#links). Links allow building supervisor trees and implement the "Let It Crash" philosophy. To build such a framework, we need to build a more high-level communication mechanism that will support such primitives as links and monitors. It's still possible to do based on ractors message passing, but rather complicated. Also, neither blocking ractor API (`take` and `receive`) supports timeouts . It means it's impossible to build any non-blocking constructions based on the Ractors.


### Related links:

- [Abstract](http://www.atdot.net/yarv/oopsla2005eabstract-rc1.pdf) about YARV by Koichi Sasada
- YARV [presentations](http://www.atdot.net/yarv/#i-5-4) (in English)
- [Parallelism is a Myth in Ruby](https://www.igvita.com/2008/11/13/concurrency-is-a-myth-in-ruby/) by Ilya Grigorik
- 2016 RubyKaigi [presentation](http://www.atdot.net/~ko1/activities/2016_rubykaigi.pdf) with guilds proposal
- [Redmine ticket](https://bugs.ruby-lang.org/issues/17100) with the Ractors proposal
- Erlang processes [communication](https://erlang.org/doc/apps/erts/communication.html)

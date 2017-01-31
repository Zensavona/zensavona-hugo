+++
date = "2015-11-14T12:46:52+11:00"
title = "How To Make The Elixir Shell (IEX) Save History"
draft = false
+++

<p><p>If you've ever used <code>irb</code>, <code>node</code> or pretty much any other command line REPL, you probably really miss being able to use the up and down keys to recall your last commands from a previous session in <code>iex</code>.</p>

<p>Today I discovered <a href="https://github.com/ferd/erlang-history">a package</a> which hacks this functionality into the Erlang shell, and thus also works for Elixir.</p>

<p>It's very simple to install:</p>

<pre><code>git clone git@github.com:ferd/erlang-history.git  
cd erlang-history  
make install  
</code></pre>

<p>Note that you may need to run the last line as <code>sudo make install</code> depending how and where you installed Erlang/Elixir. I needed to do this, and I installed Erlang with Homebrew on OS X.</p>

<p>Enjoy!</p></p>

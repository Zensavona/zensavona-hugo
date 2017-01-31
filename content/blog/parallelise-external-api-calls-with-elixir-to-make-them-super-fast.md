+++
date = "2015-10-26T12:46:52+11:00"
title = "Parallelise external API calls with Elixir to make them super fast"
draft = false
+++

<p><p>I'm working on a Phoenix application which uses the Instagram API via the <a href="https://github.com/zensavona/elixtagram">Elixtagram library</a>. There is a particular page where I need to display a bunch of users who need to be retrieved from the API... Here's the "before" (sequential) code:</p>

<pre><code>def users_list(%Account{favourite_users: users}) do  
  Elixtagram.configure
  users
  |&gt; Enum.map(fn(u) -&gt; Elixtagram.user(u.ig_id).username end)
  |&gt; Enum.join(", ")
end  
</code></pre>

<p>What I'm doing here is simply passing in a list of Instagram IDs and returning a string of comma separated usernames. The obvious problem here is that Elixtagram is making an API call which can take some relatively large amount of milliseconds, and it's silly to wait for each one to finish before starting the next one, especially when we're using a language with such easily accessible concurrency as Elixir.</p>

<p>Here's the "fixed" code:</p>

<pre><code>def users_list(%Account{favourite_users: users}) do  
  Elixtagram.configure
  users
  |&gt; Enum.map(fn(u) -&gt; Task.async(fn -&gt; Elixtagram.user(u.ig_id).username end) end)
  |&gt; Enum.map(&amp;Task.await/1)
  |&gt; Enum.join(", ")
end  
</code></pre>

<p>The difference here is that instead of blocking for each request in the <code>Enum.map</code>, I'm filling the list up with async tasks, which I then wait for on the next line. What this means is that the amount of requests we make here has little impact on the performance of the code (until we saturate the network...).</p>

<p>Good luck with your concurrency adventures, friends!</p></p>

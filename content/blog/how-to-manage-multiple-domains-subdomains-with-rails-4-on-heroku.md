+++
date = "2015-05-06T12:46:52+11:00"
title = "How To Manage Multiple Domains & Subdomains with Rails 4 on Heroku"
draft = false
+++

<p><p>Recently I've been working on a multi-tenant blog hosting application with a Rails front end, and this required us to let users set their own domains. Here's how to dynamically route your Rails application with multiple domains and subdomains (of yours and theirs).</p>

<h2 id="dynamicroutinginrails">Dynamic routing in Rails</h2>

<p>In my application there is the concept of a <code>Site</code> which always has a <code>slug</code>, which becomes <code>slug.mydomain.com</code>. And sometimes has a <code>domain</code>, which belongs to them.</p>

<p>What we need to do is match the request based on a <code>constraint</code> which is simply a class we create, which implements the <code>matches?</code> method. This is very simple as you will see below:</p>

<p><code>routes.rb</code></p>

<pre><code>...
class CustomDomainConstraint  
  # Implement the .matches? method and pass in the request object
  def self.matches? request
    matching_site?(request)
  end

  def self.matching_site? request
    # handle the case of the user's domain being either www. or a root domain with one query
    if request.subdomain == 'www'
      req = request.host[4..-1]
    else
      req = request.host
    end

    # first test if there exists a Site with a domain which matches the request,
    # if not, check the subdomain. If none are found, the the 'match' will not match anything
    Site.where(:domain =&gt; req).any? || Site.where(:slug =&gt; request.subdomain).any?
  end
end

match '/', :to =&gt; 'sites#index', :constraints =&gt; CustomDomainConstraint, via: :all  
match '/category/:slug' =&gt; 'sites#category', :constraints =&gt; CustomDomainConstraint, via: :all  
match '/:slug', to: 'sites#show', :constraints =&gt; CustomDomainConstraint, via: :all  
...
</code></pre>

<p>It's worth noting here that if your app allows users to specify their own domains, you should have a system of blacklisting your own domain/s and the <code>www</code> subdomain/slug.</p>

<p>Then all I need to do in the <code>sites_controller.rb</code> is implement a <code>before_filter</code> which finds the <code>Site</code> based on some request data, this is the code I used:</p>

<pre><code>before_filter :find_site

...

private

  def find_site
    # generalise away the potential www. or root variants of the domain name
    if request.subdomain == 'www'
      req = request.host[4..-1]
    else
      req = request.host
    end

    # first test if there exists a Site with the requested domain,
    # then check if it's a subdomain of the application's main domain
    @site = Site.find_by(domain: req) || Site.find_by(slug: request.subdomain)

    # if a matching site wasn't found, redirect the user to the www.&lt;root url&gt;
    redirect_to root_url(subdomain: 'www') unless @site
  end
</code></pre>

<h2 id="pointingdomainsandsubdomainstoyourrailsapplication">Pointing Domains And Subdomains To Your Rails Application</h2>

<p>All you need to do, or have your users do is <code>CNAME</code> their domain name to your main domain which points to your app.</p>

<h2 id="multipledomainsonherokuwithrails">Multiple Domains On Heroku With Rails</h2>

<p>My application lives on Heroku, and they use virtual hosts to route domain names, this means we need to tell them when we want to <code>CNAME</code> a new domain to our instance.  To do this I just implemented an <code>after_save</code> on my <code>Site</code> model which uses Heroku's API to register a new domain with my application.</p>

<p><code>site.rb</code></p>

<pre><code>after_save do |site|  
  heroku_environments = %w(production staging)
  if site.domain &amp;&amp; (heroku_environments.include? Rails.env)
    added = false
    heroku = Heroku::API.new(api_key: ENV['HEROKU_API_KEY'])
    heroku.get_domains(ENV['APP_NAME']).data[:body].each do |domain|
      added = true if domain['domain'] == site.domain
    end

    unless added
      heroku.post_domain(ENV['APP_NAME'], site.domain)
      heroku.post_domain(ENV['APP_NAME'], "www.#{site.domain}")
    end
  end
end  
</code></pre>

<p>I was looking around the internet for a while before I figured out how to do this, so I hope this makes your day easier :) Comment below or tweet me if you have any questions or suggestions!</p></p>

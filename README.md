SitemapGenerator
================

This plugin enables ['enterprise-class'][enterprise_class] Google Sitemaps to be easily generated for a Rails site as a rake task, using a simple 'Rails Routes'-like DSL.

**Now supporting Rails 3 as of version 0.2.5!**

Foreword
-------

Unfortunately, Adam Salter passed away in 2009.  Those who knew him know what an amazing guy he was, and what an excellent Rails programmer he was.  His passing is a great loss to the Rails community.

[Karl Varga](http://github.com/kjvarga) has taken over development of SitemapGenerator.  The canonical repository is [http://github.com/kjvarga/sitemap_generator][canonical_repo]

Installation
=======

**Rails 3:**

1. Add the gem to your <tt>Gemspec</tt>

    <code>gem 'sitemap_generator'</code>

2. `$ rake sitemap:install`

**Rails 2.x: As a gem**

1. Add the gem as a dependency in your <tt>config/environment.rb</tt>

    <code>config.gem 'sitemap_generator', :lib => false</code>

2. `$ rake gems:install`

3. Add the following to your <tt>RAILS_ROOT/Rakefile</tt>

    <pre>begin
      require 'sitemap_generator/tasks'
    rescue Exception => e
      puts "Warning, couldn't load gem tasks: #{e.message}! Skipping..."
    end</pre>

4. `$ rake sitemap:install`

**Rails 2.x: As a plugin**

1. <code>$ ./script/plugin install git://github.com/kjvarga/sitemap_generator.git</code>

----

Installation creates a <tt>config/sitemap.rb</tt> file which will contain your logic for generating the Sitemap files.  If you want to create this file manually run <code>rake sitemap:install</code>.

You can run <code>rake sitemap:refresh</code> as needed to create Sitemap files. This will also ping all the ['major'][sitemap_engines] search engines.  If you want to disable all non-essential output run the rake task with <code>rake -s sitemap:refresh</code>.

Sitemaps with many urls (100,000+) take quite a long time to generate, so if you need to refresh your Sitemaps regularly you can set the rake task up as a cron job. Most cron agents will only send you an email if there is output from the cron task.

Optionally, you can add the following to your <code>public/robots.txt</code> file, so that robots can find the sitemap file.

    Sitemap: <hostname>/sitemap_index.xml.gz

The Sitemap URL in the robots file should be the complete URL to the Sitemap Index, such as <tt>http://www.example.org/sitemap_index.xml.gz</tt>


Example 'config/sitemap.rb'
==========

    # Set the host name for URL creation
    SitemapGenerator::Sitemap.default_host = "http://www.example.com"

    SitemapGenerator::Sitemap.add_links do |sitemap|
      # Put links creation logic here.
      #
      # The Root Path ('/') and Sitemap Index file are added automatically.
      # Links are added to the Sitemap output in the order they are specified.
      #
      # Usage: sitemap.add path, options
      #        (default options are used if you don't specify them)
      #
      # Defaults: :priority => 0.5, :changefreq => 'weekly',
      #           :lastmod => Time.now, :host => default_host


      # Examples:

      # add '/articles'
      sitemap.add articles_path, :priority => 0.7, :changefreq => 'daily'

      # add all individual articles
      Article.find(:all).each do |a|
        sitemap.add article_path(a), :lastmod => a.updated_at
      end

      # add merchant path
      sitemap.add '/purchase', :priority => 0.7, :host => "https://www.example.com"

      # add all individual news with images
      News.all.each do |n|
        sitemap.add news_path(n), :lastmod => n.updated_at, :images=>n.images.collect{ |r| :loc=>r.image.url, :title=>r.image.name }
      end

    end

    # Including Sitemaps from Rails Engines.
    #
    # These Sitemaps should be almost identical to a regular Sitemap file except
    # they needn't define their own SitemapGenerator::Sitemap.default_host since
    # they will undoubtedly share the host name of the application they belong to.
    #
    # As an example, say we have a Rails Engine in vendor/plugins/cadability_client
    # We can include its Sitemap here as follows:
    #
    file = File.join(Rails.root, 'vendor/plugins/cadability_client/config/sitemap.rb')
    eval(open(file).read, binding, file)

Raison d'être
-------

Most of the Sitemap plugins out there seem to try to recreate the Sitemap links by iterating the Rails routes. In some cases this is possible, but for a great deal of cases it isn't.

a) There are probably quite a few routes in your routes file that don't need inclusion in the Sitemap. (AJAX routes I'm looking at you.)

and

b) How would you infer the correct series of links for the following route?

    map.zipcode 'location/:state/:city/:zipcode', :controller => 'zipcode', :action => 'index'

Don't tell me it's trivial, because it isn't. It just looks trivial.

So my idea is to have another file similar to 'routes.rb' called 'sitemap.rb', where you can define what goes into the Sitemap.

Here's my solution:

    Zipcode.find(:all, :include => :city).each do |z|
      sitemap.add zipcode_path(:state => z.city.state, :city => z.city, :zipcode => z)
    end

Easy hey?

Other Sitemap settings for the link, like `lastmod`, `priority`, `changefreq` and `host` are entered automatically, although you can override them if you need to.

Other "difficult" Sitemap issues, solved by this plugin:

- Support for more than 50,000 urls (using a Sitemap Index file)
- Gzip of Sitemap files
- Variable priority of links
- Paging/sorting links (e.g. my_list?page=3)
- SSL host links (e.g. https:)
- Rails apps which are installed on a sub-path (e.g. example.com/blog_app/)

Compatibility
=======

Tested and working on:

- **Rails** 3.0.0, sitemap_generator version >= 0.2.5
- **Rails** 1.x - 2.3.5 sitemap_generator version < 0.2.5
- **Ruby** 1.8.7, 1.9.1

Notes
=======

1) For large sitemaps it may be useful to split your generation into batches to avoid running out of memory. E.g.:

    # add movies
    Movie.find_in_batches(:batch_size => 1000) do |movies|
      movies.each do |movie|
        sitemap.add "/movies/show/#{movie.to_param}", :lastmod => movie.updated_at, :changefreq => 'weekly'
      end
    end

2) New Capistrano deploys will remove your Sitemap files, unless you run `rake sitemap:refresh`. The way around this is to create a cap task:

    after "deploy:update_code", "deploy:copy_old_sitemap"

    namespace :deploy do
      task :copy_old_sitemap do
          run "if [ -e #{previous_release}/public/sitemap_index.xml.gz ]; then cp #{previous_release}/public/sitemap* #{current_release}/public/; fi"
      end
    end

3) If generation of your sitemap fails for some reason, the old sitemap will remain in public/.  This ensures that robots will always find a valid sitemap.  Running silently (`rake -s sitemap:refresh`) and with email forwarding setup you'll only get an email if your sitemap fails to build, and no notification when everything is fine - which will be most of the time.

Known Bugs
========

- Sitemaps.org [states][sitemaps_org] that no Sitemap XML file should be more than 10Mb uncompressed. The plugin will warn you about this, but does nothing to avoid it (like move some URLs into a later file).
- There's no check on the size of a URL which [isn't supposed to exceed 2,048 bytes][sitemaps_xml].
- Currently only supports one Sitemap Index file, which can contain 50,000 Sitemap files which can each contain 50,000 urls, so it _only_ supports up to 2,500,000,000 (2.5 billion) urls. I personally have no need of support for more urls, but plugin could be improved to support this.

Wishlist
========

- Auto coverage testing.  Generate a report of broken URLs by checking the status codes of each page in the sitemap.

Thanks (in no particular order)
========

- [Dan Pickett](http://github.com/dpickett)
- [Rob Biedenharn](http://github.com/rab)
- [Richie Vos](http://github.com/jerryvos)
- [Adrian Mugnolo](http://github.com/xymbol)


Copyright (c) 2009 Adam @ [Codebright.net][cb], released under the MIT license

[canonical_repo]:http://github.com/kjvarga/sitemap_generator
[enterprise_class]:https://twitter.com/dhh/status/1631034662 "I use enterprise in the same sense the Phusion guys do - i.e. Enterprise Ruby. Please don't look down on my use of the word 'enterprise' to represent being a cut above. It doesn't mean you ever have to work for a company the size of IBM. Or constantly fight inertia, writing crappy software, adhering to change management practices and spending hours in meetings... Not that there's anything wrong with that - Wait, what?"
[sitemap_engines]:http://en.wikipedia.org/wiki/Sitemap_index "http://en.wikipedia.org/wiki/Sitemap_index"
[sitemaps_org]:http://www.sitemaps.org/protocol.php "http://www.sitemaps.org/protocol.php"
[sitemaps_xml]:http://www.sitemaps.org/protocol.php#xmlTagDefinitions "XML Tag Definitions"
[sitemap_generator_usage]:http://wiki.github.com/adamsalter/sitemap_generator/sitemapgenerator-usage "http://wiki.github.com/adamsalter/sitemap_generator/sitemapgenerator-usage"
[boost_juice]:http://www.boostjuice.com.au/ "Mmmm, sweet, sweet Boost Juice."
[cb]:http://codebright.net "http://codebright.net"
[sitemap_images]:http://www.google.com/support/webmasters/bin/answer.py?answer=178636


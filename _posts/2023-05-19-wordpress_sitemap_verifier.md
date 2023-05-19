---
layout: post
title: "Verify all urls described by the sitemap plugin in Wordpress"
date:  2023-05-19 13:56:00
tags:  ruby wordpress sitemap networking http 
---

I developed a toy Ruby application to verify all urls described by the sitemap plugin in Wordpress instance. 
For this case, the customer has several websites based on Wordpress and they wanted to verify if all urls were 
working properly because they did a DNS migration recently. For the verified URLs the application must track the HTTP 
status code and the response time (start time, end time, duration) per request.

Additionally, the application must be able to run multiple requests asynchronously to speed up the verification process.
So, I implemented this application based on the Ruby HTTP library, and the Nokogiri and Async gems. 

The source code is available on [this github repository](https://github.com/karmatr0n/sitemap_verifier).

## Sequence diagram
In this secuence diagram is possible to see the main classes and their interactions.
![ethernet encapsulation](/img/sitemap_verifier/sequence_diagram.png)
Additionally, the HTTPHelper module is required to reuse the methods to make HTTP requests.

## HTTPHelper
This module provides methods for creating an HTTP GET request with a random user agent,
as well as initializing an HTTP connection with or without SSL support and its purpose
is to be a mixin for the URLChecker and SiteMapper classes.

{% highlight ruby linenos %}
module HttpHelper
USER_AGENTS = [
'Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/81.0',
'Mozilla/5.0 (compatible; MSIE 10.0.0; Windows Phone OS 8.0.0; Trident/6.0.0; IEMobile/10.0.0; Lumia 630',
'Mozilla/5.0 (iPad; CPU OS 6_0_1 like Mac OS X) AppleWebKit/536.26 (KHTML, like Gecko) Version/6.0 Mobile/10A523 Safari/8536.25'
].freeze

def http_get_request(path)
request = Net::HTTP::Get.new(path)
request['User-Agent'] = user_agent
request
end

def user_agent
USER_AGENTS.sample
end

def http_request(uri)
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = uri.scheme == 'https'
http.verify_mode = OpenSSL::SSL::VERIFY_NONE if http.use_ssl?
http
end
end
{% endhighlight %}

## URLChecker
The URLChecker class verifies the status of an URL  by making an HTTP request using the HttpHelper module methods and also returns 
statistics such as the URL, status code, start time, end time, duration, and whether the URL was successfully verified.

{% highlight ruby linenos %}
class URLChecker
  include HttpHelper
  attr_reader :status_code, :uri, :url_verified

  def initialize(url)
    @uri = URI.parse(url)
    @status_code = nil
    @start_time = Time.now
    @url_verified = false
  end

  def verify_status
    retries ||= 0
    http = http_request(@uri)
    response = http.request(http_get_request(@uri.path))
    @end_time = Time.now
    @url_verified = true
    @status_code = response.code
  rescue Net::OpenTimeout, OpenSSL::SSL::SSLError => _e
    retry if (retries += 1) <= 2
  end

  def stats
    {
      url: @uri.to_s,
      status_code: @status_code,
      start_time: @start_time,
      end_time: @end_time,
      duration: @end_time.to_f - @start_time.to_f,
      url_verified: @url_verified
    }
  end
end
{% endhighlight %}

## SiteMapper
The code of the SiteMapper class maps URLs from the XML described in the sitemap of a Wordpress instance. 
It includes methods to initialize the URL, make an HTTP request using the HttpHelper methods, retrieve the XML response, 
and extract URLs from the XML by parsing it with Nokogiri. 

{% highlight ruby linenos %}
class SiteMapper
  include HttpHelper
  attr_reader :uri

  def initialize(url)
    @uri = URI.parse(url)
  end

  def map_urls
    retries ||= 0
    http = http_request(@uri)
    response = http.request(http_get_request(@uri.path))
    response.code == '200' ? urls_from_xml(response.body) : []
  rescue Net::OpenTimeout, OpenSSL::SSL::SSLError => _e
    (retries += 1) <= 2 ? retry : []
  end

  def urls_from_xml(xml)
    doc = Nokogiri::XML(xml)
    doc.xpath('//xmlns:loc').map(&:text)
  end
end
{% endhighlight %}

## SitemapVerifier
This class verifies the URLs from a sitemap and gather statistics for analysis or reporting purposes. It includes methods 
to initialize the verifier with the sitemap URL, debug mode, and maximum number of asynchronous requests. 
It uses the SiteMapper and URLChecker classes to map and verify URLs asynchronously, as well as stores the stats per
requests to save them in a JSON file named as the sitemap's host with the current timestamp.
  
{% highlight ruby linenos %}
class SitemapVerifier
  attr_reader :stats, :debug, :max_async_requests

  def initialize(sitemap_url, debug: false, max_async_requests: 30)
    @site_mapper = SiteMapper.new(sitemap_url)
    @stats = []
    @debug = debug
    @max_async_requests = max_async_requests
    @output_filename = nil
  end

  def verify_urls
    all_urls = map_urls
    all_urls_size = all_urls.size
    max_requests = all_urls_size.positive? && all_urls_size < max_async_requests ? all_urls_size : max_async_requests
    all_urls.each_cons(max_requests).each do |urls|
      async_scan_urls(urls)
    end
  end

  def map_urls
    @site_mapper.map_urls.map do |child_url|
      puts("Getting urls from: #{child_url}") if debug
      child_map = SiteMapper.new(child_url)
      child_map.map_urls
    end.flatten
  end

  def async_scan_urls(urls)
    Async do
      urls.each do |url|
        Async do
          url_checker = URLChecker.new(url)
          url_checker.verify_status
          puts(url_checker.stats) if debug
          stats.push(url_checker.stats)
        end
      end
    end
  end

  def save_json(filename = output_filename)
    file = File.new(filename, 'w')
    file.puts(JSON.pretty_generate(stats))
    file.close
  end

  def output_filename
    @output_filename ||= "#{@site_mapper.uri.host}_#{Time.now.to_i}.json"
  end
end
{% endhighlight %}

## Usage

1. Clone the repository:

```bash
git clone https://github.com/karmatr0n/sitemap_verifier
```

2. Install the dependencies:

```bash
bundle install
```  

3. Run the script:

```bash
ruby sitemap_verifier.rb https://example.com/sitemap_index.xml
```

## Know issues
* The application does not support needs to finish the URLs verification process before saving the JSON file.


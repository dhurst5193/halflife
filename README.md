Halflife
========

This repository has been created to hold the results of experiments on a random sample of the holdings of the [Open UK Web Archive](http://www.webarchive.org.uk/).  

* Compare with LDUKWA

* PLUS: Is same/similar content at different URLs.
* PLUS: Are our holdings unique or is it all in IA?
* Check fraction of 2013 LDWA that is already gone. Extend to plot urls/host versus lifespan?
* Trial a site url, various tools???
* Also sample host root pages, via url_type?
* reliability of ssdeep method
* look at fine-structure of ssdeep similarity results (i.e. edit distance). 
* Check for moved content (gap, use Solr indexes of hashes and fuzzy hashes), 

* 200s v soft 404s?
* compare entities of example.com/realurl, example.com/garbage/realurl, example.com/lsfkj, etc. if they match then soft 404s . 
* HTTP redirect analysis also shown to be feasible to ID soft404s [Detecting Soft Errors by Redirection Classification](www2009.eprints.org/140/1/p1119.pdf)

* PLUS Idea: allowing shallow contextual crawl to collect metadata, in-links, etc. CommonCrawl, IA.
* In UI, are we able to emphasis the lost ones.

PLUS? http://en.wikipedia.org/wiki/The_Million_Dollar_Homepage

See <http://britishlibrary.typepad.co.uk/webarchive/2014/10/what-is-still-on-the-web-after-10-years-of-archiving-.html>

* Mention Interject in GA talk?


Dependencies
------------

* ssdeep 2.13 or later

    $ pip install BeautifulSoup


Link Half-Life
--------------

The goal is to randomly sample URLs from previous years of the web archive, and for each one determine its current status. Has is long since gone? Or is it still online? With a few thousand URLs, it should be possible to build up a reasonably accurate picture of the 'half-life' of the URLs in our archive.

Step 1. Random Sampling
-----------------------

All of our content is indexing in Apache Solr, so we can use that to generate random samples of content for each crawl year. The largest sample contains one hundred thousand archived URLs, hence the name of this repository.

Faceting by year looks something like:

<http://localhost:8080/discovery/select?q=*%3A*&wt=json&indent=true&rows=0&facet.range.gap=%2B1YEAR&facet.range=crawl_date&f.crawl_date.facet.range.start=1980-01-01T00:00:00Z&f.crawl_date.facet.range.end=2020-01-01T00:00:00Z&facet=on>

Then something like this to sample randomly, using the Solr random sort feature:

<http://chrome.bl.uk:8080/solr/select/?q=*:*&rows=1&sort=random_2%20desc&fq=timestamp:[2004-01-01T00:00:00Z%20TO%202005-01-01T00:00:00Z]>

The `tools/halflife/yearwise_sampler.py` implements this logic. 

    $ cd archive-sample
    $ python ../tools/halflife/yearwise_sampler.py

It queries Solr for year-wise samples of various sizes, and them process the response from Solr to generate source sample files of the form:

    timestamp, url, title, first_text, ssdeep_hash
    
The text is processed and normalised before output and hashing, so for the smallest sample size (100), the source JSON is also included here in order to make the hashing process more transparent.

Step 2. Checking Current Status Of The Samples
----------------------------------------------

Then, periodically, for each sample, we revisit those links and attempt to determine what has happened to them. We use the following taxonomy:

* **OK** - Host and URL known: got a 200 response at the original URL
* **MOVED** - Host and URL known: got a 200 response after following any 3xx redirects.
* **MISSING** - Host is known, but URL returned a 404 or similar, after zero or more 3xx redirects.
* **ERROR** - Host is known, but URL returned a 500 or similar, after zero or more 3xx redirects.
* **GONE** - No connection possible (UNRESOLVABLE, UNREACHABLE, CONNECTION-REFUSED etc.).

This works reasonably well, although it does not compare the contents, so the **OK** and **MOVED** really only refers to the URL itself. If we add the ability to determine similarity of the content, then we have:

* **UNCHANGED** - Host and URL known: got an identical 200 response at the original URL
* **MOVED** - Host and URL known: got an identical 200 response after following any 3xx redirects.
* **CHANGED** - Host and URL known: got an different 200 response at the original URL
* **REDIRECTED** - Host and URL known: got an different 200 response after following any 3xx redirects.
* **MISSING** - Host is known, but URL returned a 404 or similar, after zero or more 3xx redirects.
* **ERROR** - Host is known, but URL returned a 500 or similar, after zero or more 3xx redirects.
* **GONE** - No connection possible (UNRESOLVABLE, UNREACHABLE, CONNECTION-REFUSED etc.).

Although **CHANGED** and **REDIRECTED** could probably be merged really.

So, we process the source samples and attempt to resolve each URL in turn. In the root folder:

    $ python tools/halflife/sample_scanner.py

The results are output in a folder based on the date of execution, with columns of the form:

    year, quarter, month, state, status_code, reason, identicalness, similarity, url

This can then be further processed to generate visual representations of what is going on.


Step 3. Publish The Overall Status
----------------------------------

The results from the hundred thousand are then visualised. A simple example is here:

    $ python tools/halflife/scan_graph.py sample-of-100.csv
    
But a more detailed view is generated using the IPython Notebook

    half-life.ipynb




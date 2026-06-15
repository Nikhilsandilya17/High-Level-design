---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠== You can decompress Drawing data with the command palette: 'Decompress current Excalidraw file'. For more info check in plugin settings under 'Saving'


# Excalidraw Data

## Text Elements
* Additional Deep dives

1. How to handle dynamic content: 
Many websites are built with JavaScript frameworks like React or Angular. This means that the content of the page is not in the HTML that is returned by the server, 
but is instead loaded dynamically by the JavaScript. To handle this, we'll need to use a headless browser like Puppeteer to render the page and extract the content.

2. How to monitor the health of the system: We'll want to monitor the health of the system to ensure that everything is running smoothly. We can use a monitoring service 
like Datadog or New Relic to monitor the performance of the crawlers and parser workers and to alert us if anything goes wrong.

3. How to handle large files: Some websites have very large files that we may not want to download. We can send an HTTP HEAD request first to check the Content-Length 
header and skip files that exceed a threshold (say 10MB) without downloading the full content.

4. How to handle continual updates: While our requirements are for a one-time crawl, we may want to consider how we would handle continual updates to the data. 
This could be that we plan to re-train the model every month or that our crawler is for a search engine that needs to be updated regularly. 
I'd suggest adding a new component "URL Scheduler" that is responsible for scheduling URLs to be crawled. So rather than putting URLs on the queue directly from 
the parser workers, the parser workers would put URLs in the Metadata DB and the URL Scheduler would be responsible for scheduling URLs to be crawled by using some logic based on last crawl time, popularity, etc.

5. How to implement priority crawling: If you need to prioritize certain URLs (e.g., popular domains first), you can use multiple SQS queues with different priorities 
and have crawlers poll the high-priority queue first. Kafka doesn't natively support priority-based consumption, but you can achieve a similar effect with separate topics per priority level. ^B9GK9fhg

A web crawler is a program that automatically traverses the web by downloading web pages and following links from one page to another.
It is used to index web for search engines or monitor websites for changes.

 ^0tj4E8SB

Core Entities

1. Text data
2. URL metadata
3. Domain metadata

Interface

1. input : Set of seed URLs
2. output : Text data

Data flow

1. take seed URLs from a frontier and fetch the IP from DNS for the URL
2. Fetch HTML
3. Extract text from HTML
4. Store the text in Database
5. Extract the URLs from text and add it to frontier 
6. Repeat steps 1-5 until frontier is empty. ^AVBi7hDT

Frontier 
queue (Kafka/SQS) ^4QBohfXM

Set of workers(lets call them crawler) ^U3WwJ0Rs

- pull a URL from the queue
- fetch the webpage
- extract the text
- extract the URL ^WlUcTY82

DNS ^CoYHcRgD

Webpage ^rzWWp6Y7

S3 text
data ^BQKIj33p

once it 
gets the
data it
is going to 
store it ^5oYfviD5

Internal to our system ^P1PWTCeM

External to our system ^GYC3U7P6

- start wit seed URLs ^izVf7YfN

put extracted URL back to
queue ^Il04PNFy

Basic HLD ^2qbpWVgB

* Deep dives

1. How can we ensure we are fault tolerant and dont loose progress?
- The first thing we should notice is that our crawler service is doing a lot
Its hitting DNS, fetching web pages, extracting text data, extracting new URLs to add to frontier queue.
- Fetching web pages is the most likely task to fail. The internet is a messy place and there are many reasons why a fetch might fail. The server might be down
 the connection might be slow, the page might be too large.
- To handle this we should break the crawler service into smaller, pipelined stages. This way if there is a failure in any stage, we can retry that stage without losing progress on the rest of the data. 

# Heres how we can break crawler stage :-

a. URL fetcher = This stage fetches the HTML of the web page from the external server. If there is a failure, we can retry the fetch without losing progress on the rest of the data. We will store the raw HTML 
in blob storage for later processing.
b. Text and URL extraction = This stage extracts the text data from the HTML and extracts any linked URLs to add to the frontier queue. There is an arguement that text extraction and URL extraction should be separate stages, but these tasks are both simple 
and can be in parallel without much overhead so I would prefer to simplify the design and combine them into a single stage.
 ^WB8PP77F

In order to make this work, we need to add some additional state. We will add a Metadata DB(dynamo Db is fine here, PSql or MySql could also work) with a table for URLs that have fetched and processed.
As a starting point this will store the link to the blob storage where the HTML is stored and the link to the blob storage where the text data is stored.
This is important because it is an anti pattern to store the raw HTML in the queue itself. Queues are not optimised for large payloads and it would be expensive to store the HTML in the queue.
Instead the queue message will just be the id of the URL in the Metadata DB.

Now it will allows us to be robust to changing requirements. You can image that the ML team consuming this data wants to change the text extraction process. A simple example could be including image alt text in the extracted text.
If we we have a separate stage for text extraction, we can easily swap out the text extraction function without needing to redo the expensive part of fetching the web pages.

# What about if we fail to fetch a URL?
Might be server down or slow, to confirm we need to retry.

-> Bad solution : In memory timer
Approach: The easiset and worst thing we could do is just to wait a few seconds using an in-memory timer and try again
Challenges: Beyond any issues with politeness, which we will address next, this isn't robust because if a crawler were to go down, we would lose the timer.
It is also very likely that the fetch wont succeed in just a few seconds. We need something sort of exponential backoff.

-> Good solution : Kafka with Manual exponential backoff
Approach: Kafka does not support retries out of box, but we could implement them ourselves. We could have a separate topic for failed URLs and a separate service that reads from this topic and retries the fetch with exponential backoff.
Challenges: Just complex to implement and maintain.

-> Great solution : SQS with exponential backoff
Approach: SQS does not have a built in exponential backoff setting, but it provides the primitives to build one cleanly. The key mechanism is the visibility timeout: when a consumer receives a message but does not delete it, the message becomes visible again after the 
visibility timeout expires (default 30 seconds, configurable upto 12 hrs)
Challenges: We dont want to retry indefinitely. After a certain number of failures ( configured via the queue redrive policy using maxReceiveCount), the message is automatically moved to DLQ. For our purposes we will consider the site offline after 5 retries.





 ^wxIfCpWe

# What happens when a crawler goes down?
- We spin a new one, We will just have to make sure that the half finished URL is not lost.
- Good news is the URL will stay in the queue until it is confirmed to have been fetched by a crawler and the HTML is stored in blob storage. 
This way if a crawler goes down the URL will be picked up by another craweler and the process will continue. How Kafka and SQS handles this

* Apache Kafka
- kafka retains messages in a log and does not remove them even after they are read. Crawlers track there progress via offsets, which are not updated in Kafka until the URL 
is successfully fetched and processed. If a crawler fails, the next one picks up right where the last one left off, ensuring no data is lost.
 ^QqO9gECT

* Amazon SQS
- With SQS, messages remain in the queue until they are explicitly deleted. A visibility timeout hides a message from other crawlers once its fetched. If the crawler fail before 
confirming successful processing, the message will automatically become visible again after the timeout expires, allowing another crawler to attempt the fetch. On the other hand
once the HTML is stored in blob storage, the crawler will delete the message from the queue, ensuring it is not processed again.

Of course the same applies if a parsing worker goes down. The URL will remain in the queue until it is confirmed to have been processed and the text data is stored in blob storage.
Given SQS visibility timeout mechanism (which makes implementing exponential backoff straighforward), automatic DLQ support, and managed scaling.
We will use SQS for our system.  ^gxTjMK65

2. How can we ensure politeness and adhere to robots.txt?
- First thing, what is politeness and what is a robots.txt file?
- Politeness refers to being resepectful with the resources of the websites we are crawling.
This involves ensuring that our crawling activity does not disrupt the normal function of the site by overloading its servers
- robots.txt is a file that websites use to communicate with web crawlers. It tells crawlers which pages they are allowed to crawl and which pages they are not.
It also tells crawler how frequently they can crawl the site
- Ex:     User-agent: *
           Disallow: /private/
           Crawl-delay: 10

The user-agent line specifies which crawler the rules apply to. In this case '*' means all crawlers. The disallow line specifies which pages the crawler is not allowed to crawl. In this case
the crawler is not allowed to crawl any pages in the /private/ directory. The Crawl-delay line specifies how many seconds the crawler should wait betwen requests, in this case 10 seconds.

To ensure politeness and adhere to robots.txt, we will need to do two things:
a. Respect robots.txt: Before crawling a page, we will need to check the robots.txt file to see if we are allowed to crawl the page. If we are not allowed to crawl the page, we will need to skip it. 
We will also need to respect the Crawl-delay directive and wait the specified number of seconds between requests.
b. Rate limiting: We will want to limit the number of requests we make to any single domain. The industry standard is to limit the number of requests to 1 request per second.

First we need to save the robots.txt file for each domain we crawl. with this file saved, we can check it before crawling a page.
We need to consider two things:
a. Is the crawler allowed to crawl the page? Easy, just check the Disallow directive and confirm that this page is not disallowed. If it is, we can ack the message (remove it from the queue) 
and move on to the next URL.
b. How long should we wait between requests? This is a bit more complex. We need to check the Crawl-delay directive and wait the specified number of seconds between requests.

TO handle the crawl delay, we need to introduce some additional state. We can add a Domain table to our Metadata DB that stores the last time we crawled each domain.
This way we can check the Crawl-delay directive and wait the specified number of seconds before crawling the next page. If we pull a URL off the queue for a domain that we have recently crawled, 
we use SQS ChangeMessageVisibility API to extend the visibility timeout, deferring when the message visible again.

** Since we have multiple crawlers running in parallel, two crawlers could pull URLs from the same domain simultaneously and both see a stale "last crawl time". To prevent this we can use atomic operations
(like Redis SET with NX and a TTL matching the crawl delay) to acquire a per-domain lock before  crawling. If a crawler cant acquire the lock, it defers the message using ChangeMessageVisibility.


# Flow
a. Fetch the robots.txt file for the domain.
b. Parse the robots.txt file and store it in the Metadata DB.
c. When we pull a URL off the queue, check the rules stored in the Metadata DB for that domain.
d. If the URL is disallowed, ack the message and move on to the next URL.
e. If the URL is allowed, check the Crawl-delay directive.
f. If the Crawl-delay time has not passed since the last crawl, use ChangeMessageVisibility to extend the visibility timeout and defer reprocessing.
g. If the Crawl-delay time has passed, crawl the page and update the last crawl time for the domain. ^VQrvyeax

# What about Rate limiting?
- We also need to respect the rate limit of 1 domain a second. With multiple crawlers, this can get a little trickier since, in theory, all N crawlers could be hitting a single domain at the same time.
- We can implement a global, domain-specific rate limiting mechanism using a centralized data store (like Redis) to track request counts per domain per second. Each crawler, before making a request, 
checks this store to ensure the rate limit has not been exceeded. We'll use a sliding window algorithm to track the number of requests per domain per second. If the rate limit has been exceeded, 
the crawler will wait until the next second to make the request.
- A potential issue with this method is the risk of synchronized behavior among multiple crawlers. If several crawlers are waiting to make requests and simultaneously retry when the rate limit window resets, 
they'll all try and only one will succeed and the process will repeat.
- Fortunately, there is a relatively simple solution to this problem: jitter. By introducing a small amount of randomness to the rate-limiting algorithm, we can prevent synchronized behavior among crawlers. 
This jitter can be implemented by adding a random delay to each crawler's request, ensuring that they do not all retry at the same time ^KNIOdQMF

3. How to scale to 10B pages and efficiently ^XH7zthNC

crawl them in under 5 days?
 ^V3JpZM5t

* What about the parser workers? - Just scale up
* Dont forget about DNS
- You might wonder why we call it out explicitly, since tools like curl hadnle DNS automatically. The issue is scale. At thousands of requests
per second across millios of unique domains, DNS resolution becomes a real bottleneck. Each lookup for a new domain requires multiple network round trips through the DNS hierarchy.

If we are using a 3rd party DNS provider, we will want to make sure they can handle load. Most 3rd party providers have rate limits that can be increased by throwing money.
Optimisations:-
a. DNS caching = We can cache DNS lookups in our crawlers to reduce the number of DNS requests we need to make. This way all URLs to the same domain will reuse the same DNS lookup
b. Multiple DNS providers = We can use multiple DNS providers and round robin between them. This can help distribute the load across multiple providers and reduce the risk of hitting rate limits.

## Now lets focus more on efficiency
- To be efficient we need to ensure that we dont waste time crawling pages that have already been crawled.
- We can first check if a URL as already been crawled by checking the Metadata DB before putting it on the queue. If the URL already exists in our table, we skip it.
This is our first line of defense against reduntant work : URL-LEVEL-DEDUPLICATION

but URL level dedup is not enough. Different URL can serve identical content. For example http://example.com and http://www.example.com might return the same
page. Its also common for totally different domains to have exaclty the same content. For these cases, we cant just compare the URLs. Instead we need CONTENT-LEVEL-DEDUPLICATION. 
We hash the page content after fetching it and compare that hash to what we have seen before. If we've already stored this exact content, we skip the expensive parsing step

* Options to hash

-> Great solution: Hash and store in Metadata DB w/index

Approach: We could hash the content of the page and store this hash in our URL table in the Metadata DB. When we fetch a new URL, we hash the content and compare it to the hashes in the Metadata DB. 
If we find a match, we skip the page. To make sure the look up is fast, we need to build an index on the hash column in the Metadata DB. This would allow us to quickly look up the hash of the new URL 
and see if it already exists in the DB.
Challenges: While the index will become large and may slow down writes, this would be overly pessimistic. Modern databases are quite efficient at handling indexes, even large ones. While it's true that maintaining 
an index incurs overhead, this overhead is generally well-optimized in modern systems so it's safe to overlook this concern.

-> Great Solution: Bloom filter

Approach: Another possible approach is to use a Bloom filter. A Bloom filter is a probabilistic data structure that allows us to test whether an element is a member of a set. It can tell us definitively if an element 
is not in the set, but it can only tell us with some probability if an element is in the set. This is perfect for our use case. We can use a Bloom filter to store the hashes of the content of the pages we have already 
crawled. When we fetch a new URL, we hash the content and check the Bloom filter. If the hash is not in the Bloom filter, we know definitively that we haven't crawled this page before. If the hash is in the Bloom filter, 
we've probably crawled this page before and can skip it (with a small chance of false positive).
From a technology perspective, we can use Redis with the RedisBloom module (part of Redis Stack) to store the Bloom filter. RedisBloom provides native commands like BF.ADD and BF.EXISTS for this purpose. 
If using a Redis service without RedisBloom, you could implement a Bloom filter using Redis bit operations or store it in application memory.

Challenges:
The main challenge with a Bloom filter is that it can give false positives. This means that it might tell us that we have crawled a page when we actually haven't. We could argue that this is an acceptable trade-off for the 
performance benefits and can configure our bloom filter to reduce the probability of false positives by increasing the size of the filter and the number of hash functions used
 ^BGDjJuMy

Architecture of Web crawler ^S2H4KMCD

## Embedded Files
7091834f53c85ea28594dfaec5c4e4b97f521be4: [[Pasted Image 20260615233739_300.png]]

1e099e611c805666c597f684bfc7f7f867527063: [[Pasted Image 20260615233755_555.png]]

0b6a1f653f3437a5a3b26c60a893614469a75630: [[Pasted Image 20260615233902_804.png]]

ab301ba7dfb951f1462772d84a7ffad8c59348a4: [[Pasted Image 20260615233928_192.png]]

%%
## Drawing
```compressed-json
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

ADSqiVVBh0IkqZtTBkIshNAdwgPMIuhKdQgAA===
```
%%
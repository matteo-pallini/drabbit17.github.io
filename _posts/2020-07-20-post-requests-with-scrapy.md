---
layout: posts
comments: true
author_profile: true
title: Sending multiple POST requests from a spidered page using Scrapy
excerpt: A possible pattern for replicating programmatically AJAX POST requests when scraping webpages using Scrapy
date: 2020-07-20
---

Have you ever tried to scrape a data point whose value changed with changes in other options available on the webpage? In this article I am going to show a possible pattern for scraping all the values available when these are retrieved through AJAX requests. 

More specifically I found myself using this pattern while building [smarthiker.co.uk](https://smarthiker.co.uk/), a price comparison website for hiking, climbing and mountaineering products. While scraping products details I realised that the price for the same product may change for a different colour or size. The two pictures below (which are made up) should give you an idea of what I am talking about.


![brown_trousers](/assets/images/post_requests_with_scrapy/trousers_brown.png)


![blue_trousers](/assets/images/post_requests_with_scrapy/trousers_blue.png)


## Basic Scraping with Scrapy

Scrapy is a python library making available an easy to use framework for scraping websites. One of the main components of the library is the Spider class. Through this it is possible to specify from what URL to start spidering the website, how to parse the HTML pages retrieved, and possibly send other requests from them.

I will dare to claim that usually when scraping an e-commerce website the final goal is obtaining one object (an Item in the Scrapy framework) per product sold containing all the details of interest. Therefore, the final yield statement, the one not leading to further requests, should return all the scraped data for the single product considered.

Below you can find some python pseudo-code for a spider scraping the hiking e-commerce website [BananaFingers.co.uk](https://www.bananafingers.co.uk/). Using a [series of brand-specific pages](https://www.bananafingers.co.uk/brands), presenting each a paginated list of products, it should be possible to get all the products available on the website. The code below is a simplified version of the one used in production and it is only meant to make it easier to understand how to spider the website, not to actually spider it.

{% gist 52504d5a5712e1bd0a91f90b7abc883d %}

So, using a modified version of the snippet above you should be able to successfully scrape all the products sold on the website. However, only one price is scraped for each product, the one charged for the default colour-size combination loaded on the webpage. In order to get the specific colour-size combination prices you need to replicate the AJAX requests sent for each combination. When you pick a colour from the pick-list on the webpage an AJAX POST request is sent and the price value is updated on the page. So, to get the full picture we need to send as many calls as colour-size combinations available and somehow return their results with the original response for the product page.

## Figuring out the POST specifics

Using your preferred browser development tools It should be relatively easy to see the POST request specifics (target URL, payload, etc). As you can see in the picture below we can easily access the payload content and get a feeling of what we will need to change to get the colour-size prices. In this case, we need to specify the field colour, the size, and the product id.

![inspecting_network_calls](/assets/images/post_requests_with_scrapy/dev_tools_post_request.png)

The product-specific POST request payload keys and values (“attributes[field_waist_size]”: 1595, “attributes[field_color]”: 730), can be automatically scraped from the DOM. As you can see in the image below, the key for the colour/size field is available at the “name” attribute in the “select” HTML element. While the value can be accessed using the “value” attribute in the “option” HTML element.

![inspecting_dom_key_value_pairs](/assets/images/post_requests_with_scrapy/dev_tools_dom.png)

Once you have obtained the key/value pairs for all the options available it should be relatively straightforward to create a list of possible combinations and assemble the needed payloads for each combination.

Now we should have a list of key-value pairs that can be used as payloads for the different post requests we will send. Now we will send the requests and store the results somewhere so that they can be returned with the final item.

## Two possible approaches
I personally think that the most convenient way of getting all the colour-size prices for a product is sending the POST requests sequentially, carrying over the prices fetched for each combination, and once the list of combinations is exhausted return the item with the original product page HTML and the list of prices fetched. Alternatively, we could be yielding for each POST request an item with the original HTML and relative price returned, and then merge the all the items for a product further down the pipeline.

In the first case, for each POST request, we will wait for the response (and parse it) before sending the next one. While in the latter case the requests will be all sent roughly at the same time (eg through looping over the iterable containing the payloads) and each response will be parsed independently from the others whenever returned. The extra waiting time of the first approach will probably make it take longer than the first one when spidering the whole website, although I haven’t tested this.

## cb_kwargs (formerly known as meta) to the rescue!
If we want to go down the first route we can use the [cb_kwargs](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.cb_kwargs) argument in the Request to carry over all the product details to the response. So, we will be able to carry along the list of prices already obtained, the list of payloads we still need to send, and the HTML for the original page. The pseudo-code below should give you an idea of a possible pattern to do that.

{% gist 5262acd3a52265b68f7464a5ecfbc67b %}

As shown in the snippet above the parse_product_item is called as soon as the product page is reached by the spider. If this is the first time the method gets called for this page the response_params_for_post list won’t exist, so it will be created and filled with the list of payloads that should be sent for each POST request. Alternatively, if the list already exists, the next element will be popped and used as payload for the next POST request whose response will be parsed again through the parse_product_item method. This process will continue until the list of payloads for the POSTs has been exhausted. At that point, the original HTML body will be passed with all the fetched prices to a custom [ItemLoader](https://docs.scrapy.org/en/latest/topics/loaders.html), so that the values for the item will be extracted.

## Wrapping up...

To summarise, going through the steps below is possible to fetch the values loaded through multiple AJAX calls triggered by interactive components:
- Understand the details of the POST/GET requests using the product page data
- Create a list of payloads/parameters, containing one element per request you want to send
- Add one if-else statement in the parsing method, so that you can either create the list of payloads/parameters or parse the response coming from the previous POST/GET request
- Add another if-else statement in the parsing method, so that you will either send the next POST/GET request either yield the Item with the original HTML and all the values fetched

This is probably one of the most interesting scraping problems I worked on while building [smarthiker.co.uk](https://smarthiker.co.uk/). I was quite surprised that the same hiking/climbing/mountaineering product may be sold at a different price depending on colour or size. Still, whether this is surprising or not a robust price comparison website should be able to handle these situations as well.

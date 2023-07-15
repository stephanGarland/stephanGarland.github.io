---
title: 'Adventures With 811'
date: Mon, 20 May 2019 12:06:02 +0000
draft: false
tags: ['automation', 'gis', 'python']
---

For those of you unfamiliar with 811, it's a public service that attempts to prevent people from hitting buried utilities. Some utilities, like internet (at least on the to-the-home branches) are frequently trenched minimally at best, and with little to no protection. Electrical is _usually_ adequately buried, but a. you never know b. a big project like adding a pool, a deck, etc. will easily surpass the required depth for any utility. If you, the hapless homeowner, penetrate one of these utilities, you're probably on the hook for the cost of repairs, not to mention the ire of your neighbors, and of course, the threat of death (from the utility line, not your neighbors - hopefully).

"Well Stephan, that's terrific!" you say, "but surely there is a cost associated with this marvelous service." Nope, totally free. I don't mean free as in free at the point of use, either. Utilities pay for this service, as it's far better for them to send someone out to mark locations than to deal with the hassle of fixing buried utilities that you broke.

Now, obviously it doesn't make sense to just split the bill among every utility in the state, given the tightly-defined service territories they have. So, how do you figure out if a given address you had to locate is in a service territory? Easy, the utilities tell you. Or, they tell the state, anyway. At least some of the time. And it might be accurate. Truthfully, I don't know how every state does it - like so many other things in the United States, states do their own thing - but in Indiana, where I worked as a Distribution Engineer, it's run by [Indiana MAP](https://www.indianamap.org/index.php), which is a hodgepodge of government, private, and municipal partners.

If you play around [on the map for electric utilities](https://maps.indiana.edu/index.html?x=494668.6063081417&y=4507915.287779287&z=2&sBasemap=bm1&URLLayers=Infrastructure_Energy_Electric_Service_Territories), you can see who owns an area by clicking on it. For example, clicking in the vaguely-apricot-colored area surrounding the town of Monticello, you'll see Carroll White (still erroneously labeled White County REMC, which tells me they _still_ haven't submitted an update) Rural Electric Membership Cooperative's service territory. It's 587.990746 square miles, and has some inner areas not served - those are towns which have municipal power providers, who ironically often just outsource it back to the nearest REMC, sometimes with a markup, sometimes with a markdown that's more often than not due to the inability to perform cost analysis of a service.

So, you've got a service territory defined. Terrific. When 811 finishes up the month, they send each utility an invoice, detailing every service call they received that is in that utility's service territory, as defined by the above map, or whatever method that particular state is using. The utility pays it, and all is well. Unless, of course, you disagree with their assessment. You can contest this, but I can tell you from experience that they'll stand firm on the service territory delineation (after all, you were the one who told them via the state that you served that address), and you'll end up paying. But never mind that, let's figure out how we could figure out what we should be paying for, and what we should contest.

**Iteration 1:** When I joined CWREMC, they were using the most obvious method, which is a paper copy (you have to ask for them to send the invoices electronically, but more on that fun tangent later), and an extremely slow Java program with our service territory (like the above map, but with way more detail). Armed with these, you type in each address, and mark it Yes/No. This, as you might imagine, is rather time-consuming. On any given month, we had over 1,000 individual service calls to go through. An extremely dedicated co-worker and friend of mine had this unfortunate task, so she made the best of it. After watching her for a bit, I exported our service territory (hereafter referred to simply as territory) to a KML, downloaded Google Earth to test my theory, and then showed it to her. Boom, took a ~3 day task down to about ~1 day. But wait, there's more.

**Iteration 2**: Teach yourself Python. This step is rather important. I mean, pick any language, but I chose Python, since you can write half of whatever you're trying to do with enough import statements.

This problem, I recognized, was a point-in-polygon. All I needed to do was convert the locations in question to coordinates, and I could then check if they were in or out of our service territory. Easy, right? First struggle, the afore-mentioned record. I contacted 811, and requested an electronic copy. This was an alien request for them, but nonetheless, they eventually managed to send us... a PDF. A PDF that was clearly converted from Excel, mind you, which would have been perfect for my use. I inquired about getting an Excel, CSV, or really anything in a sane format, but was told they couldn't do that. Argh.

After struggling with various libraries and OCR apps, I found [Tabula](https://tabula.technology/). Tabula is great. No, Tabula is amazing. Through some form of magic, it perfectly extracts tables from PDFs, and presents them in CSV or Excel format. Did I mention it's FOSS? Hallelujah, great victory.

With that problem solved, I set about parsing the addresses. First, I tried regexes. I'm convinced regexes are never the answer, but that's never stopped me from trying. The issue is that the addresses were presented in a variety of formats, and often, they weren't addresses at all, but intersections of the nearest street (remember, this is a rural area). My first attempt was to classify them into two types:

*   `$HOUSE_NUM $STREET_NAME $TOWN $NEAREST_INTERSECTION`
*   `$INTERSECTING_STREET_1_NAME $TOWN $INTERSECTING_STREET_2_NAME`

This more or less worked, so I wrote a program that imported the CSV with a reader, and spat out a KML point for each row. The result was a rather cluttered map, but given that I could now give my co-worker two layers, one with our service area, and the other with hundreds of pins, it further reduced her time working from ~1 day to ~1 hour or so.

**Iteration 3:** At one point, I recall I had up to five regexes to try to classify the address, as some failed, and finally admitted defeat when I found an edge case that slipped through. Turning to libraries, I found a few that worked some of the time, but with no greater success than my regexes.

I then found [Geocodio](https://www.geocod.io/), which provides address parsing, normalization, reverse geocoding, Congressional districts, you name it - anything you want to know about an address or coordinates, even if it's slightly bungled - all for free, provided you stay under 2,500/day. Done.

This final iteration had two basic logic paths: first, it did an extremely rough check with a bounding box that encompassed our entire territory. If the point was in there, there was a pretty good chance we owned it, so pop a pin onto the map. If not, still pin it (in hindsight, this decision may not have been ideal), but also write to a log. Next, load the territory as a [GeoJSON](https://en.wikipedia.org/wiki/GeoJSON) file (laboriously obtained using [QGIS](https://www.qgis.org/en/site/) from the [Shapefile](https://doc.arcgis.com/en/arcgis-online/reference/shapefiles.htm) provided by the state), and do a point-in-polygon check. Log both positive and negative hits, but create a CSV that is then fed to Pandas (again... I'm not sure what my reasoning was for this extra step, but here we are) to output an Excel file containing only the negative hits. This process that originally took ~3 days now takes < 1 minute. Not bad.

Oh, finally, have your manager cancel the entire project and just pay the invoice a month after you get this working, because 811 threatened legal action if you didn't stop contesting charges. To be fair, if I recall our method of contesting charges was to refuse to pay any of the invoice, so they probably had a decent legal argument.

Things I learned from this project:

*   Python is wonderful, and you should use it.
*   ArcGIS, a commercial product, lacks (or at least obfuscates) certain features that QGIS, a FOSS product, quickly and easily provides.
*   The coordinate system you use can and will impact your geocoding, and you can't assume it's in WGS 84 format.
*   Someone has probably already tackled at least part of your problem, and solved it. Look before you re-invent the wheel.
*   Despite what to you seems a stunning achievement, your manager may not be impressed, and the entire project may be scrapped shortly after you complete it.
*   Minimum viable products are good. With each iteration, I made my customer (co-worker) extremely pleased with the reduction in her work, and she didn't have to wait for the entire project to be completed before seeing results.

Should you want to view the code, it's available on [GitHub](https://github.com/stephanGarland/811-geocoder), but I must caution you that it's not great. Please bear in mind that I was teaching myself Python (and really, to code - my previous coding experience, other than a brief foray into C++ as a child, consisted of an extremely angsty but W3C compliant HTML + CSS website) with this project, so it doesn't meet the single responsibility principle, uses some globals when they aren't needed, etc. Still, it works with Python 2/3, and the dependencies are listed in the README. I'll get around to making it better and/or containerizing it one of these days.
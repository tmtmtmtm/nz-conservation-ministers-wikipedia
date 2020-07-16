Note: This repo is largely a snapshop record of bring Wikidata
information in line with Wikipedia, rather than code specifically
deisgned to be reused.

The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Check the Position Item
===============================

The Wikidata item for the Minister of Conservation
  https://www.wikidata.org/wiki/Q6866068
didn't link to (and wasn't linked back from) that of the Department
  https://www.wikidata.org/wiki/Q1191417
so I added those links.

It did have sensible instance of + subclass of, country + jurisdiction,
and is marked as part of the Cabinet.

Step 2: Tracking page
=====================

Initial PositionHolderHistory list set up at https://www.wikidata.org/wiki/Talk:Q97446369

Current status: knows of 2 members

Step 3: Set up the metadata
===========================

I'm going to try experimenting with a slightly different setup now, where 
more is controlled from the JSON supplied in the [add_P39.js script](add_P39.js).

That knows the item ID we're dealing with, and the URL to be scraped (as
it sets that as the reference URL), so we should be able to extract
values from that to pass as arguments to the other scripts.

So the first step now is always to edit that file.

Step 4: Scrape
==============
Comparison/source = [Minister of Conservation (New Zealand)](https://en.wikipedia.org/wiki/Minister_of_Conservation_(New_Zealand))

I've adjusted the scraper to take the URL as an argument, rather than
being hardcoded into the script itself, so we now call this as:

    wb ee --dry add_P39.js  | jq -r '.claims.P39.references.P4656' |
      xargs bundle exec ruby scraper.rb | tee wikipedia.csv

Step 5: Get local copy of Wikidata information
==============================================

Again, we can now get the argument to this from the JSON, so call it as:

    wb ee --dry add_P39.js | jq -r '.claims.P39.value' |
      xargs wd sparql office-holders.js | tee wikidata.json


Step 6: Create missing P39s
===========================

    bundle exec ruby new-P39s.rb wikipedia.csv wikidata.json |
      wd ee --batch --summary "Add missing P39s, from $(wb ee --dry add_P39.js | jq -r '.claims.P39.references.P4656')"

-> https://editgroups.toolforge.org/b/wikibase-cli/aa2f97873c2b6/

Step 7: Add missing qualifiers
==============================

    bundle exec ruby new-qualifiers.rb wikipedia.csv wikidata.json |
      wd aq --batch --summary "Add missing qualifiers, from $(wb ee --dry add_P39.js | jq -r '.claims.P39.references.P4656')"

Only one statement to be added from this (https://tools.wmflabs.org/editgroups/b/wikibase-cli/355f673f9aff9/)

But we also have a warning about Q355398 having a mismatched start date.
Let's see what our report looks like before deciding what to do about
that.

Step 8: Refresh the Tracking Page
=================================

https://www.wikidata.org/w/index.php?title=Talk:Q6866068&oldid=1232665270

That's reporting an overlap based on the problem, so I'm going to fix
that up through the ridiculously convolted approach of grabing STDERR
rather than STDOUT, grepping out the lines I don't need, and the piping
the rest to `update-claim`

    bundle exec ruby new-qualifiers.rb wikipedia.csv wikidata.json 2>&1 >/dev/null | 
     fgrep -v \* | wd uq --batch --summary \
     "Update qualifiers based on $(wb ee --dry add_P39.js | jq -r '.claims.P39.references.P4656')"

which went through as https://tools.wmflabs.org/editgroups/b/wikibase-cli/700cc758b2cb7/

After the dust settles and the QueryService catches up, that gives a
final version as: https://www.wikidata.org/wiki/Talk:Q6866068




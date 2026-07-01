# Graphical display of acequia gauges in the NPT basin in northern New Mexico. 

Feature planning by Greg Swift. 
Coding by Anthropic Claude Opus 4.8
Version Beta 1 completed 6/25/2026, with revisions through early July after review by Sharon Dogruel and Chris Sheehan.

# Important detail: Launching the webpage must be done differently from my own PC vs from the Github site, to allow refresh of OSE data 

With automation, we can only access the html displays of each gauge's cfs data, which show only the most recent 4 days worth.  The web page needs to sample those sites periodically, adding the latest readings to the data already accumulated in the web page's folder to keep the graphs up to date.

However, a web page's JavaScript is sandboxed by the browser — it has no permission to execute a Python program to fetch data from OSE, or write files on your computer. That's a deliberate security wall (you wouldn't want any web page running programs on your machine), and there's no way around it from inside a plain HTML file.  The workarounds differ, Github vs your own PC:

On Github, fetch-gauges.yml takes care of this as a Github "scheduled Action."  Details in SETUP.md.  We try for every 30 minutes, but Github has a mind of its own, roughly every 6 hours.  When this all becomes part of a real website (e.g. part of PVID's website), the 30 minute refresh should work.

Testing the webpage locally on a pc, a small local helper is needed that has permission to run the fetcher. That helper is a tiny local server. So Claude made launchFromMyPC.py: you double-click it (or run python launchFromMyPC.py), and it pulls fresh data, opens the explorer html file from harddisk in your browser, and stays running to answer update requests.  Ctrl+C stops it.

Updating takes up to a minute (it's contacting 21 gauges), so be patient. If a gauge or two errors out, it still loads what came through and says so.

#  Mirror of CA.gov DMV's "Report of Traffic Accident Involving an Autonomous Vehicle (OL 316)"

### or, "How to semi-autonomously scrape the California DMV's autonomous vehicle accident reports"

### or, "*How to curl when wget gets blocked by JavaScript*"


Mirror page:

https://wgetsnaps.github.io/ca-dmv-autonomous-vehicle-accidents/

Original page:

https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316


This page is where California's DMV posts accident reports involving autonomous vehicles, such as the ones operated by Google/Waymo. These reports used to be self-published, but now we have to go to dmv.ca.gov to find them. As [Business Insider reports](http://www.businessinsider.com/waymo-ends-publishing-self-driving-car-accident-reports-website-2017-1):

> [Google/Waymo] removed the page of monthly reports detailing traffic collisions and other accidents on public roads that involve its self-driving vehicles. And Waymo will no longer publish the accident reports on its website, Business Insider has learned. 

> The page that once hosted all of the accident reports now redirects to Waymo's general website, which makes no mention of any accidents.


> ...Jessica Gonzalez, a spokesperson for the California DMV, told Business Insider that Waymo is not required to publish the monthly public reports. But the company is required to report any accident involving a Waymo self-driving vehicle to the DMV. Those accidents are published on the California DMV's website. 






## Web-scraping the manual way

**tl;dr**: California's DMV page is a **hot shit mess** that can't be used with a client that lacks Javascript, i.e. `wget` So I've written a workaround that has one point-and-click step, and then a bunch of fancy `bash`-ing with regexes to at least mirror the content of the page (i.e. the PDF reports)


So here's a manual workaround using `curl`, `sed`, `ack`, and a simple Bash loop. And Google Chrome's dev tools. 


### Saving a proper copy of the HTML


The CA DMV page is mostly a bunch of links to PDFS (which you can view in the [pdfs/](pdfs/) directory on this repo):

![image sample-site-screenshot.png](sample-site-screenshot.png)


However, visiting the [CA DMV's Autonomous Reports accidents page](https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316) in a non-JS enabled browser will get you something similar to what the [Internet Archive crawler gets](http://web.archive.org/web/20170425060918/https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316) when encountering CA.gov's un-robust Javascript:

![image archive-example.png](archive-example.png)

The dmv.ca.gov's Javascript, as far as I can tell, is used to set a cookie -- and check for headers, such as a `User-Agent` that is *not* `wget` -- that is validated server-side. Currently, `wget` doesn't execute Javascript, so that makes it fairly impossible to use `wget` alone to capture the dmv.ca.gov page.

So we require a *manual* step: visiting the CA.gov page using the Chrome web browser to get a `cURL` command that can be used to save the raw, rendered HTML.


1. Visit the page using Chrome and have the **Network Panel** activated
2. Copy the request for the `autonomousveh_ol316` page using the **Copy as cURL** feature:
  
   ![image copy-as-curl.png](copy-as-curl.png)

3. This should result in a shell command that looks something like this:

    ```sh
    curl 'https://www.dmv.ca.gov/portal/dmv/detail/vr/autonomous/autonomousveh_ol316' \
      -H 'Pragma: no-cache' -H 'DNT: 1' \
      -H 'Accept-Encoding: gzip, deflate, sdch, br' \
      -H 'Accept-Language: en-US,en;q=0.8' -H 'Upgrade-Insecure-Requests: 1' \ -H 'User-Agent: Mozilla' \
      -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8' \ 
      -H 'Cache-Control: no-cache' \
      -H 'Cookie: pieceofshitcookie' -H 'Connection: keep-alive' --compressed
    ```
4. Paste that `curl` into your Terminal and save the results as a file named [`original-index.html`](original-index.html)


### Extracting the PDFs

[original-index.html](original-index.html) contains the HTML as found on the real live site, which means that the URLs for the PDFs are relative and in this kind of format:

```html
<p dir="ltr">
  <a href="/portal/wcm/connect/7775ab52-ff6a-4473-8300-e3d589cd6448/GMCruise_052517.pdf?MOD=AJPERES" >
    GM Cruise May 25, 2017
  </a>
</p>
```

So we need to extract those relative URLs, translate them to absolute URLs, and download them to our computer. This can be done with a combination of regexes and a Bash `while` loop, with `curl`:


1. Make a directory called `pdfs`, e.g. `$ mkdir pdfs`
2. Use a grep-like program to match and extract the URLs from the `[original-index.html](original-index.html)` file. I like using [ack](https://beyondgrep.com/) because it not only provides PCRE-flavor regexes, but the use of capture groups to format output just as we need it (you could consider it a lazy person's `awk`):

    ```sh
    ack 'href="(.+?\.pdf)' original-index.html \
      --output '$1'
    ```

    Example output:

    ```
      /portal/wcm/connect/4fdef05c-ae0e-43c3-ac53-c96427ef40bd/GM+Cruise_052517_2.pdf
      /portal/wcm/connect/7775ab52-ff6a-4473-8300-e3d589cd6448/GMCruise_052517.pdf
      /portal/wcm/connect/40372935-f84b-402a-bd29-fd38924eccae/Google_041917.pdf
      /portal/wcm/connect/9ff287ca-2562-48f2-999d-be18e1714eba/Google_032617.pdf
      /portal/wcm/connect/c7b00b79-1bc3-46d5-bd96-606342046aa8/GMCruise_032317.pdf
      /portal/wcm/connect/562b91e1-e05f-4ba3-8d62-6794d0f6509e/GMCruise_032217.pdf
      /portal/wcm/connect/a549dbc4-a164-4813-bc72-3164261fc89a/GMCruise_021617.pdf
      /portal/wcm/connect/3d358211-3f0c-430e-b8db-b13392239e1e/Google_121116.pdf
      /portal/wcm/connect/4a39c1b9-ca1f-4184-8b78-96d6baf98628/Google_102616.pdf
    ```

3. Feed that `ack` command into `curl` via a loop; note that the `MOD=AJPERES` has to be included in the GET request, as well as a nominal user agent (`'Mozilla'` seems to work fine):


    ```sh
    $ ack 'href="(.+?\.pdf)' original-index.html \
      --output '$1' \
      | while read -r href; do  
        url="https://www.dmv.ca.gov${href}?MOD=AJPERES"
        fname=$(basename $href)
        curl -A 'Mozilla' $url -o pdfs/$fname
    done
    ```

That sequence will download all the PDFs from the original site into the local subdirectory of `pdfs`, i.e. this URL:

https://www.dmv.ca.gov/portal/wcm/connect/a35d0b74-02dc-4725-9a5f-cc4ac71e421b/Google_110215.pdf?MOD=AJPERES


Will be saved as:

[/pdfs/Google_110215.pdf](pdfs/Google_110215.pdf)


### Creating `index.html` with relative links

Now the [pdfs/](pdfs/) directory contains all of the PDFs in one place, locally, for easy browsing. This next step is optional; it simply creates a version of [original-index.html](original-index.html) in which the PDF links have been translated to point to `/pdfs` instead of `https://www.dmv.ca.gov/portal/wcm/connect/`.

The result is [index.html](index.html), which can be visited as hosted on Github Pages at:

https://wgetsnaps.github.io/ca-dmv-autonomous-vehicle-accidents/



```sh

$ sed  -e 's/?MOD=AJPERES//' \
       -e 's#/portal/wcm/connect/[a-z0-9\-]*/#./pdfs/#' \
       -e 's/<base href/<meta data-href/' \
       original-index.html > index.html
```

Short explanation of this `sed` command: it runs 3 substitution expressions:

1. Remove all instances of `?MOD=AJPERES`
2. Replace the `/portal/wcm/connect/...` paths with `./pdfs/`
3. Neuter the `<base>` tag which caused the browser to prepend the original `dmv.ca.gov` domain to relative URLs.







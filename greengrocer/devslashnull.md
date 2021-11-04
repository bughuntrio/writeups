The Greengrocer Market bughuntr.io scenario starts with a simple web application which appears to be a landing page for a small greengrocer shop. The URL provided for this scenario was https://ddosedlaptop.br2.bughuntr.net/ (each scenario run appears to use a random URL though).

The first thing I tried was using (ffuf)[https://github.com/ffuf/ffuf] with the `raft-medium-words.txt` wordlist against the URL:

```
$ ./ffuf -w /opt/wordlists/raft-medium-words.txt -u https://ddosedlaptop.br2.bughuntr.net/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1
________________________________________________

 :: Method           : GET
 :: URL              : https://ddosedlaptop.br2.bughuntr.net/FUZZ
 :: Wordlist         : FUZZ: /opt/wordlists/raft-medium-words.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

.                       [Status: 200, Size: 18716, Words: 1185, Lines: 385]
flag                    [Status: 403, Size: 205, Words: 20, Lines: 5]
server-status           [Status: 403, Size: 199, Words: 14, Lines: 8]
```

This found that the `/flag` and `/server-status` paths result in 403 errors, and very little else. It's safe to assume that the goal of this scenario is to get the contents of the `/flag` file somehow. Visiting the `/flag` URL in a browser returned the error:

> Forbidden
> The 'anonymous' user does not have the permission to access the requested resource.

As this did not provide any useful leads, I moved on. Loading up the URL in a browser through Burp Suite I could see there was not much functionality to this website, no internal or external links. There did appear to be some odd session handling though. Sessions were created by `POST`ing to the `/api/session/create` path and storing the JSON `id` response in a `session` cookie and the browser local storage. I played with this for a while but could not get anything note worthy to happen.

Whilst browsing the site I noticed an interesting HTTP response header:
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 04 Nov 2021 16:26:36 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 18716
Connection: keep-alive
Content-Disposition: inline; filename=index.html
Last-Modified: Tue, 12 Oct 2021 11:29:58 GMT
Cache-Control: no-cache
ETag: "1634038198.0-18716-742655413"
Onion-Location: http://aaepq3ietvknut7v2yvxspfvisgpigwd2ti44g3ew5zxgligvsh74ryd.onion/
```

Googling for the `Onion-Location` header lead to a support page from the Tor Project, https://support.torproject.org/onionservices/onion-location/. This states:

> Onion-Location is a new HTTP header that web sites can use to advertise their onion counterpart

In Tor parlance, an 'onion' site or hidden service, is a site that can only be accessed over the Tor network. So I loaded up the discovered URL in the (Tor Browser)[https://www.torproject.org/download/] and was presented with a whole different type of market:

**TODO**: Screenshot

It appears as though the Greengrocer Market was a front for a Darknet Market 😱 Here I found the first flag:

> Wellcome to the Greengrocer (Darknet) Market where you will find the freshest produce available for delivery direct to your door. Here is a flag for finding us '$flag{419df3c991618bce7f16cf41caa6d005aa715dea7353cd143bcabdf4aa518810}'.

*'Wellcome' typo is on the site, not one of my making!*

Whilst this website looks different, as before there did not seem to be much / any extra functionality. At this point, Googling for Tor hidden service vulnerabilities I discovered a promising looking tool (OnionScan)[https://onionscan.org/]. Unfortunately it appears as though this tool has not been actively developed since 2016, and does not support v3 onion addresses, which is what was being used by Greengrocer Market.

Reading about the (checks that OnionScan performs)[https://github.com/s-rah/onionscan/blob/master/doc/what-is-scanned-for.md], I saw something familiar:
> Seriously, don't even run the tool, go to your site and check if you have `/server-status` reachable. If you do, turn it off!

I found earlier that `/server-status` responded with a 403 error being accessed via the regular clearnet URL, but from the onion service over Tor it give a '200 OK' response! Apache mod_status, the Apache module behind the `/server-status` path, lists among other information URLs which have recently been visited. In this case, the output included the following interesting entries:
```
0-0	11	0/1069/1069	_ 	20.30	186	1	1446	0.0	0.79	0.79 	10.8.0.1	http/1.1	10.8.0.2:80	GET /api/session/29703cdf8eb7dbbccf006cca0794467a HTTP/1.0
0-0	11	0/1156/1156	_ 	20.30	44	1	1528	0.0	0.81	0.81 	10.8.0.1	http/1.1	10.8.0.2:80	POST /api/session/create HTTP/1.0
```

These requests appeared to be related to the session handling I noted at the beginning, but for a user other than myself. Visiting `/api/session/29703cdf8eb7dbbccf006cca0794467a` gave a JSON response:
```json
{
  "id": "29703cdf8eb7dbbccf006cca0794467a",
  "user": "admin",
  "connection": "tor-exitnode",
  "ip": "199.249.230.163",
  "flag": "$flag{105644e4ac72dabaad890b69b92f9ea8bb781df6106ae075cf771ad876506c83}"
}
```

This included another flag (two down so far) and a session id for the 'admin' user! I set this value in the `session` cookie, and attempted to access the `/flag` path again:

> Forbidden
> The client IP address does not match the IP address of the 'admin' session.

That's a new error at least. Looking back at the `/api/session/29703cdf8eb7dbbccf006cca0794467a` response, I could see that there is an IP value of `199.249.230.163`. It would appear that the session was tied to that IP address. An IP whois lookup for that address gave an odd looking response:

```
OrgName:        Quintex Alliance Consulting
OrgId:          QAC-4-Z
Address:        6732 Goodland Loop
City:           San Angelo
StateProv:      TX
PostalCode:     76901
Country:        US
RegDate:        2017-03-22
Updated:        2017-12-01
```

I got stuck here for quite a while until I reviewed the API response again and noticed the `"connection": "tor-exitnode"` entry. Since this scenario used Tor quite extensively, I did another IP address lookup, but this time on the Tor ('ExoneraTor' service)[https://metrics.torproject.org/exonerator.html]. This indicated that the IP address was associated with a Tor Exit Relay. So now all I had to do was use the same Tor exit relay to access the `/flag` path. I spent quite a while refreshing Tor circuits to try and get a Tor circuit with the same exit relay, but didn't have any luck. Reading through the Tor documentation (reading the documentation is always my last resort 🤦‍♂️), I noticed that you can set the Exit Relay in the `torrc` configuration. This can be done in Tor Browser by adding the following configuration in the `./Browser/TorBrowser/Data/Tor/torrc` file, and restarting Tor Browser:
```
ExitNodes 199.249.230.163
```

Checking the new IP address with https://www.whatsmyip.org/ showed `199.249.230.163`. Fantastic. I then set the admin session id cookie and requested the `/flag` (from the clearnet https://ddosedlaptop.br2.bughuntr.net/flag URL though, so our Exit Node IP address is applied), this resulted in a '200 OK' with the 3rd and final flag:
```
{"flag": "$flag{501c9639c311b73378df3e6732bc9c5c658fb496a806d0d142c86105c3838b2a}"}
```

That wraps up this BugHuntr.io challenge.

**Important Note:** don't forget to remove the `ExitNodes` configuration from Tor Browser. It could seriously compromise any future anonymous browsing leaving this setting in place.

# USPS Tracking API after the Web Tools shutdown

USPS turned off the old Web Tools API on January 25, 2026. If you had anything calling `secure.shippingapis.com` (TrackV2, RateV4, Verify...), it stopped working that day.

I needed USPS tracking working again, so I spent some time looking at the options. These are my notes, plus code you can copy.

- [What changed](#what-changed)
- [The official USPS APIs](#the-official-usps-apis)
- [What I ended up using](#what-i-ended-up-using)
- [Code examples](#code-examples)
- [My test results](#my-test-results)
- [USPS tracking API comparison](#usps-tracking-api-comparison)
- [FAQ](#faq)

## What changed

Web Tools was XML over HTTP with a USERID in the query string. Old, but it worked for years.

The replacement is the new USPS APIs platform on the USPS Developer Portal (`developers.usps.com`). It's REST/JSON with OAuth 2.0:

| Web Tools (retired) | New USPS APIs |
|---|---|
| XML | JSON |
| USERID in the URL | OAuth 2.0 client credentials |
| TrackV2 / TrackFieldRequest | Tracking API 3.x |
| Verify / ZipCodeLookup | Addresses API 3.0 |
| RateV4 / IntlRateV2 | Prices API |

USPS has spreadsheets mapping the old fields to the new ones on the Web Tools retirement page.

If you're not sure whether you're affected, these are the usual signs:

- calls to `secure.shippingapis.com/ShippingAPI.dll?API=TrackV2` return errors
- your WooCommerce, Magento or custom shipping plugin stopped showing USPS tracking or rates
- tracking on your order pages is stuck on an event from late January 2026

## The official USPS APIs

If you need labels, prices, pickups and the rest, migrating to the official APIs is the way to go. The steps are roughly:

1. Create a USPS Developer Portal account and register an app
2. Implement the OAuth token flow (get, cache, refresh)
3. Rewrite the XML parsing for the new JSON schemas
4. Map the old fields using the USPS spreadsheets
5. Ask for access and higher limits for the APIs you need

Step 5 is the catch. The default product allows 60 calls per hour per API, according to the USPS migration FAQ. For anything more you have to contact USPS API Support. And since April 2026 there's also API Access Control on tracking, which adds authorization requirements for platforms tracking packages for other merchants.

For tracking only, that was too much work for what I needed, so I went looking for something faster.

## What I ended up using

I tried a few third-party tracking APIs. The one that performed best was [Wonca's USPS Tracking API](https://www.wonca.com.br/api/usps?utm_source=github&utm_medium=readme&utm_campaign=usps_api).

What sold me on it:

- success rate above 99% in my tests
- about 3 seconds per lookup on average, with the full event history
- one API key in a header. No OAuth, no token refresh, no USPS developer account
- webhooks, so new USPS events get pushed to you instead of you polling
- batch lookups for many tracking numbers in one request
- 1,000 free lookups to try it out
- paid plans from $29.70/month for 12,000 lookups (Black Friday price, until November 30, 2026; normally $99/month)

The same key also works for Correios, FedEx, UPS, CAINIAO, J&T, DPD, Evri, GLS, La Poste and other carriers, which was a plus for me.

Worth knowing: it's a third-party service, not the official USPS API, and it isn't affiliated with USPS.

## Code examples

Same endpoint in every example. Swap in your API key and tracking number.

### cURL

```bash
curl -X POST \
  https://api.wonca.com.br/wonca.labs.v1.LabsService/Track \
  -H "Authorization: Apikey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "code": "9589071052702634870131", "carrier": "CARRIER_USPS" }'
```

### Node.js

```javascript
// Node 18+
const res = await fetch(
  "https://api.wonca.com.br/wonca.labs.v1.LabsService/Track",
  {
    method: "POST",
    headers: {
      Authorization: `Apikey ${process.env.WONCA_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      code: "9589071052702634870131",
      carrier: "CARRIER_USPS",
    }),
  }
);

console.log(await res.json());
```

### Python

```python
import os
import requests

resp = requests.post(
    "https://api.wonca.com.br/wonca.labs.v1.LabsService/Track",
    headers={
        "Authorization": f"Apikey {os.environ['WONCA_API_KEY']}",
        "Content-Type": "application/json",
    },
    json={"code": "9589071052702634870131", "carrier": "CARRIER_USPS"},
    timeout=30,
)
resp.raise_for_status()
print(resp.json())
```

### PHP

```php
<?php
$ch = curl_init("https://api.wonca.com.br/wonca.labs.v1.LabsService/Track");
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        "Authorization: Apikey " . getenv("WONCA_API_KEY"),
        "Content-Type: application/json",
    ],
    CURLOPT_POSTFIELDS     => json_encode([
        "code"    => "9589071052702634870131",
        "carrier" => "CARRIER_USPS",
    ]),
]);
echo curl_exec($ch);
curl_close($ch);
```

### Go

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
)

func main() {
	body := []byte(`{"code":"9589071052702634870131","carrier":"CARRIER_USPS"}`)
	req, _ := http.NewRequest("POST",
		"https://api.wonca.com.br/wonca.labs.v1.LabsService/Track",
		bytes.NewBuffer(body))
	req.Header.Set("Authorization", "Apikey "+os.Getenv("WONCA_API_KEY"))
	req.Header.Set("Content-Type", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	out, _ := io.ReadAll(resp.Body)
	fmt.Println(string(out))
}
```

### C#

```csharp
using System.Net.Http;
using System.Text;

var client = new HttpClient();
var req = new HttpRequestMessage(HttpMethod.Post,
    "https://api.wonca.com.br/wonca.labs.v1.LabsService/Track");
req.Headers.Add("Authorization",
    $"Apikey {Environment.GetEnvironmentVariable("WONCA_API_KEY")}");
req.Content = new StringContent(
    "{\"code\":\"9589071052702634870131\",\"carrier\":\"CARRIER_USPS\"}",
    Encoding.UTF8, "application/json");

var res = await client.SendAsync(req);
Console.WriteLine(await res.Content.ReadAsStringAsync());
```

## My test results

Tested with real USPS tracking numbers:

| | Wonca USPS Tracking API |
|---|---|
| Success rate | above 99% |
| Average response time | ~3 s per lookup |
| Event history | full (date, location, USPS description) |
| Auth | single API key |

Of everything I tried, this was the best mix of success rate and speed.

## USPS tracking API comparison

Every provider charges in a different way (per month, per year, prepaid, per tracker), so I converted each entry plan to cost per 1,000. Public pricing as of November 2026. Prices change often, so double-check before you decide.

| Provider | Entry plan | What you get | Cost per 1,000 | Webhooks |
|---|---|---|---|---|
| [Wonca](https://www.wonca.com.br/api/usps?utm_source=github&utm_medium=readme&utm_campaign=usps_api) (Black Friday) | $29.70/month | 12,000 lookups/month | **$2.48** | yes |
| Ship24 (per-call API) | $399/month | 100,000 API calls/month | $3.99 | yes |
| [Wonca](https://www.wonca.com.br/api/usps?utm_source=github&utm_medium=readme&utm_campaign=usps_api) (regular) | $99/month | 12,000 lookups/month | **$8.25** | yes |
| 17TRACK API | $119 prepaid, valid 12 months | 5,000 tracking numbers | $23.80 | yes |
| EasyPost | $0.03 per USPS tracker | pay as you go | $30.00 | yes |
| AfterShip | $29/month | 6,000 shipments/year (500/month) | $58.00 | yes |
| TrackingMore | $11/month | 150 shipments/month | $73.33 | yes |
| USPS APIs (official) | free | 60 calls/hour per API | $0, but rate-limited | no |

Not all units are the same. Wonca and Ship24 count lookups or API calls. AfterShip, TrackingMore, 17TRACK and EasyPost count one tracking number, however many times it updates. A per-lookup price and a per-shipment price only line up if you check each package about once, so keep that in mind for your own volume.

A few things I noticed:

The official API costs nothing, but 60 calls an hour won't keep up with a live order-status page.

Wonca came out cheapest per 1,000 at the Black Friday price. At the regular price, only Ship24's per-call plan is cheaper, and that one starts at $399/month.

17TRACK is paid upfront for the year. The others bill monthly or per use.

## FAQ

**Is the USPS Web Tools API still working?**
No. It was retired on January 25, 2026.

**What replaced it?**
The USPS APIs on the USPS Developer Portal: REST/JSON under `apis.usps.com`, with OAuth 2.0.

**What is the USPS API rate limit?**
60 calls per hour per API by default. You can ask USPS API Support for more.

**Is there a USPS tracking API without OAuth?**
Not from USPS. Third-party services like Wonca just use an API key.

**Can I track USPS packages in bulk?**
Yes. Wonca supports batch lookups and webhooks.

**Is there a free USPS tracking API?**
The official one is free, with the rate limit above. Wonca has 1,000 free lookups for testing.

## Links

- [Wonca USPS Tracking API](https://www.wonca.com.br/api/usps?utm_source=github&utm_medium=readme&utm_campaign=usps_api)
- [USPS Web Tools retirement page](https://www.usps.com/business/web-tools-apis/)
- [USPS Developer Portal](https://developers.usps.com)

## License

MIT

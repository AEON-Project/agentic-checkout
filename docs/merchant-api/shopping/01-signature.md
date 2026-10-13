# Signature (V3)

> As of 2026-10 the signing protocol has been upgraded to **V3** as a hard cutover: the legacy scheme (plain `k=v&…&key=secret` concatenation with no timestamp/nonce and no request-path binding) is **no longer accepted**. What changed in V3: the string to sign is bound to the HTTP method and request path (prevents cross-endpoint replay), every request must carry `timestamp` (±5-minute window) and `nonce` (server-side replay protection — unique within the window), and keys/values are RFC 3986 percent-encoded (eliminates ambiguity when values contain `&` / `=`).

*   Currently `appId` and `secret` are obtained offline or via the merchant management platform;
*   secret: the secret used to digest and sign the input parameters of API requests. The merchant keeps it and must strictly guarantee the security of this key; it must not be leaked.

##### Common Parameters (carried by every request, at the top level of the request body JSON, same level as business parameters; in the query string for GET requests)

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier, assigned when onboarding to the platform |
| userId | Yes | string | User identifier (email); funds belong to this user — this user's wallet / AI Card bears the charge, and the order belongs to this user; for a new user, call "Initialize Account" first |
| timestamp | Yes | string | Signing timestamp, **standard epoch milliseconds** (UTC, no timezone offset); the server enforces a ±5-minute window, out-of-window requests get `91024` |
| nonce | Yes | string | Random anti-replay string, ≥8 characters (≥16 random hex recommended); must not repeat within the window for the same appId, duplicates get `91024` |
| sign | Yes | string | SHA-512 signature (uppercase hex) generated per the rules in this document; does not itself participate in signing |

##### V3 String-to-Sign Structure

```
METHOD + "\n" + PATH + "\n" + parameter section + "\n" + "key=" + secret
```

*   **METHOD**: the HTTP method in uppercase (`GET` / `POST`…);
*   **PATH**: the full request path, without host and query string, e.g. `/agentapi/v1/orders/create`;
*   **parameter section**: all parameters with non-empty values (query string and request-body top level merged, including timestamp/nonce, excluding `sign`), sorted in ascending dictionary (ASCII) order of parameter names, each pair rendered as `encode(k)=encode(v)` and joined with `&`;
*   **encode**: RFC 3986 percent-encoding — encode UTF-8 bytes one by one; unreserved characters (`A-Z a-z 0-9 - . _ ~`) stay as-is, everything else becomes `%XX` (uppercase hex). **Do not use your language's form-encoding** (a space must be `%20`, never `+`);
*   **object fields**: first build the sub-object's own substring with the same rule (dictionary order + `encode(k)=encode(v)` joined with `&`, **no `key=` appended**), then **encode that whole substring once more and use it as the field's value** in the outer concatenation (i.e. the substring is double-encoded — see the TRAVALA example below);
*   empty values (null / empty string), `sign` and `key` fields are removed (also inside sub-objects);
*   digest: SHA-512 over the string to sign, hex result converted to **uppercase**.

##### Signature String Example

*   Given the parameters (create order `POST /agentapi/v1/orders/create`, SHOPIFY, secret assumed to be `9999`):

		{
		  "appId": "TEST000001",
		  "userId": "user@example.com",
		  "channel": "SHOPIFY",
		  "variantId": "gid://shopify/ProductVariant/4409",
		  "shopDomain": "acmegear.com",
		  "country": "US",
		  "timestamp": "1700000000000",
		  "nonce": "f3a9c2d14b7e86a1",
		  "sign": "..."
		}

*   The string to sign (note the RFC 3986 encoding: `@` → `%40`, `://` → `%3A%2F%2F`; line breaks are `\n`):

		POST
		/agentapi/v1/orders/create
		appId=TEST000001&channel=SHOPIFY&country=US&nonce=f3a9c2d14b7e86a1&shopDomain=acmegear.com&timestamp=1700000000000&userId=user%40example.com&variantId=gid%3A%2F%2Fshopify%2FProductVariant%2F4409
		key=9999

*   SHA-512 digest converted to uppercase gives the final signature:

		20BFC0CC4F9407AF7B9691DEB19CF636174A5DF08798309B0C2B68DB125F4AAAA0967BF36C6302AA5151D33F0409940C642CFE02D7A84FEE1248965CCBEC8CB5

*   Endpoints with no business parameters work the same way, e.g. `POST /agentapi/v1/wallet/assets` (the parameter section contains only the four common parameters):

		POST
		/agentapi/v1/wallet/assets
		appId=TEST000001&nonce=8c41be0f29d7a356&timestamp=1700000000000&userId=user%40example.com
		key=9999
		→ sign = 9C0D1CE50D2448CA908AC978E60F01B630A8B3258591B4FF8A16C03E7A54D73E56B6C114E320097BD775B5B1FAD641A4088F36FACBD115C7843764171A142FF6

##### Signature String Example with an Object Parameter (TRAVALA create order, `guest`)

*   Given the request parameters (secret still assumed to be `9999`):

		{
		  "appId": "TEST000001",
		  "userId": "user@example.com",
		  "channel": "TRAVALA",
		  "packageId": "PKG_abc123",
		  "sessionId": "sess_9f2c",
		  "guest": { "firstName": "Waters", "lastName": "Alexander", "phone": "+8529319534025" },
		  "timestamp": "1700000000000",
		  "nonce": "1d5e8a37c6f4b290",
		  "sign": "..."
		}

*   Step 1: build the sub-object `guest`'s **internal** substring in dictionary order with encoded pairs (**no trailing `&`, no `key=` appended**; the `+` in the phone number is encoded as `%2B`):

		firstName=Waters&lastName=Alexander&phone=%2B8529319534025

*   Step 2: **encode that whole substring once more and use it as the value of the `guest` field** in the outer concatenation (`=` → `%3D`, `&` → `%26`, and the already-encoded `%2B` becomes `%252B`), producing the string to sign:

		POST
		/agentapi/v1/orders/create
		appId=TEST000001&channel=TRAVALA&guest=firstName%3DWaters%26lastName%3DAlexander%26phone%3D%252B8529319534025&nonce=1d5e8a37c6f4b290&packageId=PKG_abc123&sessionId=sess_9f2c&timestamp=1700000000000&userId=user%40example.com
		key=9999
		→ sign = B5F53ED8609C7C8D7EEB488B19CC7ADEC91C1E342359FDAA121CA16BCCCD1F08DAEAED710F3BA139044C70E361C386093BDDE2FD3E241CD5DC37A9B22EF1D044

*   Common mistakes (each of them results in `91024`):
    1. Missing timestamp/nonce, or timestamp not in epoch milliseconds (e.g. seconds, or local time with a timezone offset);
    2. String to sign missing the METHOD/PATH prefix, or PATH carrying the host / query string;
    3. Using form-encoding instead of RFC 3986 (space became `+`, or `~` got encoded);
    4. Concatenating the sub-object as JSON text (`guest={"firstName":...}`), or not double-encoding the whole substring;
    5. Flattening the sub-object's fields into the outer sort — sorting happens strictly within each level;
    6. Not removing empty/null fields inside the sub-object — the removal rules (empty values, `sign`, `key`) apply inside sub-objects as well;
    7. Reusing a nonce within 10 minutes (fixed value, second-rounded values, etc.) — every request needs a fresh random value.

##### Java Reference Implementation

		import java.nio.charset.StandardCharsets;
		import java.security.MessageDigest;
		import java.util.Map;
		import java.util.SortedMap;
		import java.util.TreeMap;

		public class SignV3 {

		    /** V3 signature: uppercase hex */
		    public static String sign(String method, String path, SortedMap<String, Object> params, String secret) throws Exception {
		        String raw = method.toUpperCase() + "\n" + path + "\n" + canonical(params) + "\nkey=" + secret;
		        MessageDigest md = MessageDigest.getInstance("SHA-512");
		        byte[] digest = md.digest(raw.trim().getBytes(StandardCharsets.UTF_8));
		        StringBuilder out = new StringBuilder();
		        for (byte b : digest) {
		            out.append(String.format("%02X", b));
		        }
		        return out.toString();
		    }

		    /** parameter section: dictionary order + rfc3986-encoded k=v joined with &; nested Maps recurse, the substring is re-encoded as the value */
		    static String canonical(SortedMap<String, Object> params) {
		        StringBuilder sb = new StringBuilder();
		        for (Map.Entry<String, Object> e : params.entrySet()) {
		            String k = e.getKey();
		            Object raw = e.getValue();
		            String v;
		            if (raw instanceof Map) {
		                SortedMap<String, Object> sorted = new TreeMap<>();
		                ((Map<?, ?>) raw).forEach((nk, nv) -> sorted.put(String.valueOf(nk), nv));
		                v = canonical(sorted);
		            } else {
		                v = raw == null ? null : raw.toString();
		            }
		            if (v == null || v.isEmpty() || "null".equals(v) || "sign".equals(k) || "key".equals(k)) {
		                continue;
		            }
		            if (sb.length() > 0) {
		                sb.append('&');
		            }
		            sb.append(encode(k)).append('=').append(encode(v));
		        }
		        return sb.toString();
		    }

		    /** RFC 3986: UTF-8 bytes one by one; unreserved characters stay, everything else %XX uppercase */
		    static String encode(String s) {
		        byte[] bytes = s.getBytes(StandardCharsets.UTF_8);
		        StringBuilder out = new StringBuilder(bytes.length * 3);
		        for (byte b : bytes) {
		            int c = b & 0xff;
		            if ((c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9')
		                    || c == '-' || c == '.' || c == '_' || c == '~') {
		                out.append((char) c);
		            } else {
		                out.append('%').append(String.format("%02X", c));
		            }
		        }
		        return out.toString();
		    }

		    public static void main(String[] args) throws Exception {
		        SortedMap<String, Object> params = new TreeMap<>();
		        params.put("appId", "TEST000001");
		        params.put("userId", "user@example.com");
		        params.put("timestamp", String.valueOf(System.currentTimeMillis()));
		        params.put("nonce", java.util.UUID.randomUUID().toString().replace("-", ""));
		        System.out.println(sign("POST", "/agentapi/v1/wallet/assets", params, "9999"));
		    }
		}

##### Python Reference Implementation

		import hashlib
		import secrets
		import time
		from urllib.parse import quote

		def rfc3986(s: str) -> str:
		    return quote(s, safe="-._~")

		def canonical(params: dict) -> str:
		    parts = []
		    for k in sorted(params.keys()):
		        v = params[k]
		        if isinstance(v, dict):
		            v = canonical(v)
		        elif isinstance(v, bool):
		            v = "true" if v else "false"   # Java Boolean.toString() semantics
		        elif v is not None:
		            v = str(v)
		        if v is None or v == "" or v == "null" or k in ("sign", "key"):
		            continue
		        parts.append(f"{rfc3986(k)}={rfc3986(v)}")
		    return "&".join(parts)

		def sign_v3(method: str, path: str, params: dict, secret: str) -> str:
		    raw = f"{method.upper()}\n{path}\n{canonical(params)}\nkey={secret}"
		    return hashlib.sha512(raw.strip().encode("utf-8")).hexdigest().upper()

		body = {
		    "appId": "TEST000001",
		    "userId": "user@example.com",
		    "timestamp": str(int(time.time() * 1000)),
		    "nonce": secrets.token_hex(16),
		}
		body["sign"] = sign_v3("POST", "/agentapi/v1/wallet/assets", body, "9999")

##### Authentication Errors

| Scenario | code | Action |
|---|---|---|
| Missing common parameters | `91001` | Supply appId / userId / timestamp / nonce / sign |
| `appId` invalid or not enabled | `91003` | Verify onboarding info or contact operations |
| Signature verification failed / timestamp out of window / nonce reused | `91024` | Check the string to sign (METHOD/PATH prefix, dictionary order, RFC 3986 encoding, empty-value removal, object recursion, uppercase), your server clock (NTP), and nonce randomness |
| Transient server error | `91002` | Retry the original request once with a fresh nonce (the old nonce is already registered and cannot be reused) |

Always call over HTTPS; if the secret is leaked, contact operations immediately to rotate it.

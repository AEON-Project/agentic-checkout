# Signature

*   Currently `appId` and `secret` are obtained offline or via the merchant management platform;
*   secret: the secret used to digest and sign the input parameters of API requests. The merchant keeps it and must strictly guarantee the security of this key; it must not be leaked;
*   Building the string to sign: put all parameters subject to signature verification into an array and sort them in ascending dictionary (ASCII) order of the parameter names — **note that only the parameter keys are sorted; values do not participate in sorting**;
*   All non-empty field values participate in the signature (excluding sign); if a field is an object, the string must be concatenated recursively (the sub-object's fields are concatenated into a `k=v&k=v` substring in the same dictionary-order rule, and that substring participates in the outer concatenation as the field's value);
*   Common parameters `appId` / `userId` / `sign` go at the top level of the request body JSON (at the same level as business parameters).

##### Common Parameters (carried by every request)

| Parameter | Required | Type | Description |
|---|---|---|---|
| appId | Yes | string | Merchant identifier, assigned when onboarding to the platform |
| userId | Yes | string | User identifier (email); funds belong to this user — this user's wallet / AI Card bears the charge, and the order belongs to this user; for a new user, call "Initialize Account" first |
| sign | Yes | string | SHA-512 signature (uppercase) generated per the rules in this document; does not participate in signature verification |

##### Signature String Example:

*   Given the parameters (using create order `POST /orders/create`, SHOPIFY, with the secret assumed to be `9999`):

		{
		  "appId": "TEST000001",
		  "userId": "user@example.com",
		  "channel": "SHOPIFY",
		  "variantId": "gid://shopify/ProductVariant/4409...",
		  "shopDomain": "acmegear.com",
		  "country": "US",
		  "sign": "..."
		}

*   Concatenate the string in preparation for signing: convert to the form parameterName=parameterValue, join with the & symbol, and finally append key(secret). Note: (if a field is an object, the string must be concatenated recursively). The result is as follows:

		appId=TEST000001&channel=SHOPIFY&country=US&shopDomain=acmegear.com&userId=user@example.com&variantId=gid://shopify/ProductVariant/4409...&key=9999

*   Then digest the string to sign with the SHA-512 algorithm and convert the hex result to uppercase to produce the final signature. The signature result is as follows:

		1AF02D42CA64D20DF9973417F8834D9E876CA07C26406F6F8D41ABF3C3987198DB90B14A50002EEAFC1663CE325B7DF407D208A6EDA5277CD2AAEAEAC062EA22

*   Endpoints with no business parameters work the same way, e.g. `POST /wallet/assets` (the string to sign contains only the two common parameters):

		appId=TEST000001&userId=user@example.com&key=9999
		→ sign = 540AAABF9343EFA3C04A3590889CFF2B7B4BA802169E5DCB53ECCF983C84FD6B55A84CD07627F198ACAC036170B4A91AA72E8594C42CAD47003E9A66A20589B4

##### Signature String Example with an Object Parameter (TRAVALA create order, `guest`):

*   Given the request parameters (secret still assumed to be `9999`):

		{
		  "appId": "TEST000001",
		  "userId": "user@example.com",
		  "channel": "TRAVALA",
		  "packageId": "PKG_abc123",
		  "sessionId": "sess_9f2c...",
		  "guest": { "firstName": "Waters", "lastName": "Alexander", "phone": "+8529319534025" },
		  "sign": "..."
		}

*   Step 1: concatenate the sub-object `guest` **internally** in dictionary order into a `k=v&k=v` substring (**no trailing `&`, and no `key=` appended to the substring**):

		firstName=Waters&lastName=Alexander&phone=+8529319534025

*   Step 2: embed that substring **verbatim as the value of the `guest` field** in the outer concatenation (in the outer sort, `guest` participates only by its own field name; the sub-object's fields are never mixed into the outer sort), then append `key=secret` at the very end:

		appId=TEST000001&channel=TRAVALA&guest=firstName=Waters&lastName=Alexander&phone=+8529319534025&packageId=PKG_abc123&sessionId=sess_9f2c...&userId=user@example.com&key=9999
		→ sign = AEB89527AB2B186214B393336A42B7F25AAFE65D4A29CB7A260EA5CCF7F92C159D56EBCD0CC194D63E80775F3FE7F5E160F551B3D4DD3BE33FBE7CBDFED7BE6B

*   Common mistakes (each of them results in `91024`):
    1. Concatenating the sub-object as JSON text (`guest={"firstName":...}`) — wrong; it must be flattened into a `k=v&k=v` substring;
    2. Flattening the sub-object's fields into the outer sort (e.g. `firstName=...&guest.lastName=...`) — wrong; sorting happens strictly within each level;
    3. Adding a trailing `&` to the substring or appending `key=` inside it — `key=secret` is appended exactly once, at the very end of the outermost string;
    4. URL-encoding or escaping values — wrong; all values are concatenated verbatim (including characters such as `+`, `@`, `&`, `=` — see the `+` in the phone number above);
    5. Not removing empty/null fields inside the sub-object — the removal rules (empty values, `sign`, `key`) apply inside sub-objects as well.

##### Signing Utility Class:

		import java.io.UnsupportedEncodingException;
		import java.security.MessageDigest;
		import java.security.NoSuchAlgorithmException;
		import java.util.Iterator;
		import java.util.Map;
		import java.util.Set;
		import java.util.SortedMap;
		import java.util.TreeMap;

		public class SHA512Utils {

		    public static final String ENCODE = "UTF-8";

		    /**
		     * Sign the message with SHA-512
		     *
		     * @param signParams signature parameters
		     * @param key        secret key
		     * @return
		     */
		    public static String SHAEncrypt(SortedMap<Object, Object> signParams, String key) {
		        String signStr = stringConcatenation(signParams, key);
		        return encrypt(signStr, ENCODE).toUpperCase();
		    }

		    public static String stringConcatenation(SortedMap<Object, Object> signParams, String key) {
		        StringBuffer sb = new StringBuffer();
		        Set es = signParams.entrySet();
		        Iterator it = es.iterator();
		        while (it.hasNext()) {
		            Map.Entry entry = (Map.Entry) it.next();
		            String k = (String) entry.getKey();
		            String v = null;
		            if (entry.getValue() instanceof Map) {
		                SortedMap<Object, Object> sortedMap = new TreeMap();
		                sortedMap.putAll((Map<?, ?>) entry.getValue());
		                v = stringConcatenation(sortedMap, null);
		            } else {
		                if (entry.getValue() != null) {
		                    v = entry.getValue().toString();
		                }
		            }
		            if (null != v && !"".equals(v) && !v.equals("null") && !"sign".equals(k) && !"key".equals(k)) {
		                sb.append(k + "=" + v + "&");
		            }
		        }
		        if (key != null && !key.isEmpty()) {
		            sb.append("key=" + key);
		        } else {
		            return sb.substring(0, sb.length() - 1);
		        }
		        return sb.toString();
		    }

		    public static String encrypt(String aValue, String encoding) {
		        aValue = aValue.trim();
		        byte value[];
		        try {
		            value = aValue.getBytes(encoding);
		        } catch (UnsupportedEncodingException e) {
		            value = aValue.getBytes();
		        }
		        MessageDigest md = null;
		        try {
		            md = MessageDigest.getInstance("SHA-512");
		        } catch (NoSuchAlgorithmException e) {
		            e.printStackTrace();
		            return null;
		        }
		        return toHex(md.digest(value));
		    }

		    public static String toHex(byte input[]) {
		        if (input == null)
		            return null;
		        StringBuffer output = new StringBuffer(input.length * 2);
		        for (int i = 0; i < input.length; i++) {
		            int current = input[i] & 0xff;
		            if (current < 16)
		                output.append("0");
		            output.append(Integer.toString(current, 16));
		        }
		        return output.toString();
		    }

		    /**
		     * Verify the signature
		     *
		     * @param signParams signature parameters
		     * @param key        secret key
		     * @return true = verification succeeded
		     */
		    public static boolean verifySHA(TreeMap<String, Object> signParams, String key) {
		        String verifySign = String.valueOf(signParams.get("sign"));
		        String sign = SHAEncrypt(signParams, key);
		        if (sign.equalsIgnoreCase(verifySign)) {
		            return true;
		        }
		        return false;
		    }
		}

##### Java Signature Generation Example:

		import java.util.TreeMap;
		import com.alibaba.fastjson.JSONObject;

		public class SignTest {
		    public static void main(String[] args) {
		        String jsonData = "{\n" +
		                "    \"appId\": \"TEST000001\",\n" +
		                "    \"userId\": \"user@example.com\",\n" +
		                "    \"channel\": \"SHOPIFY\",\n" +
		                "    \"country\": \"US\"\n" +
		                "}";
		        // Sign the data
		        TreeMap resultMap = JSONObject.parseObject(jsonData, TreeMap.class);
		        String result = SHA512Utils.SHAEncrypt(resultMap, "9999");
		        System.out.println(result);

		        // Verify the signature on the data, true = verification succeeded
		        resultMap.put("sign", "DB1107649D205EE9A542EEF5DF27F69F6C2A2604D183A834ABCDD7B7A457738AD5E059B86C188B5A00973454CF90C207CBA4EDDBC91ABB66968A0ACBDE270CB0");
		        System.out.println(SHA512Utils.verifySHA(resultMap, "9999"));
		    }
		}

##### Authentication Errors

| Scenario | code | Action |
|---|---|---|
| Missing common parameters | `91001` | Supply appId / userId / sign |
| `appId` invalid or not enabled | `91003` | Verify onboarding info or contact operations |
| Signature verification failed | `91024` | Check the string to sign (dictionary order / empty-value removal / object recursion / uppercase) and the secret |
| Transient server error | `91002` | Retry the original request once |

The signature contains no timestamp — always call over HTTPS, and if the secret is leaked, contact operations immediately to rotate it.

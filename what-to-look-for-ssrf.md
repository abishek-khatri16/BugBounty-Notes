<h1 align="center">🌐 Server-Side Request Forgery (SSRF)</h1>
<p align="center"><b>Study notes: making a server send an HTTP request its owner did not intend</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/Topic-SSRF-red?style=for-the-badge" alt="Topic">
  <img src="https://img.shields.io/badge/Type-Study%20Notes-blue?style=for-the-badge" alt="Type">
  <img src="https://img.shields.io/badge/OWASP-Top%2010-orange?style=for-the-badge" alt="OWASP">
</p>

---

## 📖 What is SSRF?

SSRF happens when an application fetches a URL that the user can control. The attacker makes
the **server** send a request on their behalf, often to places the attacker cannot reach directly,
such as internal services or cloud metadata.

---

## 🧭 The 5 W's

| Question | Answer |
|----------|--------|
| ❓ **WHY** (attacker goal) | Reach internal services and read sensitive data<br>Scan the internal network and find hidden services<br>Bypass security controls and reach restricted functions<br>Send data to an external server (exfiltration)<br>Perform actions as the server or a legitimate user<br>Abuse application functions or cause disruption |
| 🎯 **WHO** is the victim | The organization hosting the infrastructure |
| 🛠️ **WHAT** is exploited | Server-side code that makes requests (axios, HttpURLConnection, HttpClient, cURL)<br>Command injection (curl, wget)<br>A CVE in the infrastructure<br>A misconfigured cloud service |
| ⏰ **WHEN** | Targeted: when the service is up<br>Enumeration is noisy, so timing affects the chance of detection |
| 📍 **WHERE** | Anywhere with an internet connection |
| 📦 **HOW** (delivery) | An HTTP request carrying user-controlled input: a parameter, header, cookie or body field |

---

## 🧨 Examples of Vulnerable Code

In each example the URL comes from the user and is fetched with no checks.

<details>
<summary><b>🟨 JavaScript (axios)</b></summary>

```javascript
const axios = require('axios');

const url = req.query.url; // comes from user input

axios.get(url)
  .then((response) => {
    console.log(response.data);
  })
  .catch((error) => {
    console.error(error);
  });
```
</details>

<details>
<summary><b>☕ Java (HttpURLConnection)</b></summary>

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;

public class SSRFExample {
    public static void main(String[] args) {
        String inputURL = args[0];
        try {
            URL url = new URL(inputURL);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.setRequestMethod("GET");
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(connection.getInputStream()));
            String line;
            StringBuilder response = new StringBuilder();
            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            reader.close();
            System.out.println(response.toString());
            connection.disconnect();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
</details>

<details>
<summary><b>🟦 C# / .NET (HttpClient)</b></summary>

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        string inputURL = args[0]; // assume this comes from user input

        using (HttpClient client = new HttpClient())
        {
            try
            {
                HttpResponseMessage response = await client.GetAsync(inputURL);
                response.EnsureSuccessStatusCode();
                string responseBody = await response.Content.ReadAsStringAsync();
                Console.WriteLine(responseBody);
            }
            catch (HttpRequestException e)
            {
                Console.WriteLine($"Request error: {e.Message}");
            }
        }
    }
}
```
</details>

<details>
<summary><b>🐘 PHP (cURL)</b></summary>

```php
<?php
$inputURL = $_GET['url']; // assume this comes from user input

$ch = curl_init($inputURL);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

$response = curl_exec($ch);

if ($response === false) {
    echo 'cURL error: ' . curl_error($ch);
} else {
    echo $response;
}

curl_close($ch);
?>
```
</details>

---

## 🛡️ How to Prevent SSRF

1. **Allowlist** the domains and protocols the app may call. Prefer this over a blocklist.
2. **Resolve the hostname and check the IP**, then block private and special ranges:
   `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` and `169.254.169.254` (cloud metadata).
3. **Disable or re-validate redirects**, because a safe URL can redirect to an internal one.
4. **Allow only `http` and `https`**, and block `file://`, `gopher://` and similar schemes.
5. **Use cloud protections** such as IMDSv2 on AWS, and give the server a minimal-permission role.
6. **Segment the network** so the application server cannot reach sensitive internal services.
7. **Do not return the raw response** from a fetched URL to the user.

---

## 🧠 Key Takeaways

- ✔️ Any feature that fetches a URL is a place to look: webhooks, image import, link previews, PDF generators
- ✔️ The impact depends on what the server can reach, so show real impact in a report
- ✔️ Only test on programs that allow it, and stay within scope

---

<p align="center">
  <sub>📚 Written for learning purposes. Use this knowledge only on systems you are authorized to test.</sub>
</p>

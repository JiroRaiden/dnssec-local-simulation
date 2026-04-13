# DNSSEC Local Simulation

---

## What is this project?

This project simulates the real-world DNSSEC (Domain Name System Security Extensions) infrastructure on a single PC. Instead of using real servers spread across the internet, you created 4 isolated virtual servers running on your machine using Docker, and made them talk to each other exactly like real DNS servers do on the internet.

By the end of the project you had:
- A working 3-tier DNS hierarchy (root → TLD → authoritative zone)
- Every DNS record cryptographically signed
- A validating resolver that checks signatures end-to-end
- Two real attacks simulated and caught by DNSSEC

---

## How basic DNSSEC chain of trust works 

When you type `example.lab` into a browser, a resolver has to find the IP address. It asks:

1. The **root server** — "who handles `.lab`?"
2. The **TLD server** — "who handles `example.lab`?"
3. The **authoritative server** — "what is the IP of `example.lab`?"

Without DNSSEC, any of these answers could be faked by an attacker. DNSSEC fixes this by making every answer come with a **digital signature**. The resolver checks the signature before trusting the answer. If the signature doesn't match, the resolver throws the answer away and returns `SERVFAIL`.

The trust flows in a chain: the root signs its own keys, the TLD proves it trusts the zone below it using a DS record, and the zone signs all its own records. This is called the **chain of trust**.

---

## PHASE 1 — Network Foundation

### What you built
Four Docker containers on a virtual private network, each acting as a separate server with its own IP address.

### Why Docker?
Real DNSSEC requires at least 4 separate machines — a root server, a TLD server, an authoritative zone server, and a resolver. You don't have 4 computers. Docker lets you run 4 isolated "virtual computers" (containers) on your single PC. Each container has its own IP, its own installed software, and cannot see inside the others.

### Why the folder structure?
```
dnssec-lab/
├── docker-compose.yml    ← describes all 4 containers
├── root-ns/
│   └── Dockerfile        ← recipe for root-ns container
├── tld-ns/
│   └── Dockerfile
├── auth-ns/
│   └── Dockerfile
└── resolver/
    └── Dockerfile
```
Each container needs its own folder because later each will get different configuration files. The Dockerfile is the recipe that tells Docker what to install inside that container.

### What is the Dockerfile doing?
```dockerfile
FROM ubuntu:22.04
```
Start from a clean Ubuntu 22.04 operating system.

```dockerfile
RUN apt-get update -y && apt-get install -y bind9 ...
```
Install BIND9 (the DNS server software) plus some useful tools.

```dockerfile
CMD ["tail", "-f", "/dev/null"]
```
Keep the container running forever doing nothing — so it doesn't shut down immediately after starting. We start BIND9 manually later.

### What is docker-compose.yml doing?
Instead of typing a long `docker run` command for each container, `docker-compose.yml` describes all 4 in one file. The important parts are:

```yaml
subnet: 172.20.0.0/24
```
Creates a private fake network. All 4 containers live on this network and can reach each other.

```yaml
ipv4_address: 172.20.0.2
```
Assigns a fixed IP to each container. root-ns always has `.2`, tld-ns `.3`, auth-ns `.4`, resolver `.5`. These never change so we can hardcode them in config files.

```yaml
cap_add: [NET_ADMIN]
```
Gives the container permission to manage network settings.

### What does `docker compose build` do?
Reads each Dockerfile, downloads Ubuntu, installs everything, and creates a frozen image. Like baking a cake — once built, you can spin up containers from it instantly without re-installing.

### What does `docker compose up -d` do?
Starts all 4 containers from their images. The `-d` means "detached" — runs in background so your terminal stays usable.

### Problem faced: permission denied on Docker socket
When you ran `docker compose build` you got `permission denied while trying to connect to the Docker daemon socket`.

**Why this happened:** Docker's socket file is owned by the `docker` group. Your user wasn't in that group.

**Fix:**
```bash
sudo usermod -aG docker $USER
```
Then close and reopen WSL. The group change only takes effect when you start a new session.

### Problem faced: `version` attribute warning
docker-compose.yml had `version: "3.9"` at the top which is now obsolete in newer Docker versions.

**Fix:** Remove that line from docker-compose.yml.

### Verification
```bash
docker exec resolver ping -c 3 172.20.0.2
```
This runs the `ping` command inside the resolver container targeting root-ns. Three packets sent, three received = the virtual network is working.

```bash
docker exec root-ns named -v
```
Runs `named -v` (BIND9 version check) inside the container. If it prints a version number, BIND9 is correctly installed.

---

## PHASE 2 — 3-Tier DNS Hierarchy

### What you built
Actual working DNS — the root server delegates to the TLD server, which delegates to the authoritative server, which answers queries. Exactly like the real internet.

### What is BIND9?
BIND9 (Berkeley Internet Name Domain, version 9) is the most widely used DNS server software in the world. It reads configuration files (`named.conf`) and zone files, then answers DNS queries based on what's in them.

### What is named.conf?
The main config file for BIND9. It tells BIND9:
- What IP to listen on
- Whether to allow recursive queries
- Which zones it is responsible for and where the zone files are

```
options {
    listen-on { 172.20.0.2; };   ← only answer on this IP
    recursion no;                 ← don't chase answers, just give what you know
};
zone "." {                        ← responsible for the root zone
    type master;                  ← this is the authoritative source
    file "/etc/bind/zones/root.zone";
};
```

### What is a zone file?
A text file containing all the DNS records for a zone. It uses a special format called the "master file format." Key records:

**SOA (Start of Authority)** — mandatory first record, identifies the zone and its admin:
```
. IN SOA root-ns. admin.root. (
    2024010101  ; serial number — increment when zone changes
    3600        ; refresh — how often secondaries check for updates
    900         ; retry — how long to wait before retrying
    604800      ; expire — when to stop serving if primary is unreachable
    86400 )     ; minimum TTL
```

**NS (Name Server)** — which server is responsible for a zone:
```
lab. IN NS tld-ns.lab.
```

**A (Address)** — maps a hostname to an IP:
```
tld-ns.lab. IN A 172.20.0.3
```
This is called a **glue record** — it provides the IP of the nameserver itself so resolvers don't get stuck in a loop.

**Delegation** — when one zone hands off responsibility to another:
```
# In root zone: "lab. is handled by tld-ns.lab."
lab. IN NS tld-ns.lab.
tld-ns.lab. IN A 172.20.0.3

# In lab. zone: "example.lab. is handled by auth-ns.example.lab."
example.lab. IN NS auth-ns.example.lab.
auth-ns.example.lab. IN A 172.20.0.4
```

### Updated Dockerfiles
The original Dockerfile just kept the container alive. Now you added:
```dockerfile
COPY etc/named.conf /etc/bind/named.conf
COPY zones/ /etc/bind/zones/
CMD ["named", "-g", "-c", "/etc/bind/named.conf"]
```
This copies your config files into the container when it's built, and starts BIND9 automatically when the container starts.

### Testing with dig
`dig` is the main tool for querying DNS servers. Key flags:
- `@172.20.0.4` — ask this specific server
- `example.lab.` — the domain to look up
- `A` — what record type to request
- `+dnssec` — request DNSSEC records too
- `+trace` — follow the full delegation chain

The full chain test:
```bash
dig @172.20.0.2 lab. NS        # root-ns says "lab. → 172.20.0.3"
dig @172.20.0.3 example.lab. NS # tld-ns says "example.lab. → 172.20.0.4"
dig @172.20.0.4 example.lab. A  # auth-ns says "IP = 172.20.0.4"
```

**What `status: NOERROR` means:** The query was answered successfully. No errors.

**What `flags: qr aa rd` means:**
- `qr` — this is a query response
- `aa` — authoritative answer (this server owns this data)
- `rd` — recursion desired (we asked for it)

**What is missing at this point:** No `ad` flag. `ad` means "authenticated data" — cryptographically verified. It won't appear until signing is complete.

---

## PHASE 3 — Key Generation, Zone Signing, DS Chain, Validation

### Step 1: Key Generation

DNSSEC uses two types of keys per zone:

**ZSK (Zone Signing Key)**
- Signs every individual DNS record in the zone (A, MX, NS, etc.)
- Short-lived — rotated every few months
- Generates smaller signatures

**KSK (Key Signing Key)**
- Only signs the ZSK itself
- Long-lived — rotated yearly
- The "root of trust" for a zone

Why two keys? If you used one key for everything, rotating the key would break the chain of trust. By splitting into KSK and ZSK, you can rotate the ZSK frequently without changing the KSK that the parent zone has already registered as trusted.

```bash
# -a ECDSAP256SHA256 : the algorithm (Elliptic Curve, 256-bit, SHA-256 hash)
# -n ZONE            : this is a zone key
# -f KSK             : flag this as a Key Signing Key
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.lab.       # ZSK
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.lab. # KSK
```

This generates 4 files per zone:
- `Kexample.lab.+013+XXXXX.key` — public key (goes in the zone)
- `Kexample.lab.+013+XXXXX.private` — private key (never shared, used only for signing)

The `+013` is the algorithm number (13 = ECDSAP256SHA256). The `XXXXX` is the key ID (a random number).

**How to tell KSK from ZSK:**
```bash
grep -l "257" Kexample.lab.+013+*.key
```
Flag 257 = KSK. Flag 256 = ZSK.

### Step 2: Zone Signing

```bash
dnssec-signzone \
  -A \                                          # sign all records
  -3 $(head -c 16 /dev/urandom | sha1sum | cut -b 1-16) \  # NSEC3 salt
  -N INCREMENT \                                # auto-increment serial
  -o example.lab. \                             # zone origin (name)
  -t \                                          # print stats when done
  -k /etc/bind/keys/Kexample.lab.+013+07525.key \  # KSK
  /etc/bind/zones/example.lab.zone \            # unsigned input zone
  /etc/bind/keys/Kexample.lab.+013+48791.key    # ZSK
```

This creates `example.lab.zone.signed` — a new zone file with RRSIG records added to every RRset.

**What is an RRSIG?** A Resource Record SIGnature. It's a digital signature over a set of DNS records. When Unbound receives a record, it also receives its RRSIG, and mathematically verifies that the signature was made by the ZSK. If the record was tampered with even by one byte, the signature verification fails.

```
example.lab. IN A     172.20.0.4        ← the record
example.lab. IN RRSIG A 13 2 86400 ...  ← signature over that record
              ^       ^ ^ ^
              type    alg labels ttl
```

**What is NSEC3?** A way to prove a name does NOT exist without revealing all zone names. The `-3` flag enables it.

### Step 3: DS Records — chaining trust upward

A DS (Delegation Signer) record is a hash of a zone's KSK that lives in the **parent zone**. It's how the parent says "I trust the keys in this child zone."

```bash
dnssec-dsfromkey Kexample.lab.+013+07525.key
# Output: example.lab. IN DS 7525 13 2 24668DC4...
```

This DS record goes into `lab.zone` (the parent). Now when Unbound resolves `example.lab`:
1. It gets the DS record from `lab.`
2. It gets the DNSKEY records from `example.lab.`
3. It hashes the KSK and compares it to the DS record
4. If they match, the keys are trusted
5. It then uses the trusted ZSK to verify all the RRSIGs

This is done for all 3 levels: root has DS for lab., lab. has DS for example.lab.

### Step 4: Validating Resolver (Unbound)

BIND9 can serve signed zones but doesn't validate. Unbound is a separate DNS resolver whose job is to validate signatures.

**The trust anchor:** Unbound needs one starting point it unconditionally trusts — the root KSK public key. This is the equivalent of the real-world IANA root key that every resolver ships with. In your simulation it's your self-generated root KSK.

```
. 86400 IN DNSKEY 257 3 13 cWUE6byjK2/...
```

**What Unbound does with it:**
1. Gets DNSKEY records from root — verifies them against the trust anchor
2. Gets DS for lab. from root — verifies it with root ZSK
3. Gets DNSKEY records from lab. — verifies them against the DS record
4. Gets DS for example.lab. from lab. — verifies it with lab. ZSK
5. Gets DNSKEY records from example.lab. — verifies them against DS
6. Gets A record + RRSIG from example.lab. — verifies RRSIG with ZSK
7. If everything checks out → sets `ad` flag on the response

**Key Unbound config directives:**
```
val-permissive-mode: no      # strict mode — reject BOGUS responses
trust-anchor-file: "..."     # where the root key lives
access-control: 0.0.0.0/0 allow  # allow queries from any IP
stub-zone: name "." stub-addr 172.20.0.2  # ask root-ns for all queries
```

### Problem faced: `rndc reload` failed
After changing `named.conf` inside the running container, `rndc reload` failed with a connection error.

**Why:** `rndc` requires a control channel configured in `named.conf`. We didn't set one up.

**Fix:** Kill named directly and restart it:
```bash
pkill named
named -g -c /etc/bind/named.conf &
```
The `-g` flag means "run in foreground and log to stderr" — useful for seeing errors.

### Problem faced: Unbound config errors
The first Unbound config used BIND9-style keywords (`allow-query`, `recursion`) which Unbound doesn't understand.

**Unbound's equivalents:**
- `allow-query { any; }` → `access-control: 0.0.0.0/0 allow`
- `recursion yes` → Unbound is recursive by default, no option needed

### Problem faced: trust-anchor inline syntax failing
Putting the KSK directly in `unbound.conf` as `trust-anchor: "..."` kept failing because the base64 key had a space in the middle (from how dig formats it) and the `trust-anchor-file` directive wasn't in the right block.

**Fix:** Use a separate file for the trust anchor and move the directive inside the `server:` block:
```
server:
    trust-anchor-file: "/etc/unbound/root.key"
```

### The `ad` flag — what it means
When you finally got:
```
flags: qr rd ra ad
```
The `ad` (Authenticated Data) flag means Unbound successfully validated the entire chain of trust from the root key all the way to the final record. Without this flag, signatures either weren't checked or failed.

---

## PHASE 4 — Attack Simulations

### Attack 1: Tampered record (DNS spoofing simulation)

**What an attacker would do in the real world:** Intercept a DNS response and change the IP address to redirect traffic to their server. Without DNSSEC, the resolver has no way to detect this.

**What you did:**
```bash
sed -i 's/172.20.0.4/10.10.10.99/g' /etc/bind/zones/example.lab.zone.signed
```
Changed every occurrence of the real IP `172.20.0.4` to a fake attacker IP `10.10.10.99` in the signed zone file, without re-signing.

**What happened:**
- auth-ns now serves `example.lab. IN A 10.10.10.99`
- But the RRSIG is still a signature over `172.20.0.4`
- Unbound fetches both records, verifies the RRSIG against the A record
- The math doesn't match → validation fails → `SERVFAIL`

**Why Unbound returned `SERVFAIL` and not the tampered IP:**
This is exactly the point of DNSSEC. The signature is mathematically tied to the exact bytes of the record. Even a one-byte change causes the verification to fail. An attacker cannot fake a valid signature without the private key.

**Result:**
```
status: SERVFAIL    ← query failed
flags: qr rd ra     ← no "ad" flag — not authenticated
ANSWER: 0           ← refused to return anything
```

### Why you needed to flush the Unbound cache
The first time you ran the test after tampering, Unbound returned the correct answer from its cache. Cache stores previous answers temporarily so it doesn't re-query every time. You had to flush it to force Unbound to fetch fresh from auth-ns:
```bash
unbound-control flush_type example.lab. A
```

### Attack 2: Expired signature (replay attack simulation)

**What an attacker would do in the real world:** Capture a signed DNS response and replay it later, after the original signature has expired. Or, if they somehow got old zone data, serve it to resolvers.

**What you did:**
```bash
sed -i 's/20260506192246/20230506192246/g' /etc/bind/zones/example.lab.zone.signed
```
Changed the RRSIG expiry date from May 2026 to May 2023 — a date 2 years in the past.

**The RRSIG expiry field:**
```
RRSIG A 13 2 86400 20260506192246 20260406192246 48791 ...
                   ^expiry         ^inception
```
Every RRSIG has an inception (when it became valid) and expiry (when it stops being valid). Unbound checks the current time against both dates. If the current time is outside this window, the signature is rejected regardless of whether the cryptography is correct.

**Why this matters:** DNSSEC signatures are time-limited on purpose. If an attacker captures old signed responses, they can't use them forever. The expiry forces zone operators to re-sign regularly, ensuring fresh signatures are always required.

**Result:**
```
status: SERVFAIL    ← expired signature rejected
flags: qr rd ra     ← no "ad" flag
ANSWER: 0
```

### How to restore after attacks
After each attack you:
1. Reverted the tampered file using `sed`
2. Restarted named inside auth-ns to serve the correct data
3. Flushed Unbound's cache to force it to re-fetch
4. Verified the `ad` flag came back

This is important — if you leave the zone tampered and move to the next phase, everything will be broken.

---

## Key concepts summary

| Term | What it is |
|------|-----------|
| BIND9 | DNS server software — serves zone data |
| Unbound | Validating resolver — checks DNSSEC signatures |
| ZSK | Signs all zone records |
| KSK | Signs the ZSK, creates chain of trust with parent |
| RRSIG | Digital signature attached to a DNS record |
| DS | Hash of child KSK stored in parent zone |
| DNSKEY | Public key published in DNS |
| NSEC3 | Proves a name doesn't exist (signed denial) |
| ad flag | Authenticated Data — Unbound validated the chain |
| SERVFAIL | Query failed — validation rejected the response |
| Chain of trust | Root KSK → DS → TLD KSK → DS → Zone KSK → ZSK → Records |
| Glue record | IP address of a nameserver in the parent zone |
| TTL | Time to live — how long resolvers cache an answer |

---

## IP address reference

| Container | IP | Role |
|-----------|-----|------|
| root-ns | 172.20.0.2 | Serves root zone, delegates lab. |
| tld-ns | 172.20.0.3 | Serves lab. zone, delegates example.lab. |
| auth-ns | 172.20.0.4 | Serves example.lab. zone with all records |
| resolver | 172.20.0.5 | Unbound — validates signatures end-to-end |

---

## Key IDs used in this simulation (your specific values)

| Zone | KSK ID | ZSK ID |
|------|--------|--------|
| example.lab. | 07525 | 48791 |
| lab. | 33400 | 44017 |
| . (root) | 51331 | 53813 |

Note: These key IDs are unique to your machine. If you rebuild, new IDs will be generated.

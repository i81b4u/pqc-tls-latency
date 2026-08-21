# Real-World TLS Performance of Post-Quantum and Hybrid Key Exchange

## Summary

This experiment measured the end-to-end HTTPS latency impact of classical, post-quantum, and hybrid TLS key-exchange groups under real network conditions.

Two independent clients at different locations repeatedly connected to the same HTTPS server for roughly one week. Each measurement batch tested eight key-exchange groups using four curl HTTP modes, with the order randomized for every batch.

The resulting dataset contains **56,512 measurements**. Of these, **56,511 completed successfully** with HTTP status `200` and curl exit code `0`, corresponding to a success rate of **99.9982%**.

The main finding is straightforward:

> **Pure ML-KEM showed no operationally meaningful end-to-end latency penalty compared with X25519. The commonly relevant hybrid group X25519MLKEM768 added only about 0.2 ms on HTTP/2 and 0.6 ms on HTTP/3 in the median paired comparison.**

The heavier hybrid groups showed larger and consistently measurable differences, especially `SecP384r1MLKEM1024`, but even those differences remained small in absolute terms for ordinary web-request latency.

This does **not** mean that ML-KEM and X25519 have identical raw computational cost. The experiment measures the complete client-observed HTTPS transaction, in which network, transport, scheduling, TLS, and application effects all contribute. The result is therefore best interpreted operationally:

> **For these tests, the additional latency introduced by ML-KEM and X25519MLKEM768 was small enough to be largely dominated by normal network and transport variation.**

---

## Why this was tested

Post-quantum cryptography is often associated with larger keys, larger handshake messages, and greater computational cost. That naturally raises a practical question:

**If post-quantum key exchange is enabled for HTTPS, will users notice a performance penalty?**

Rather than benchmarking only the cryptographic primitives in isolation, this experiment looked at what a user-facing HTTPS client actually experiences.

The comparison included:

### Classical

- `X25519`

### Pure post-quantum

- `MLKEM512`
- `MLKEM768`
- `MLKEM1024`

### Hybrid classical + post-quantum

- `X25519MLKEM768`
- `SecP256r1MLKEM768`
- `SecP384r1MLKEM1024`
- `curveSM2MLKEM768`

NIST standardized ML-KEM in FIPS 203. OpenSSL supports the pure ML-KEM TLS groups and several hybrid groups, including `X25519MLKEM768`, `SecP256r1MLKEM768`, and `SecP384r1MLKEM1024`.

---

## Test design

Two measurement clients at different locations connected repeatedly to the same server.

Each client used a dedicated test path so that the requests could also be identified independently in the server logs.

Each measurement batch contained **32 requests**:

- 8 key-exchange groups
- 4 curl HTTP modes

The tested curl modes were:

| Test mode | Purpose |
|---|---|
| `--http2` | HTTPS using HTTP/2 negotiation |
| `--http2-prior-knowledge` | HTTPS with only HTTP/2 offered via ALPN in current curl versions |
| `--http3` | Attempt HTTP/3, while allowing curl to fall back to an earlier HTTP version |
| `--http3-only` | Require HTTP/3; fail rather than fall back |

The order of all 32 requests was randomized separately for each batch.

For every request, the client recorded:

- timestamp
- target URL
- curl command/test type
- DNS lookup time
- connect time
- TLS/QUIC application-connect time
- pre-transfer time
- redirect time
- time to first byte
- total request time
- HTTP response code
- curl exit code

### Collection period

| Probe | Batches | Measurements | Approximate coverage |
|---|---:|---:|---|
| client1 | 886 | 28,352 | about 1 week |
| client2 | 880 | 28,160 | about 1 week |
| **Total** | **1,766** | **56,512** | 2 probes x about 1 week |

The probes ran independently and concurrently for approximately 165 hours each.  
Each individual test combination therefore normally has **1,766 observations** when the two locations are pooled.

---

## Data quality

The dataset was first checked for failed or invalid requests.

- Total measurements: **56,512**
- Successful requests (`HTTP 200`, curl exit `0`): **56,511**
- Failed requests: **1**
- Success rate: **99.9982%**

The single failure occurred on client1 during:

`--http3-only --curves curveSM2MLKEM768`

The request ran for approximately 10 seconds, returned HTTP response code `000`, and curl exit code `55`.

One failure among more than 56,000 measurements is insufficient evidence to associate reliability problems with that key-exchange group. The failed request was retained in the raw data but excluded from the latency aggregates.

---

## Why randomizing the test order mattered

Randomization turned out to be more important than expected.

The median DNS lookup time for the **first request in a batch** was:

**14.49 ms**

For requests in positions 2 through 32, the median was only:

**1.36 ms**

The median total request time likewise differed:

- first request in a batch: **62.38 ms**
- subsequent requests: **50.77 ms**

Had X25519 always been tested first, for example, it would have been systematically penalized by this start-of-batch effect. That could have produced the false conclusion that post-quantum groups were substantially faster.

Because the order was randomized, the first position was spread across all test combinations. Each combination appeared first between 40 and 67 times over the complete experiment.

This is a useful reminder that for small latency differences, apparently minor experimental-design choices can matter more than the effect being measured.

---

## Analysis method

Raw medians are useful for understanding overall latency, but they are not the strongest way to compare key-exchange groups because network conditions change over time.

The primary comparison therefore used a **paired design**.

For every batch and HTTP mode, the total latency of each PQ or hybrid measurement was compared with the X25519 measurement from the **same batch**.

Conceptually:

`paired difference = test total time - X25519 total time`

This substantially reduces the influence of slower or faster network periods because both measurements were made within the same short measurement window.

The tables below report the **median paired difference**.

A positive value means the test was slower than X25519.  
A negative value means it was slightly faster.

Sub-millisecond negative values should not be interpreted as evidence that ML-KEM is intrinsically faster than X25519; they show that the difference is small relative to ordinary measurement variation.

---

## Main result: latency difference relative to X25519

### Median paired difference in total request time

| Key-exchange group | HTTP/2 | HTTP/2 prior knowledge | HTTP/3 | HTTP/3 only |
|---|---:|---:|---:|---:|
| MLKEM512 | +0.019 ms | -0.092 ms | +0.609 ms | +0.947 ms |
| MLKEM768 | -0.063 ms | -0.063 ms | +0.127 ms | +0.326 ms |
| MLKEM1024 | +0.127 ms | +0.113 ms | -0.042 ms | +0.090 ms |
| **X25519MLKEM768** | **+0.217 ms** | **+0.169 ms** | **+0.596 ms** | **+0.591 ms** |
| SecP256r1MLKEM768 | +0.697 ms | +0.662 ms | +0.758 ms | +0.956 ms |
| curveSM2MLKEM768 | +2.051 ms | +2.176 ms | +3.724 ms | +3.897 ms |
| SecP384r1MLKEM1024 | +3.445 ms | +3.393 ms | +4.796 ms | +4.862 ms |

The pattern is clear.

### Pure ML-KEM

`MLKEM512`, `MLKEM768`, and `MLKEM1024` remain extremely close to X25519.

For most combinations the median difference is measured in **tenths of a millisecond**, and some values are slightly negative.

For this experiment, that is best summarized as:

> **No operationally meaningful end-to-end latency penalty was observed for pure ML-KEM compared with X25519.**

### X25519MLKEM768

The hybrid `X25519MLKEM768` result is especially relevant because it combines classical X25519 protection with ML-KEM768.

Its median paired penalty was:

- **+0.217 ms** with HTTP/2
- **+0.169 ms** with HTTP/2 prior knowledge
- **+0.596 ms** with HTTP/3
- **+0.591 ms** with HTTP/3 only

Relative to the pooled X25519 median, this corresponds to roughly:

- **0.5%** for HTTP/2
- **1.1%** for HTTP/3

For ordinary interactive HTTPS traffic, this is a very small difference.

### Other hybrid groups

A clear hierarchy appears among the heavier hybrids.

`SecP256r1MLKEM768` adds roughly **0.7–1.0 ms**.

`curveSM2MLKEM768` adds roughly:

- **2.1 ms** on HTTP/2
- **3.8–3.9 ms** on HTTP/3

`SecP384r1MLKEM1024` has the largest observed penalty:

- approximately **3.4 ms** on HTTP/2
- approximately **4.8–4.9 ms** on HTTP/3

This is consistent with OpenSSL's own documentation, which notes that `SecP384r1MLKEM1024` is substantially more CPU-intensive than the other standardized hybrid groups, largely because of the cost of the underlying P-384 operation.

---

## HTTP/3-only aggregate latency

`--http3-only` is especially useful for interpretation because curl cannot fall back to HTTP/2 or HTTP/1.x in this mode.

The following table shows the pooled raw distribution for successful HTTP/3-only requests.

| Key-exchange group | Samples | Median total | p95 | p99 |
|---|---:|---:|---:|---:|
| X25519 | 1,766 | 52.231 ms | 61.147 ms | 88.325 ms |
| MLKEM512 | 1,766 | 53.290 ms | 62.346 ms | 79.335 ms |
| MLKEM768 | 1,766 | 52.608 ms | 60.554 ms | 78.506 ms |
| MLKEM1024 | 1,766 | 52.314 ms | 60.377 ms | 76.168 ms |
| X25519MLKEM768 | 1,766 | 52.966 ms | 60.676 ms | 73.642 ms |
| SecP256r1MLKEM768 | 1,766 | 53.348 ms | 61.549 ms | 78.172 ms |
| curveSM2MLKEM768 | 1,765 | 56.377 ms | 64.158 ms | 77.238 ms |
| SecP384r1MLKEM1024 | 1,766 | 57.364 ms | 64.404 ms | 82.436 ms |

The upper percentiles do not show a general PQC-induced latency explosion. In fact, X25519 itself has the highest p99 in this particular pooled table.

The p99 values should not be over-interpreted because rare network events have a strong influence on tail latency. The important observation is that the PQ and hybrid groups do not show a systematic catastrophic tail.

---

## Baseline transport differences

The median pooled X25519 total request times were:

| curl mode | X25519 median total |
|---|---:|
| HTTP/2 | 46.686 ms |
| HTTP/2 prior knowledge | 46.660 ms |
| HTTP/3 | 52.436 ms |
| HTTP/3 only | 52.231 ms |

In this environment, the transport-level difference between the HTTP/2 and HTTP/3 measurements is therefore several milliseconds.

That difference is substantially larger than the approximately **0.2–0.6 ms** cost associated with moving from X25519 to X25519MLKEM768.

This provides useful perspective:

> **For the most practical hybrid tested, ordinary transport and network behavior contributed more latency variation than the post-quantum addition itself.**

This should not be generalized into a claim that HTTP/2 is inherently faster than HTTP/3. The experiment was designed to compare key exchange, not to provide a controlled HTTP/2-versus-HTTP/3 benchmark.

---

## Consistency between locations

The two independent probes showed the same overall ordering.

For `X25519MLKEM768`, the median paired difference was:

| Mode | client1 | client2 |
|---|---:|---:|
| HTTP/2 | +0.257 ms | +0.097 ms |
| HTTP/2 prior knowledge | +0.226 ms | +0.058 ms |
| HTTP/3 | +0.737 ms | +0.286 ms |
| HTTP/3 only | +0.673 ms | +0.443 ms |

The exact numbers differ, as expected on different network paths, but both locations support the same conclusion: the added latency is small.

The larger hybrid groups also showed the same relative pattern at both sites, increasing confidence that those differences are associated with the selected key-exchange group rather than an isolated network condition at one probe.

---

## What the experiment does and does not show

### What it supports

The measurements support the following conclusions for this particular client/server setup:

1. **Pure ML-KEM has negligible observable end-to-end HTTPS latency impact compared with X25519.**
2. **X25519MLKEM768 adds a small but measurable amount of latency: approximately 0.2 ms over HTTP/2 and 0.6 ms over HTTP/3 in the pooled paired median.**
3. `SecP256r1MLKEM768` is also inexpensive, at roughly 0.7–1.0 ms above X25519.
4. `curveSM2MLKEM768` and especially `SecP384r1MLKEM1024` show larger, consistently measurable costs.
5. Network and transport variation is larger than the latency difference between X25519 and the most relevant ML-KEM/hybrid choices.
6. No meaningful reliability difference was observed in this sample.

### What it does not prove

The experiment does **not** prove that:

- ML-KEM requires the same CPU time as X25519.
- the results will be identical on every processor, TLS implementation, server, or network.
- larger PQ handshake messages are irrelevant under packet loss, unusual MTUs, constrained links, or very high connection rates.
- server CPU and memory effects under extreme handshake load are negligible.
- the `--http3` measurements all completed using HTTP/3.

The last point is worth emphasizing: curl's `--http3` mode permits fallback to older HTTP versions. The client dataset did not record curl's `%{http_version}` variable. The `--http3-only` measurements do not have this ambiguity and therefore provide the cleanest HTTP/3 comparison.

The test also measures **end-to-end request latency**, not a microbenchmark of the cryptographic algorithms. That distinction is intentional: the original question was whether enabling PQC would impose a noticeable operational penalty on HTTPS users.

---

## Practical interpretation

For deployments considering `X25519MLKEM768`, these results are encouraging.

The hybrid retains a classical X25519 component while adding ML-KEM768, yet the measured median end-to-end latency penalty remained below one millisecond in all four tested curl modes.

In this experiment:

> **The performance cost of hybrid post-quantum key exchange was measurable, but for X25519MLKEM768 it was operationally negligible.**

This suggests that, at least from the perspective of interactive HTTPS latency, deployment concerns may be dominated more by subjects such as:

- client and server compatibility
- implementation maturity
- larger handshake messages
- behavior under packet loss and constrained networks
- operational observability
- rollout and fallback strategy

rather than by raw user-perceived latency.

---

## Conclusion

More than 56,000 real-world HTTPS measurements from two locations produced a consistent result.

Pure ML-KEM performed essentially on par with X25519 at the end-to-end request level. The `X25519MLKEM768` hybrid introduced only a fraction of a millisecond of median additional latency: approximately **0.2 ms for HTTP/2 and 0.6 ms for HTTP/3**.

Heavier hybrid constructions had a larger but still modest effect, with `SecP384r1MLKEM1024` adding roughly **3.4 ms over HTTP/2 and 4.9 ms over forced HTTP/3**.

The experiment therefore provides no evidence that adopting ML-KEM or the X25519MLKEM768 hybrid would cause a meaningful responsiveness penalty for ordinary HTTPS traffic in this environment.

Perhaps the most useful summary is simply:

> **PQC was visible in the measurements, but for X25519MLKEM768 it was effectively invisible to the user.**

---

## References

1. NIST, **FIPS 203 — Module-Lattice-Based Key-Encapsulation Mechanism Standard**, August 2024.  
   https://csrc.nist.gov/pubs/fips/203/final

2. OpenSSL documentation, **SSL_CTX_set1_curves / supported TLS groups**.  
   https://docs.openssl.org/4.0/man3/SSL_CTX_set1_curves/

3. curl documentation, **command-line HTTP version options**.  
   https://curl.se/docs/manpage.html

4. curl documentation, **HTTP/3 with curl**.  
   https://curl.se/docs/http3.html

---

## Reproducibility notes

The raw data consisted of two gzip-compressed CSV-like result files produced by independent probes.

For analysis:

- numeric timing fields were parsed as seconds and converted to milliseconds for presentation;
- measurements were considered successful only when the HTTP response was `200` and the curl exit code was `0`;
- the single failed request was excluded from latency aggregates;
- raw distribution statistics were calculated from successful measurements;
- paired latency differences were calculated within the same probe, batch, and curl mode using X25519 as the baseline.

Using paired measurements is important because it reduces the influence of time-varying network conditions and makes small algorithm-related differences easier to distinguish from ordinary latency variation.

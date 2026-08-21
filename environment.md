# Description of the environment

## Used software
Client software: `curl` -> docker image [i81b4u/byo-curl:8.21.0-2026081101](https://hub.docker.com/layers/i81b4u/byo-curl/8.21.0-2026081101/images/sha256-7119df4af2e86b0e489e2031663e26fffe92b6524f830e3dcffa32a632f085a2)
Server software: `nginx` -> docker image [i81b4u/byo-nginx:1.31.3-alpine-slim](https://hub.docker.com/layers/i81b4u/byo-nginx/1.31.3-alpine-slim/images/sha256-a3c9801db465e9f9fc3dec1e845fbf2f19081eb28b4298c34ac760f9260ec31d)

### Curl setup
The script `curves-test.sh` was executed every 10 to 12 minutes to record connection results from within a `tmux` session.
```bash
while true ; do curves-test.sh && sleep $((600 + RANDOM % 120)) ; done >> results.txt
```
#### `curves-test.sh`
```bash
#!/bin/bash

# FQDN to perform the tests on
TARGET_FQDN="https://[REDACTED]/pqc-test-client1"

# Template for neatly displaying the different curl variables
CURL_TIME="\"%{time_namelookup}\",\"%{time_connect}\",\"%{time_appconnect}\",\"%{time_pretransfer}\",\"%{time_redirect}\",\"%{time_starttransfer}\",\"%{time_total}\",\"%{response_code}\",\"%{exitcode}\""

# tests to be performed by curl
CMD_ARRAY=( \
  "--http3 --curves X25519" \
  "--http3 --curves MLKEM1024" \
  "--http3 --curves MLKEM768" \
  "--http3 --curves MLKEM512" \
  "--http3 --curves X25519MLKEM768" \
  "--http3 --curves SecP256r1MLKEM768" \
  "--http3 --curves SecP384r1MLKEM1024" \
  "--http3 --curves curveSM2MLKEM768" \
  "--http2 --curves X25519" \
  "--http2 --curves MLKEM1024" \
  "--http2 --curves MLKEM768" \
  "--http2 --curves MLKEM512" \
  "--http2 --curves X25519MLKEM768" \
  "--http2 --curves SecP256r1MLKEM768" \
  "--http2 --curves SecP384r1MLKEM1024" \
  "--http2 --curves curveSM2MLKEM768" \
  "--http3-only --curves X25519" \
  "--http3-only --curves MLKEM1024" \
  "--http3-only --curves MLKEM768" \
  "--http3-only --curves MLKEM512" \
  "--http3-only --curves X25519MLKEM768" \
  "--http3-only --curves SecP256r1MLKEM768" \
  "--http3-only --curves SecP384r1MLKEM1024" \
  "--http3-only --curves curveSM2MLKEM768" \
  "--http2-prior-knowledge --curves X25519" \
  "--http2-prior-knowledge --curves MLKEM1024" \
  "--http2-prior-knowledge --curves MLKEM768" \
  "--http2-prior-knowledge --curves MLKEM512" \
  "--http2-prior-knowledge --curves X25519MLKEM768" \
  "--http2-prior-knowledge --curves SecP256r1MLKEM768" \
  "--http2-prior-knowledge --curves SecP384r1MLKEM1024" \
  "--http2-prior-knowledge --curves curveSM2MLKEM768"
)

# Determine array size
CMD_ARRAY_SIZE=${#CMD_ARRAY[@]}

# Create a shuffled array to randomize the order of commands to be executed
VAL_ARRAY=($(seq 0 1 $(( $CMD_ARRAY_SIZE-1 )) ))
VAL_ARRAY=( $(shuf -e "${VAL_ARRAY[@]}") )

# Print header
echo "\"Time\",\"Target FQDN\",\"Test performed\",\"Time namelookup\",\"Time connect\",\"Time appconnect\",\"Time pretransfer\",\"Time redirect\",\"Time starttransfer\",\"Time total\",\"Response code\",\"Exit code\""

# Execute commands in random order
for VAL in "${VAL_ARRAY[@]}"
do
  printf '"%s",' "$(date --iso-8601=seconds)"
  echo -n "\"$TARGET_FQDN\",\"${CMD_ARRAY[$VAL]}\","
  docker run --rm i81b4u/byo-curl:latest -w "$CURL_TIME" --silent --output /dev/null ${CMD_ARRAY[$VAL]} $TARGET_FQDN
  echo ""
done
```

### Nginx setup

The following relevant settings were used for `nginx`.

```bash
  # SSL
  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:10m;
  ssl_session_tickets off;
  ssl_dyn_rec_enable on;
  ssl_certificate_compression on;
  ssl_ecdh_curve MLKEM1024:MLKEM768:MLKEM512:SecP256r1MLKEM768:X25519MLKEM768:SecP384r1MLKEM1024:curveSM2MLKEM768:X25519:P-384:P-256;
  ssl_conf_command SignatureAlgorithms mldsa87:mldsa65:mldsa44:ed448:ed25519:ecdsa_secp521r1_sha512:ecdsa_secp384r1_sha384:ed25519:ecdsa_secp256r1_sha256;
  ssl_buffer_size 4k;

  # QUIC
  http3 on;
  quic_retry on;
  quic_gso on;

  # modern configuration
  ssl_prefer_server_ciphers on;
  ssl_protocols TLSv1.2 TLSv1.3;
  #ssl_ciphers ECDHE+AESGCM;
  ssl_ciphers ECDHE+AESGCM:ECDHE+AESCCM:ECDHE+CHACHA20:@STRENGTH;
  ssl_conf_command Options PrioritizeChaCha;
  ssl_conf_command Options KTLS;
```
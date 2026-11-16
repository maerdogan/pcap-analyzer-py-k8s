# Install

Read the [security section](../README.md#security) first. HTTPS, network isolation and encrypted storage are yours to add.

Everything below works without internet in the cluster once the image is in your registry.

## Kubernetes

Push the image to your internal registry, then set the image and host names in `kubernetes/generic/`. Create the password hash on any machine with the image:

```bash
docker run --rm -it pcap-analyzer:0.8.0 pcap-analyzer hash-password
```

```bash
kubectl create namespace pcap-analyzer
kubectl -n pcap-analyzer create secret generic pcap-analyzer-admin --from-literal=password-hash='PASTE_HASH_HERE'
kubectl apply -k kubernetes/generic
```

Keep one replica: sessions and running jobs live in memory. The Ingress annotations assume ingress-nginx. Other controllers need their own upload limit of at least 1100 MB. `kubernetes/vks` is an example overlay for F5 NGINX Ingress Controller, which uses `nginx.org` annotations.

`kubernetes/generic/networkpolicy.yaml` lets only the namespace `ingress-nginx` reach port 8000, and gives the pod no way out. If your controller runs in another namespace, change the name there, or the app becomes unreachable. It works only where the network plugin enforces policies, and `kubernetes/vks` does not include it.

To use it without an Ingress:

```bash
kubectl -n pcap-analyzer port-forward deploy/pcap-analyzer 8000:8000
```

With Helm, create the same Secret if you want your own password, then install the chart. `kubernetes/helm/pcap-analyzer/values.yaml` lists the image, storage, Ingress, Gateway and NetworkPolicy settings:

```bash
helm install pcap-analyzer kubernetes/helm/pcap-analyzer -n pcap-analyzer --set image.repository=REGISTRY/pcap-analyzer,image.tag=VERSION
```

In a pod the default seccomp profile refuses namespaces, so tshark runs with Linux Landlock instead, which keeps it away from other analyses. It has no network limit there, and `-n` means it makes no lookups. A kernel without Landlock leaves only the CPU and file size limits.

## Check an install

`tools/smoke.py` signs in, uploads a capture, opens every page, downloads each export, deletes the analysis and signs out. It stops at the first failure and needs only Python. Use a capture with plain HTTP traffic, and a password set in the app or the Secret.

```bash
kubectl -n pcap-analyzer port-forward deploy/pcap-analyzer 8000:8000 &
read -rs PCAP_ANALYZER_SMOKE_PASSWORD && export PCAP_ANALYZER_SMOKE_PASSWORD
python3 tools/smoke.py http://127.0.0.1:8000 capture.pcap
```

## Settings

| Variable | Default | Meaning |
|---|---|---|
| `PCAP_ANALYZER_ADMIN_USER` | `admin` | Login name |
| `PCAP_ANALYZER_ADMIN_PASSWORD_HASH` | | Password hash, preferred |
| `PCAP_ANALYZER_ADMIN_PASSWORD` | | Plain password, as the alternative. With neither set, or with the password `admin`, the first sign-in is `admin` / `admin` and every page asks for a new password first. The new password is kept in `.admin-password.json` in the data folder and wins over these settings; delete that file to go back to them |
| `PCAP_ANALYZER_BEHIND_PROXY` | `0` | `1` when an HTTPS proxy is in front |
| `PCAP_ANALYZER_HSTS_S` | `31536000` | How long browsers must use HTTPS, sent whenever HTTPS is on. `0` tells them to stop. Applies to the host name on every port |
| `PCAP_ANALYZER_REQUEST_ID_HEADER` | `x-request-id` | Header that follows a request through proxies. `traceparent` is read when it is absent |
| `PCAP_ANALYZER_MERGECAP` | `mergecap` | Program that joins parts of a capture |
| `PCAP_ANALYZER_SANDBOX` | on where possible | `0` stops running tshark in its own namespaces |
| `PCAP_ANALYZER_TRUSTED_PROXIES` | | Proxy addresses or networks whose forwarded headers are believed, such as `10.0.0.5,10.1.0.0/16`. When empty, all users behind the proxy share one login lockout |
| `PCAP_ANALYZER_WORKDIR` | `~/.local/share/pcap-analyzer` | Where results are saved |
| `PCAP_ANALYZER_MAX_SIZE_MB` | `1000` | Largest accepted capture. Captures over 100 MB are read in parts of whole connections |
| `PCAP_ANALYZER_TTL_DAYS` | `7` | Days results are kept. The server removes older ones every hour |
| `PCAP_ANALYZER_MAX_JOBS` | `2` | Analyses running at once |
| `PCAP_ANALYZER_KEEP_CAPTURE` | `1` | Keep the capture for PCAP export, `0` to delete it after analysis. The Statistics tab, TCP graphs, Flow Graph, Follow Stream and Export Objects read the kept capture, so they need `1` |
| `PCAP_ANALYZER_GEOIP_DIR` | none | A folder of MaxMind GeoLite2 `.mmdb` files. IP endpoints then show their country and AS number |
| `PCAP_ANALYZER_LANGUAGE` | `tr` | Default language: `tr`, `en`, `de` or `es`. Each user can choose another in the app |
| `PCAP_ANALYZER_ANALYSIS_TIMEOUT_S` | `600` | Longest analysis in seconds, `0` for no limit |
| `PCAP_ANALYZER_KNOWLEDGE` | | Path of another knowledge file, built with `tools/build_knowledge.py` |

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

The hash is optional: without it the first sign-in is `admin` / `admin`. At the first start the app takes it over into the data folder and logs a warning; from then on the Secret is ignored, and the admin password only unlocks the encrypted data.

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
| `PCAP_ANALYZER_ADMIN_USER` | `admin` | Name of the setup admin, who opens the encrypted data after every restart. Other users are made in the app under **Settings → Users** |
| `PCAP_ANALYZER_ADMIN_PASSWORD_HASH` | | A first password for the setup admin, as a hash from `pcap-analyzer hash-password`. Optional |
| `PCAP_ANALYZER_ADMIN_PASSWORD` | | A first password in plain text, as the alternative. Optional. With neither set, or with the password `admin` or one under 12 characters, the first sign-in is `admin` / `admin` or that password, and every page asks for a new password first. Either is taken over once into `.admin-password.json` in the data folder, with a warning in the log to remove it from the settings, and is ignored from then on. Once the master key is set, that file is deleted: the admin password then only unwraps the admin key of the encrypted data folder |
| `PCAP_ANALYZER_LDAP_CA_FILE` | | Certificates to trust for an `ldaps://` server signed by a private CA. Without it the system's trusted certificates are used |
| `PCAP_ANALYZER_BEHIND_PROXY` | `0` | `1` when an HTTPS proxy is in front |
| `PCAP_ANALYZER_HSTS_S` | `31536000` | How long browsers must use HTTPS, sent whenever HTTPS is on. `0` tells them to stop. Applies to the host name on every port |
| `PCAP_ANALYZER_REQUEST_ID_HEADER` | `x-request-id` | Header that follows a request through proxies. `traceparent` is read when it is absent |
| `PCAP_ANALYZER_CLIENT_IP_HEADER` | `x-forwarded-for,x-real-ip,true-client-ip,x-client-ip,client-ip,forwarded` | Headers that carry the original client, comma separated, tried in order. A PROXY protocol header is used when none has one. A NetScaler adds its own with *Client IP header* on the service |
| `PCAP_ANALYZER_HEALTH_PATHS` | | More health check paths, comma separated, besides `/healthz`, `/ready`, `/actuator/health` and the like. Requests to them are marked as health checks |
| `PCAP_ANALYZER_MERGECAP` | `mergecap` | Program that joins parts of a capture |
| `PCAP_ANALYZER_SANDBOX` | on where possible | `0` stops running tshark in its own namespaces |
| `PCAP_ANALYZER_TRUSTED_PROXIES` | | Proxy addresses or networks whose forwarded headers are believed, such as `10.0.0.5,10.1.0.0/16`. When empty, all users behind the proxy share one login lockout |
| `PCAP_ANALYZER_WORKDIR` | `~/.local/share/pcap-analyzer` | The data folder: captures, results, users, settings, the activity log and the encryption keys, all encrypted once the master key is set. Uploads and running analyses use private `.tmp-` folders in it. Back it up together with the master key |
| `PCAP_ANALYZER_MAX_SIZE_MB` | `1000` | Largest accepted capture. Captures over 100 MB are read in parts of whole connections |
| `PCAP_ANALYZER_TTL_DAYS` | `7` | An analysis nobody opened for this many days is deleted, at start and every hour. One never opened counts from its upload |
| `PCAP_ANALYZER_MAX_JOBS` | `2` | Analyses running at once |
| `PCAP_ANALYZER_KEEP_CAPTURE` | `1` | Keep the capture for PCAP export, `0` to delete it after analysis. The Statistics tab, TCP graphs, Flow Graph, Follow Stream and Export Objects read the kept capture, so they need `1` |
| `PCAP_ANALYZER_GEOIP_DIR` | none | A folder of MaxMind GeoLite2 `.mmdb` files. IP endpoints then show their country and AS number |
| `PCAP_ANALYZER_LANGUAGE` | `tr` | Default language: `tr`, `en`, `de` or `es`. Each user can choose another in the app |
| `PCAP_ANALYZER_ANALYSIS_TIMEOUT_S` | `600` | Longest analysis in seconds, `0` for no limit |
| `PCAP_ANALYZER_KNOWLEDGE` | | Path of another knowledge file, built with `tools/build_knowledge.py` |

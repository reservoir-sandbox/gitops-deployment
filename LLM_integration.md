# Reservoir LLM Summarizer — DevOps Handoff

## What is already deployed

The dedicated Yandex VM runs two services:

```text
Reservoir ML API (Gunicorn/Flask) -> 127.0.0.1:8000
Ollama / Qwen 3.5 4B             -> 127.0.0.1:11434
```

VM details:

```text
Public IP:  158.160.219.46
Private IP: 10.130.0.8
OS user:    aitemir
Resources:  8 vCPU, 8 GB RAM, 4 GB swap
Code:       /home/aitemir/reservoir
```

Services are enabled across reboots:

```bash
sudo systemctl status ollama
sudo systemctl status reservoir-ml.service
```

Qwen is managed by Ollama, not stored in the Git repository:

```bash
ollama list
ollama pull qwen3.5:4b
```

The `models/` directory in the repository contains the separate sklearn file
classifier artifacts. `ml-models` is only a Docker Compose service name; it is
not a model or repository.

## Give the DevOps engineer access safely

Do not share an existing private SSH key. Give the engineer an individual
Yandex OS Login/IAM account, or have them create a dedicated key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/reservoir_yandex -C "devops-reservoir"
```

The engineer sends only:

```text
~/.ssh/reservoir_yandex.pub
```

An administrator adds that public key to the VM user or through Yandex Cloud
OS Login. The engineer connects with their own private key:

```bash
ssh -o IdentitiesOnly=yes \
  -i ~/.ssh/reservoir_yandex \
  aitemir@158.160.219.46
```

Revoke that engineer's key/account independently when access is no longer
required.

## Verify the current deployment

On the VM:

```bash
sudo systemctl is-enabled ollama reservoir-ml.service
sudo systemctl is-active ollama reservoir-ml.service
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:8000/ready
```

Readiness must include:

```json
{
  "status": "ready",
  "llm_ready": true,
  "llm_model": "qwen3.5:4b"
}
```

Useful logs:

```bash
sudo journalctl -u reservoir-ml.service -f
sudo journalctl -u ollama -f
```

## Network integration

The API currently binds to loopback for safety. A backend on another machine
cannot reach it until private access is configured.

Recommended configuration:

1. Put the backend and ML VM in the same Yandex VPC, or connect their networks
   through VPN/peering.
2. Create a security-group rule allowing TCP `8000` only from the backend's
   security group/private address.
3. Do not allow `0.0.0.0/0` to port `8000`.
4. Never expose Ollama port `11434`.
5. Bind the ML API to private IP `10.130.0.8`.

Edit `/etc/systemd/system/reservoir-ml.service` and change:

```text
--bind 127.0.0.1:8000
```

to:

```text
--bind 10.130.0.8:8000
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart reservoir-ml.service
sudo systemctl status reservoir-ml.service
```

From the backend host:

```bash
curl -fsS http://10.130.0.8:8000/ready
```

Backend environment:

```env
RESERVOIR_SUMMARIZER_URL=http://10.130.0.8:8000
RESERVOIR_SUMMARIZER_TIMEOUT_SECONDS=300
```

For development from an arbitrary network, use an SSH tunnel instead of
opening port 8000:

```bash
ssh -N -o IdentitiesOnly=yes \
  -i ~/.ssh/reservoir_yandex \
  -L 8000:127.0.0.1:8000 \
  aitemir@158.160.219.46
```

## Application flow

```text
User submits ELF
  -> backend creates analysis job
  -> static-analysis service returns inline JSON or S3 key
  -> YARA scanner returns matched rules
  -> backend worker fetches S3 report when needed
  -> worker calls POST /api/v1/summarize-static
  -> worker stores the complete response
  -> frontend reads the stored result from the backend
```

Inference is CPU-only and currently takes about two to three minutes. Call the
summarizer from a background worker, not from a frontend request.

Suggested states:

```text
STATIC_PENDING -> STATIC_READY -> SUMMARY_PENDING -> SUMMARY_READY
                                           \-> SUMMARY_FAILED
```

## Request: inline report

The backend may forward the static service envelope and add the actual ELF hash
and YARA scanner matches:

```http
POST http://10.130.0.8:8000/api/v1/summarize-static
Content-Type: application/json
```

```json
{
  "task_id": "12345-abcde",
  "status": "success",
  "location": "inline",
  "report": {
    "filename": "sample.elf",
    "file_size_bytes": 111111,
    "metadata": {},
    "header": {},
    "sections": [],
    "symbols": {},
    "strings_analysis": {},
    "security_mitigations": {}
  },
  "sha256": "actual-elf-sha256",
  "yara_matches": [
    {
      "rule": "Mirai_EM_MIPS",
      "namespace": "auto_yara",
      "meta": {
        "author": "auto-yara",
        "description": "Mirai-derived ELF signature"
      },
      "strings": [
        {"identifier": "$s1", "data": "busybox"}
      ]
    }
  ]
}
```

`yara_matches` is optional. Omit it when no scanner rule fired. It is required
for defensible malware-family inference when a rule does fire.

## Request: report stored in S3

If static analysis returns:

```json
{
  "location": "s3",
  "s3_report_key": "reports/12345-abcde_report.json"
}
```

the trusted backend worker must download that object using its existing S3
credentials, parse it, and send:

```json
{
  "static_analysis": {"filename": "sample.elf"},
  "sha256": "actual-elf-sha256",
  "yara_matches": []
}
```

The ML service intentionally does not accept arbitrary S3 credentials/keys.

## Response

```json
{
  "verdict": "malicious",
  "confidence": 0.95,
  "family": "Mirai",
  "malware_type": "botnet",
  "summary": "One-line user-facing summary.",
  "report_markdown": "Detailed analyst report.",
  "evidence": [
    {
      "title": "Persistence mechanisms",
      "explanation": "Cron persistence commands are embedded in the file.",
      "source_ids": ["string:persistence_cron"],
      "examples": ["/etc/cron.d/tengu"]
    }
  ],
  "indicators": {
    "ips": [],
    "domains": [],
    "urls": [],
    "hashes": ["sha256:..."],
    "mutexes": [],
    "file_paths": []
  },
  "mitre_attack": [],
  "source": {},
  "generation": {
    "mode": "llm",
    "model": "qwen3.5:4b"
  }
}
```

The response includes `yara_matches` only when matches were supplied. The
backend should persist the complete response. The frontend renders:

1. Verdict/confidence/family/type/summary card.
2. Sanitized `report_markdown`.
3. Evidence entries and examples.
4. IOC tables.
5. MITRE ATT&CK tags.

If `generation.mode` is `deterministic_fallback`, show the result as
`inconclusive` and expose the diagnostic reason only to administrators.

## Updating the service

After the ML changes are committed and available in the shared repository:

```bash
cd /home/aitemir/reservoir
git pull --ff-only
.venv/bin/pip install -r requirements.txt
sudo systemctl restart reservoir-ml.service
curl -fsS http://127.0.0.1:8000/ready
```

Do not overwrite local configuration or model storage during deployment. Use a
normal CI/CD artifact or controlled Git revision in production.
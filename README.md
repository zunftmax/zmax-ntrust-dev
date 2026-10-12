# Zmax ntrust · dev

Zmax is proud to support and open source these reference images for trusted and confidential inference and token streaming. This repository is the CPU-first tryout image in the ntrust family: same guest packages and service layout as [zmax-ntrust-vllm](https://github.com/ZUNFTMAX/zmax-ntrust-vllm), but the boot gate accepts machines without a GPU so developers can exercise cloud-init, TLS issuance, authentication, and the inference API without GPU quota. This repository packages **vLLM 0.31.0** for the CPU tryout path.

**Status:** experimental tryout image derived from the vLLM recipe. No cloud run has been validated yet. A successful local boot proves provisioning and API behavior only. Never present a dev boot as confidential GPU evidence: the CPU path performs no GPU verification.

## What runs

```text
Client ── TLS 1.3 / JSON or SSE ── cloud TCP forwarding ── guest NGINX :443
                                                           │ loopback
                                                      vLLM :8080 (CPU)
```

NGINX obtains its certificate directly from Let's Encrypt with TLS-ALPN-01. Only `GET /v1/models` and `POST /v1/chat/completions` are public. Responses return over the client's existing HTTPS connection; streaming uses SSE. Keep any load balancer at layer 4 so TLS terminates in the guest.

The boot gate still requires a confidential CPU VM (AMD SEV-SNP or Intel TDX) and disabled swap before minting the per-boot API credential. When exactly one NVIDIA GPU in production CC-On mode is attached, the gate runs full local NVIDIA verification first; when no GPU is attached, the gate logs the CPU tryout path and continues. More than one GPU is always rejected.

## Privacy and key lifecycle

- TLS/ACME state, the API credential, request working files and JIT caches stay in RAM. Swap, core dumps, suspend, hibernation and pstore are disabled.
- A full HTTPS restart deliberately creates a new TLS key and certificate. Reload retains the running ACME state. Reboot creates a new API credential. Use ACME staging while testing repeated restarts.
- The inference process can connect only to loopback. No prompt/response logging, remote inference fallback or telemetry service is configured. Upstream logging outputs are discarded; usage tracking and OpenTelemetry are disabled.
- The root disk contains software and a separately provisioned public model. It is not an encrypted secret store. Cloud metadata contains only public bootstrap configuration.

These controls do not let a customer authenticate the running image, and the CPU path adds no GPU evidence at all. Certificate Transparency records issuance, not image approval. The cloud still observes traffic metadata, allocation and availability.

## Build on Linux

Use an x86-64 Linux host with KVM, Packer 1.16.1, QEMU 8.2.2 or compatible tools, OVMF 4M, xorriso, Python 3, gettext and ShellCheck. The [minimal kernel build](kernel/README.md) also needs GCC/G++, make, bc, bison, flex, libssl-dev, libelf-dev, GnuPG and dpkg tools. Image builds for this project run on `axp.computer` over SSH. Do not emulate the image on macOS.

```sh
make check
make image
make export
bash tests/smoke-guest.sh
bash tests/inspect-image.sh
make sbom
sha256sum --check dist/SHA256SUMS
```

Outputs are `dist/qemu/ntrust.qcow2`, `dist/ntrust-azure.vhd`, `dist/ntrust-gcp.tar.gz` and `dist/SHA256SUMS`. Build/SSH keys and disk files are ignored by Git. The ordinary KVM smoke test must reject GPU inference while permitting the CPU tryout path; it cannot exercise confidential GPU inference.

The [hashed Python dependency lock](image/requirements.lock) pins the engine wheels. Base package versions are recorded from each build. Ubuntu packages are not snapshot-pinned, so reproducible releases are not claimed.

## Deploy

Launch on any SEV-SNP or TDX CPU confidential VM for tryout work; attach a CC-capable NVIDIA GPU (H100, H200, B100, B200) only when moving to GPU validation. Follow the separate [Azure and GCP launch recipes](image/clouds.md). Obtain a DNS hostname, an operator SSH public key and TCP 443 pass-through. Copy `guest/cloud-init.example.yaml`, replace the public placeholders and use the staging CA first.

The example provisions Qwen2.5-0.5B-Instruct from an exact revision, verifies every file against [the model manifest](guest/model.json), and installs it as `/var/lib/ntrust/model`. It is a small bring-up model.

## Run

Retrieve `/run/ntrust-gpu/api-key` using your trusted operator SSH channel and place it in your client's secret storage. Do not send it through cloud metadata or application logs.

```sh
# NTRUST_API_KEY is the per-boot credential; DOMAIN is your provisioned hostname.
curl --fail --no-buffer "https://$DOMAIN/v1/chat/completions" \
  -H "Authorization: Bearer $NTRUST_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"model","messages":[{"role":"user","content":"Reply with hello."}],"max_tokens":16,"stream":true}'
```

Use `sudo ntrust-status` for payload-free local diagnostics. Full HTTPS restarts reissue certificates; use staging during diagnosis to avoid production issuance limits.

## Project contents

| Path | Purpose |
| --- | --- |
| `guest/` | Small startup helpers, upstream server configuration and service isolation |
| `kernel/` | Linux 7.2.10 feature allowlist, signed headless NVIDIA modules and build recipe |
| `image/` | Packer recipe, hashed dependencies, disk export and provider launch instructions |
| `tests/` | Admission/credential tests, ordinary UEFI boot checks and artifact inspection |
| `docs/REMOTE-ATTESTATION.md` | Reserved follow-on scope for customer verification and sealing |

Project code uses the MIT license. Upstream Python wheels, NVIDIA components, NGINX and Ubuntu retain their own licenses. Installed license files and package metadata are retained in the image; the public demonstration model includes its own license.

# cert-mgmt

Ansible playbooks for ACME certificate issuance using DNS-01 validation.

## Structure

```
ansible-cert-mgmt/
├── main.yml                              # Orchestrator playbook
├── vars/
│   ├── acme_account.yml                  # ACME account config (gitignored)
│   ├── acme_account.yml.sample           # Template
│   ├── certs/
│   │   ├── <domain>.yml                  # Per-cert config (gitignored)
│   │   └── example.yml.sample            # Template
│   └── dns_providers/
│       ├── dnsmadeeasy.yml               # Provider credentials (gitignored)
│       └── dnsmadeeasy.yml.sample        # Template
├── tasks/
│   ├── acme_account.yml                  # Account init / verify
│   ├── issue_cert.yml                    # Per-cert issuance logic
│   └── dns_providers/
│       └── dnsmadeeasy/
│           ├── create.yml                # Publish TXT challenge records
│           └── cleanup.yml              # Remove TXT challenge records
├── account/                              # ACME account keys (gitignored)
└── certs/                                # Generated certs and keys (gitignored)
```

## Prerequisites

- Ansible installed
- Collections:
  - `community.crypto`
  - `community.general`

```bash
ansible-galaxy collection install community.crypto community.general
```

## Configure

### 1. ACME account (`vars/acme_account.yml`)

Copy `vars/acme_account.yml.sample` to `vars/acme_account.yml` and set:

- `acme_email` — contact email registered with Let's Encrypt
- `acme_directory` — production or staging Let's Encrypt endpoint
- `acme_account_key` — path where the account private key will be stored

### 2. DNS provider credentials

Copy the appropriate sample to remove the `.sample` suffix and fill in credentials.

**DNS Made Easy** (`vars/dns_providers/dnsmadeeasy.yml`):
```yaml
dnsmadeeasy_api_key: YOUR_KEY
dnsmadeeasy_secret_key: YOUR_SECRET
```

For CI/CD pipelines (e.g. Jenkins), credentials can be passed as Ansible extra vars
at runtime instead of using a file:

```bash
ansible-playbook main.yml \
  -e dnsmadeeasy_api_key=$DME_KEY \
  -e dnsmadeeasy_secret_key=$DME_SECRET
```

### 3. Certificate config (`vars/certs/<domain>.yml`)

Copy `vars/certs/example.yml.sample` to a new file named after your domain and set:

| Field | Required | Description |
|-------|----------|-------------|
| `domain_name` | yes | Primary domain (used for directory and CSR CN) |
| `subject_alt_name` | yes | List of SANs; include `domain_name` as first entry |
| `key_type` | yes | `RSA` or `ECC` |
| `key_size` | RSA only | Key size: `2048`, `3072`, or `4096` (default: `4096`) |
| `key_curve` | ECC only | Curve: `secp256r1`, `secp384r1`, or `secp521r1` (default: `secp384r1`) |
| `organization_name` | no | O field in the CSR |
| `country_name` | no | C field in the CSR (default: `US`) |
| `dns_provider` | yes | DNS provider to use: `dnsmadeeasy` |

Every `*.yml` file in `vars/certs/` is automatically processed on each run.

## Run

```bash
ansible-playbook main.yml
```

The playbook is fully idempotent:
- The ACME account key is only generated and registered once; subsequent runs verify the existing account.
- Certificates with more than 91 days of validity are skipped.
- DNS challenge records are created and cleaned up automatically.

## Output

Per-domain files are created under `certs/<domain_name>/`:

| File | Description |
|------|-------------|
| `privkey.pem` | Certificate private key (passphrase-encrypted) |
| `passphrase.txt` | Passphrase for the private key |
| `cert.csr` | Certificate Signing Request |
| `cert.pem` | Signed certificate |
| `chain.pem` | Intermediate certificate chain |
| `fullchain.pem` | Certificate + chain |
| `cert.pfx` | PKCS12 bundle (same passphrase as private key) |
| `cert.jks` | Java KeyStore (password: `changeit`) |

## Adding a new DNS provider

1. Create `tasks/dns_providers/<provider>/create.yml` and `cleanup.yml`.
   Both files receive:
   - `acme_challenge` — registered result from the ACME certificate order
   - `acme_challenge_type` — challenge type (e.g. `dns-01`)
   - `subject_alt_name` — list of domains to create records for
   - Any variables loaded from `vars/dns_providers/<provider>.yml`

2. Create `vars/dns_providers/<provider>.yml.sample` with credential placeholders.

3. Set `dns_provider: <provider>` in the cert config file.

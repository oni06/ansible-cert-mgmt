# cert-mgmt

Ansible playbooks for ACME certificate issuance using DNS-01 validation with DNS Made Easy.

## Files

- `main.yml`: current playbook
- `vars/main.yml`: runtime variables (sample placeholders included)
- `account/`: ACME account private keys (ignored by git)
- `certs/`: generated keys and certificates (ignored by git)

## Prerequisites

- Ansible installed
- Collections:
  - `community.crypto`
  - `community.general`
- DNS Made Easy API access

Install collections:

```bash
ansible-galaxy collection install community.crypto community.general
```

## Configure Variables

Edit `vars/main.yml` and set:

- `acme_email`
- `acme_directory` and `acme_account_key`
- `dnsmadeeasy_api_key` and `dnsmadeeasy_secret_key`
- `domain_name`
- `subject_alt_name` (SAN list)

## Run

```bash
ansible-playbook main.yml
```

## Output

Per-domain files are created under `certs/<domain_name>/`:

- `privkey.pem`
- `cert.csr`
- `cert.pem`
- `chain.pem`
- `fullchain.pem`
- `cert.pfx`
- `cert.jks`
- `passphrase.txt`

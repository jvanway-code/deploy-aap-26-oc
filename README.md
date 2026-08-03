# Deploy Ansible Automation Platform 2.6 on OpenShift

This repository automates the deployment, bootstrapping, and licensing of Ansible Automation Platform (AAP) 2.6 on OpenShift (including Red Hat OpenShift Local / CRC) using Ansible deployment roles from `redhat-cop`.

---

## Prerequisites

Before running the deployment, ensure the following are installed and configured on your workstation:

1. **OpenShift CLI (`oc`)**: Installed and logged into the target cluster (`oc login` or active CRC session).
2. **Ansible Core**: Installed locally (`ansible-playbook` 2.15+).
3. **Red Hat Subscription Manifest**: A valid AAP subscription manifest `.zip` file from the [Red Hat Customer Portal](https://access.redhat.com/).

---

## Step-by-Step Deployment Guide

### Step 1: Verify OpenShift Connection

Ensure your shell is authenticated with OpenShift. If using Red Hat OpenShift Local / CRC:

```bash
eval $(crc oc-env)
```

Verify `oc` connectivity and logged-in user:

```bash
oc whoami
```

---

### Step 2: Configure Automation Hub Credentials & Install Collections

Set your Red Hat Automation Hub Offline Token (obtainable from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/ansible/automation-hub/token)) as an environment variable, then install required collections:

```bash
# Export Red Hat Offline Token
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN="<YOUR_RH_OFFLINE_TOKEN>"
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_VALIDATED_TOKEN="<YOUR_RH_OFFLINE_TOKEN>"

# Install Ansible Galaxy Dependencies
ansible-galaxy collection install -r requirements.yml --force
```

---

### Step 3: Add Subscription Manifest File

Copy your Red Hat Subscription Manifest `.zip` file into the root of this repository and rename it to `manifest.zip`:

```bash
cp /path/to/your/manifest_*.zip ./manifest.zip
```

---

### Step 4: Export OpenShift Auth Token & Run Playbook

Export your active OpenShift session token, then run the `deploy_aap.yml` playbook:

```bash
# 1. Export current OpenShift auth token
export OCP_TOKEN=$(oc whoami -t)

# 2. Run the deployment playbook
ansible-playbook deploy_aap.yml -e '{"controller_validate_certs": false}'
```

> **Note on Initial Deployment:** On fresh cluster installs, the AAP operator will deploy pods in the `aap` namespace. If the initial run fails while waiting for secrets during Operator reconciliation, wait 1–2 minutes for pods to initialize (`oc get pods -n aap`) and re-run the `ansible-playbook` command above.

---

## Post-Deployment & Verification

### Step 1: Verify Pod Health

Check that all AAP operator and instance pods are in `Running` or `Completed` state:

```bash
oc get pods -n aap
```

---

### Step 2: Retrieve Passwords

* **AAP Gateway Web UI Password (Primary Entrypoint):**
  ```bash
  echo "Username: admin"
  echo "Password: $(oc get secret aap-admin-password -n aap -o jsonpath='{.data.password}' | base64 --decode)"
  ```

* **Direct Controller API Password:**
  ```bash
  echo "Username: admin"
  echo "Password: $(oc get secret aap-controller-admin-password -n aap -o jsonpath='{.data.password}' | base64 --decode)"
  ```

---

### Step 3: Display Access URLs

Display the generated OpenShift routes for the Gateway and Controller:

```bash
echo "AAP Gateway Web UI:  https://$(oc get route aap -n aap -o jsonpath='{.spec.host}')"
echo "AAP Controller API:  https://$(oc get route aap-controller -n aap -o jsonpath='{.spec.host}')"
```

---

## Troubleshooting

### Clearing / Resetting Deployment

To completely wipe the AAP instance and start clean:

```bash
# Delete AAP project
oc delete project aap

# Wait for namespace deletion
oc get ns aap --watch
```

Once the namespace is completely removed, re-run **Step 4** to trigger a fresh installation.
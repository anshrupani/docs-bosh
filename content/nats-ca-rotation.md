# Rotating NATS Certificate Authorities

The procedure below rotates the NATS CA and NATS related certificates across the director, health monitor, NATS server, and all the deployed VMs. It can be used whether the certificates are still valid or have already expired. See [Components of Bosh](bosh-components.md) for more information on core components.


### Preconditions

* Director is in a healthy state.
* All VMs are in `running` state in all deployments. See [below](#expired) if your VMs are unresponsive.
* Take note of any **ignored** VMs. They will be omitted from the VM redeploy steps.
* Director versions prior to 271.12 and stemcells prior to Bionic 1.36 and Windows 2019.41 need to recreate VMs as part of the redeploy steps.


### Step 1: Update the director, health monitor, and NATS server jobs, to introduce the new CA.

```shell
bosh create-env ~/workspace/bosh-deployment/bosh.yml \
 --state ./state.json \
 -o ~/workspace/bosh-deployment/[IAAS]/cpi.yml \
 -o add-new-ca.yml \
 -o ... additional opsfiles \
 --vars-store ./creds.yml \
 -v ... additional vars
```

* Adds new variables to generate the new NATS CA and the corresponding NATS related certificates signed by it. Please note that not all of these newly generated certificates will be used in this step, as some of them will be used in the following steps.
* The director and health monitor jobs are given two CA certificates to trust when communicating with the NATS server. This is done through the concatenation of the old and new NATS CAs: `((nats_server_tls.ca))((nats_server_tls_2.ca))`. This allows the director and health monitor to trust certificates presented by the NATS server that can be either signed by the new or old CAs. See [limitations](#limitations).
* The director and health monitor jobs are updated to use new client certificates that were generated by the new CA. These client certs are used for the Mutual TLS communication with the NATS server.
* The NATS server continues to use the old certificates (signed by old NATS CA) to serve TLS connections. NATS server is given the concatenated CAs from above to verify client certificates (for mTLS) signed by both old CA and new CA.
* Each VM/agent continues to use the old client certificates to communicate with the NATS server.

!!! warning
    In the below operations file `add-new-ca.yml`, the `nats_server_tls_2` certificate is generated with the `internal_ip` as the only Subject Alternative Name. Please remember to add any other SANs that maybe neccessary to your environment.

`add-new-ca.yml`

```yaml
---
- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/ca?
  value: ((nats_server_tls.ca))((nats_server_tls_2.ca))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/client_ca?
  value:
    certificate: ((nats_ca_2.certificate))
    private_key: ((nats_ca_2.private_key))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/director?
  value:
    certificate: ((nats_clients_director_tls_2.certificate))
    private_key: ((nats_clients_director_tls_2.private_key))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/health_monitor?
  value:
    certificate: ((nats_clients_health_monitor_tls_2.certificate))
    private_key: ((nats_clients_health_monitor_tls_2.private_key))

- type: replace
  path: /variables/-
  value:
    name: nats_ca_2
    type: certificate
    options:
      is_ca: true
      common_name: default.nats-ca.bosh-internal

- type: replace
  path: /variables/-
  value:
    name: nats_server_tls_2
    type: certificate
    options:
      ca: nats_ca_2
      common_name: default.nats.bosh-internal
      alternative_names: [((internal_ip))]
      extended_key_usage:
      - server_auth

- type: replace
  path: /variables/-
  value:
    name: nats_clients_director_tls_2
    type: certificate
    options:
      ca: nats_ca_2
      common_name: default.director.bosh-internal
      extended_key_usage:
      - client_auth

- type: replace
  path: /variables/-
  value:
    name: nats_clients_health_monitor_tls_2
    type: certificate
    options:
      ca: nats_ca_2
      common_name: default.hm.bosh-internal
      extended_key_usage:
      - client_auth

```

### Step 2: Redeploy all VMs, for each deployment.

Deployed VMs need to be redeployed in order to receive new client certificates that are signed by the new CA. Also, they will receive a new list of CAs (old and new CAs certs concatenated) to trust when communicating with the NATS server. This redeployment of the VMs is crucial for the NATS CA rotation.

### Step 3: Update the director, health monitor, and NATS server jobs, to remove references for the old NATS CA and certificates signed by it.

```shell
bosh create-env ~/workspace/bosh-deployment/bosh.yml \
 --state ./state.json \
 -o ~/workspace/bosh-deployment/[IAAS]/cpi.yml \
 -o remove-old-ca.yml \
 -o ... additional opsfiles \
 --vars-store ./creds.yml \
 -v ... additional vars
```

* `nats.tls.ca` property is updated to remove the old CA from the concatenated CAs.
* The director and health monitor continue to only use new client certificates (for mTLS) that were signed by the new NATS CA. Also, in this step the director and health monitor will start to **ONLY** trust NATS server certificates that were signed by the new CA.
* The NATS server is updated to use a new certificate (used to serve TLS connections) signed by the new NATS CA. Also, in this step the NATS server will start to **ONLY** trust client certificates (for mTLS) that were signed by the new CA.
* All components now communicate using the new CA.

`remove-old-ca.yml`

```yaml
---
- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/ca?
  value: ((nats_server_tls_2.ca))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/server?
  value:
    certificate: ((nats_server_tls_2.certificate))
    private_key: ((nats_server_tls_2.private_key))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/client_ca?
  value:
    certificate: ((nats_ca_2.certificate))
    private_key: ((nats_ca_2.private_key))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/director?
  value:
    certificate: ((nats_clients_director_tls_2.certificate))
    private_key: ((nats_clients_director_tls_2.private_key))

- type: replace
  path: /instance_groups/name=bosh/properties/nats/tls/health_monitor?
  value:
    certificate: ((nats_clients_health_monitor_tls_2.certificate))
    private_key: ((nats_clients_health_monitor_tls_2.private_key))

```


### Step 4: Redeploy all VMs, for each deployment.

Redeploying all VMs will remove the old NATS CA reference from their agent settings.

### Step 5: Clean-up

Operators are encouraged to clean up the credentials file after applying the aforementioned procedure, in order to prevent the old CA from returning in a subsequent `bosh create-env` in error. The following procedure will update the credentials store to replace the old certificate values with the new values generated.

```
# update_nats_var_values.yml

---
- type: replace
  path: /nats_ca
  value: ((nats_ca_2))

- type: replace
  path: /nats_clients_director_tls
  value: ((nats_clients_director_tls_2))

- type: replace
  path: /nats_clients_health_monitor_tls
  value: ((nats_clients_health_monitor_tls_2))

- type: replace
  path: /nats_server_tls
  value: ((nats_server_tls_2))

- type: remove
  path: /nats_ca_2

- type: remove
  path: /nats_clients_director_tls_2

- type: remove
  path: /nats_clients_health_monitor_tls_2

- type: remove
  path: /nats_server_tls_2

```

Create a backup of the current credentials and apply the opsfile:

```
cp creds.yml creds.yml.bkp

bosh interpolate creds.yml \
  -o update_nats_var_values.yml \
  --vars-file creds.yml > updated_creds.yml

mv updated_creds.yml creds.yml
```

**Do not** use the `add-new-ca.yml` and `remove-old-ca.yml` ops files in subsequent `bosh create-env` commands.

!!! warning
    **Warning:** If you do not perform the clean-up procedure, you must ensure that the ops files (`add-new-ca.yml` and `remove-old-ca.yml`) are used every time a create-env is executed going forward (which can be unsustainable). Removing the ops files would revert to the old CA, which can lead to unresponsive agents for existing and newly created VMs.

### Expired

If your deployment VMs are already in the state 'unresponsive agent', then the above procedure will not return the system to a healthy state. To replace a NATS CA that has already expired:

1. Open the file used for the `--vars-store` argument to `bosh create-env` (typically `creds.yml`) and remove all NATS-related variable **keys** and **values**: `nats_ca`, `nats_clients_director_tls`, `nats_clients_health_monitor_tls`, and `nats_server_tls`.
2. Update the director with new certs with `bosh create-env`. The CLI generates new values for the credentials removed in step 1.
3. Recreate all your deployments so they receive the new certificates with `bosh recreate -d ... --fix`. `--fix` is required to ignore the unresponsive state of the VM.

### Visualization of the Steps

![image](images/nats_rotation.png)

## Troubleshooting

NATS certificates may be expired if all `bosh deploy` tasks suddenly start failing. To confirm that the certificate is expired, you can use the OpenSSL utility:

```
bosh int /path/to/creds.yml --path /nats_server_tls/ca | openssl x509 -noout -dates
```

The procedure above will enable you to restore NATS communications.

NATS will not emit specific error messages related to certificate expiration, but requests will time out after 600 seconds.

# vm-deploy — cluster-scoped VirtualMachine example

End-to-end example that provisions a VM on vSphere via `provider-vspherevm`:
credentials Secret → `ProviderConfig` → `VirtualMachine`.

## Files

| File | Purpose |
|------|---------|
| `secret.yaml.tmpl` | vSphere credentials Secret (`crossplane-system/vsphere-creds`). |
| `providerconfig.yaml` | `ProviderConfig/default` pointing at the Secret. |
| `virtualmachine.yaml` | The VM (2 vCPU / 4 GiB / 40 GiB thin, cloned from a linux template). |

> The MOIDs in `virtualmachine.yaml` (`resgroup-*`, `datastore-*`, `dvportgroup-*`,
> `templateUuid`) are environment-specific — replace them with the IDs from your
> own vCenter inventory.

## ⚠️ Credential key gotcha

The Secret JSON key **must be `user`**, not `username` — `internal/clients/vsphere.go`
reads `creds["user"]`. A wrong key sends an *empty* user to the terraform vSphere
provider, and vCenter rejects it with the misleading:

```
ServerFaultCode: Cannot complete login due to an incorrect user name or password
```

If you hit that error, first prove the creds independently (REST
`POST /rest/com/vmware/cis/session` and/or a raw SOAP `Login`). If those succeed
but the provider still fails, the Secret key names are wrong — not the values.

## Deploy

```bash
# 1) credentials (fill in your own values)
export VSPHERE_USER='administrator@vsphere.local'
export VSPHERE_PASSWORD='...'
export VSPHERE_SERVER='vcenter.example.com'
envsubst < secret.yaml.tmpl | kubectl apply -f -

# 2) provider config + VM
kubectl apply -f providerconfig.yaml
kubectl apply -f virtualmachine.yaml
```

## Verify

```bash
kubectl get virtualmachine.virtualmachine.vspherevm.stuttgart-things.com crossplane-dev-vm
# SYNCED=True, READY=True (reason Available)

kubectl get virtualmachine.virtualmachine.vspherevm.stuttgart-things.com \
  crossplane-dev-vm -o jsonpath='{.status.atProvider.defaultIpAddress}{"\n"}'
# e.g. 10.0.0.42
```

`waitForGuestNetTimeout: 5` (minutes) makes the provider wait for guest networking
so the IP lands in `status.atProvider.defaultIpAddress`. Setting it to `-1`
disables the wait and the IP stays empty.

## Cleanup

```bash
kubectl delete -f virtualmachine.yaml   # deletionPolicy: Delete removes the vCenter VM
```

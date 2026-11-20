# Windows Server PAYG on OpenShift Container Platform for Google Cloud

This procedure runs Windows Server 2025 virtual machines (VMs) on dedicated
OpenShift Virtualization workers using Google Cloud pay-as-you-go (PAYG)
licensing. It creates a licensed copy of a Red Hat Enterprise Linux CoreOS
(RHCOS) boot image and provisions **dedicated C3-metal MachineSets** from that
image. VMs select these workers by Node label and tolerate their `NoSchedule`
taint. Use an existing, compatible Windows Server 2025 Datacenter boot source.

> **Licensing assumption:** This procedure assumes Google Cloud accepts a
> copy of the current RHCOS image with its existing licenses (if any) plus
> `windows-server-2025-dc`. Confirm the licensing arrangement with Google
> Cloud and verify guest activation before running production workloads.
> Successful image creation alone does not establish that licensing or
> activation is supported.

## Prerequisites

- An OpenShift Container Platform (OCP) cluster on Google Cloud with
  OpenShift Virtualization installed and C3-metal capacity available in the
  chosen zones. Check the [OCP Virtualization installation
  requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/virtualization/installing)
  for your cluster version.
- `oc`, `gcloud`, and `jq`; cluster-admin access to manage MachineSets and
  `MachineConfiguration/cluster`; and GCP permissions to inspect and create
  Compute Engine images.
- Existing **ordinary** GCP worker MachineSets to use as read-only sources
  for the current RHCOS images and cluster-specific networks, credentials,
  user data, service accounts, and zones. Confirm the C3-metal configuration
  requirements before creating the new MachineSets; no pre-existing C3-metal
  MachineSet or worker is required.
- A Hyperdisk-backed StorageClass suitable for the Windows guest and a
  Windows Server 2025 Datacenter VM boot source available as a CDI DataSource
  in the guest's namespace. Preparing that source is a separate, generic
  OpenShift Virtualization task.
- Available C3-metal quota/capacity. PAYG licensing is associated with the
  worker boot disk and has billing implications even when no guest is running.

Use the **current worker MachineSet in each zone** as a read-only template.
Run steps 1, 2, and 4 once for **each ordinary worker MachineSet**, creating
one dedicated MachineSet alongside each source. Steps 3, 5, and 7 are
cluster-wide or per-guest checks as indicated.
Run commands from a shell with the intended `oc` cluster context and `gcloud`
credentials.

```bash
oc whoami
oc config current-context
gcloud config get-value project

export GCP_PROJECT="YOUR_GCP_PROJECT_ID"
export CLUSTER_ID="$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')"
export IMAGE_REV="$(date +%Y%m%d)" # Change for each new image version.
export MACHINE_TYPE=c3-standard-192-metal
export WINDOWS_LICENSE="https://www.googleapis.com/compute/v1/projects/windows-cloud/global/licenses/windows-server-2025-dc"
```

List the ordinary worker MachineSets belonging to the cluster:

```bash
oc get machinesets -n openshift-machine-api \
  -l "machine.openshift.io/cluster-api-cluster=$CLUSTER_ID" -o json |
  jq -r '.items[]
         | select(.spec.template.metadata.labels["machine.openshift.io/cluster-api-machine-role"] == "worker")
         | select(.metadata.labels["workload.example.com/windows-payg"] != "true")
         | .metadata.name'
```

For each listed name, set `SOURCE_MS` and follow the per-MachineSet steps.
For example, given `ci-op-8g9332r9-343b5-xzsvv-worker-a`, these commands
produce `ci-op-8g9332r9-343b5-xzsvv-windows-c3-metal-payg-a`:

```bash
export SOURCE_MS=ci-op-8g9332r9-343b5-xzsvv-worker-a
[[ "$SOURCE_MS" == *-worker-* ]] ||
  { echo "Choose a unique NEW_MS for this source MachineSet" >&2; exit 1; }

export NEW_MS="${SOURCE_MS/-worker-/-windows-c3-metal-payg-}"
export LICENSED_IMAGE="${SOURCE_MS}-payg-${IMAGE_REV}"
export MS_FILE="${NEW_MS}.json"

# Leave room for the random suffix added to each Machine/instance name.
(( ${#NEW_MS} <= 57 && ${#LICENSED_IMAGE} <= 63 )) ||
  { echo "Shorten the MachineSet name or image revision" >&2; exit 1; }
echo "Read-only source: $SOURCE_MS; new MachineSet: $NEW_MS"
```

The new MachineSet keeps the source's
`machine.openshift.io/cluster-api-cluster` label, zone, and worker role.
If the cluster uses a different naming convention, choose a unique `NEW_MS`
manually rather than relying on the substitution above.

## 1. Select the source RHCOS image

Read the **boot** disk image from an ordinary worker MachineSet. It supplies
a reference configuration only; leave it and its workers unchanged. Do not
use a Windows GCE image to boot the OpenShift worker.

```bash
oc get machineset "$SOURCE_MS" -n openshift-machine-api -o json |
  jq '{name: .metadata.name,
       zone: .spec.template.spec.providerSpec.value.zone,
       machineType: .spec.template.spec.providerSpec.value.machineType,
       disks: .spec.template.spec.providerSpec.value.disks}'

SOURCE_MS_JSON=$(oc get machineset "$SOURCE_MS" -n openshift-machine-api -o json)
ZONE=$(jq -r '.spec.template.spec.providerSpec.value.zone' <<<"$SOURCE_MS_JSON")
SOURCE_IMAGE=$(jq -r '.spec.template.spec.providerSpec.value.disks[]
                      | select(.boot == true) | .image' <<<"$SOURCE_MS_JSON")
echo "Zone: $ZONE; boot image: $SOURCE_IMAGE"
```

Confirm that the source MachineSet belongs to the intended cluster and its
zone supports the selected C3-metal type. The new MachineSet will specify
that machine type and a compatible Hyperdisk boot disk. The commands below
expect an image reference of the form
`projects/PROJECT/global/images/IMAGE`:

```bash
[[ "$SOURCE_IMAGE" =~ ^projects/([^/]+)/global/images/([^/]+)$ ]] ||
  { echo "Unexpected source image: $SOURCE_IMAGE" >&2; exit 1; }

SOURCE_PROJECT=${BASH_REMATCH[1]}
SOURCE_NAME=${BASH_REMATCH[2]}

gcloud compute images describe "$SOURCE_NAME" \
  --project="$SOURCE_PROJECT" \
  --format='yaml(name,licenses,status)'
```

Record the OpenShift release/RHCOS version associated with this image. It is
needed when configuring manual boot-image skew enforcement and during future
updates.

## 2. Copy the RHCOS image with the PAYG license

Include **all licenses on the source image, if any**, as well as the Windows
license. Preserve the source image's existing licenses.

```bash
LICENSES=$(
  {
    gcloud compute images describe "$SOURCE_NAME" \
      --project="$SOURCE_PROJECT" --format=json |
      jq -r '.licenses[]?'
    printf '%s\n' "$WINDOWS_LICENSE"
  } | sort -u | paste -sd, -
)

gcloud compute images create "$LICENSED_IMAGE" \
  --project="$GCP_PROJECT" \
  --source-image="$SOURCE_NAME" \
  --source-image-project="$SOURCE_PROJECT" \
  --licenses="$LICENSES"

gcloud compute images describe "$LICENSED_IMAGE" \
  --project="$GCP_PROJECT" \
  --format='yaml(name,licenses,status)'
```

**Stop here** if the copy fails, any existing source-image license is missing,
or the Windows license is missing. Image licenses still need to be verified on
an actual *new worker boot disk* after provisioning.

## 3. Exclude only the dedicated MachineSets from boot-image management

This configuration belongs to the cluster-scoped **`MachineConfiguration`
resource named `cluster`**, not a `MachineConfig` or `MachineConfigPool`.
Use `Partial` selection so that the Machine Config Operator (MCO) continues
updating **every ordinary MachineSet**, while omitting MachineSets with a
specific label on the *MachineSet object itself*.

First inspect the current configuration:

```bash
oc get machineconfiguration cluster -o yaml
oc get machinesets -n openshift-machine-api --show-labels
```

If `.status.bootImageSkewEnforcementStatus.mode` is `Automatic`, the API
requires `All` MachineSets to be managed. Change skew enforcement to `Manual`
**first**, before changing MachineSet selection to `Partial`. This is a
**cluster-wide** setting. The manual version must describe the **oldest boot
image still used by any relevant MachineSet**, including the new licensed
image; it is not necessarily the current cluster version. For example, edit
`oc edit machineconfiguration cluster` to add:

```yaml
spec:
  bootImageSkewEnforcement:
    mode: Manual
    manual:
      mode: OCPVersion
      ocpVersion: "<release-of-oldest-boot-image>"
```

Keep all other existing fields. Wait until the status reports `Manual`:

```bash
oc get machineconfiguration cluster \
  -o jsonpath='{.status.bootImageSkewEnforcementStatus.mode}{"\n"}'
```

Then use `oc edit machineconfiguration cluster` to set the
`machine.openshift.io` / `machinesets` manager to this selection:

```yaml
spec:
  managedBootImages:
    machineManagers:
      - apiGroup: machine.openshift.io
        resource: machinesets
        selection:
          mode: Partial
          partial:
            machineResourceSelector:
              matchExpressions:
                - key: workload.example.com/windows-payg
                  operator: DoesNotExist
```

**Preserve any other `machineManagers` entries** already present. Only one
entry per API group/resource pair is permitted. This example includes all
MachineSets **without** the marker, including future ordinary MachineSets.
It excludes the dedicated MachineSets as long as their own metadata retains
the marker. Check the accepted configuration in
`.status.managedBootImagesStatus` and review the MCO conditions:

```bash
oc get machineconfiguration cluster \
  -o jsonpath='{.status.managedBootImagesStatus}{"\n"}'
oc get co machine-config
```

Apply this policy **before** creating the dedicated MachineSets, and include
their exclusion label in their creation manifests. Do not briefly create them
unlabeled: they could otherwise be selected for an automatic image update.

## 4. Create the dedicated, labeled C3-metal MachineSet

Use **this source worker MachineSet** as a read-only starting point for its
**one new, separately named MachineSet in the same zone**. Its replicas and
workers stay unchanged. Set the new MachineSet's type and boot disk to the
required C3-metal configuration. The two label locations serve different
purposes:

- `metadata.labels` labels the **MachineSet**, excluding it from MCO boot
  image management.
- `spec.template.spec.metadata.labels` labels the **Node** created by each
  Machine, so Windows VMs can select it.
- `spec.template.spec.taints` applies a `NoSchedule` taint to each Node so
  ordinary workloads cannot consume licensed-worker capacity.

```bash
oc get machineset "$SOURCE_MS" -n openshift-machine-api -o json |
  jq --arg name "$NEW_MS" \
     --arg machineType "$MACHINE_TYPE" \
     --arg image "projects/$GCP_PROJECT/global/images/$LICENSED_IMAGE" '
    .metadata = {
      name: $name,
      namespace: "openshift-machine-api",
      labels: ((.metadata.labels // {}) +
               {"workload.example.com/windows-payg": "true"})
    }
    | del(.status)
    | .spec.replicas = 1
    | .spec.selector.matchLabels["machine.openshift.io/cluster-api-machineset"] = $name
    | .spec.template.metadata.labels["machine.openshift.io/cluster-api-machineset"] = $name
    | .spec.template.spec.providerSpec.value.machineType = $machineType
    | .spec.template.spec.providerSpec.value.onHostMaintenance = "Terminate"
    | del(.spec.template.spec.providerSpec.value.shieldedInstanceConfig.secureBoot)
    | .spec.template.spec.metadata.labels =
        ((.spec.template.spec.metadata.labels // {}) +
         {"workload.example.com/windows-payg": "true"})
    | .spec.template.spec.taints =
        ((.spec.template.spec.taints // []) +
         [{"key": "workload.example.com/windows-payg",
           "value": "true",
           "effect": "NoSchedule"}])
    | (.spec.template.spec.providerSpec.value.disks[]
       | select(.boot == true) | .type) = "hyperdisk-balanced"
    | (.spec.template.spec.providerSpec.value.disks[]
       | select(.boot == true) | .image) = $image
  ' > "$MS_FILE"
```

Review `$MS_FILE` before applying it. In particular,
verify the inherited zone and cluster ID, selected `c3-*-metal` machine type,
Hyperdisk boot disk type and capacity, `autoDelete`, the `Terminate` host
maintenance policy, network, service account, user data, worker role,
**both** label locations, and the `workload.example.com/windows-payg`
`NoSchedule` taint. Check inherited taints too: the VM needs tolerations for
any that remain. The example removes a source MachineSet's Secure Boot
setting because Shielded VM Secure Boot is not supported for GCP bare-metal
workers; verify the resulting `shieldedInstanceConfig` against your cluster
policy. Confirm any further C3-metal-specific requirements before applying
the manifest.

```bash
oc apply -f "$MS_FILE"
oc get machineset "$NEW_MS" -n openshift-machine-api
oc get nodes -l workload.example.com/windows-payg=true -o wide
oc get nodes -l workload.example.com/windows-payg=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.taints}{"\n"}{end}'
```

Verify the GCE boot disk of **each new worker**, not just the custom image:

```bash
export WORKER_INSTANCE="NEW_WORKER_GCE_INSTANCE_NAME"
gcloud compute instances describe "$WORKER_INSTANCE" \
  --project="$GCP_PROJECT" --zone="$ZONE" \
  --format='yaml(disks[].boot,disks[].licenses,disks[].autoDelete)'
```

Expect **all licenses on the source image (if any)** and
`windows-server-2025-dc`. Do not schedule licensed guests on an unverified
new worker. Recheck licenses whenever this dedicated MachineSet subsequently
provisions another worker.

Before deploying VMs, check that OpenShift Virtualization's node-specific
components (in particular `virt-handler`) and the GCP storage CSI components
needed on the workers can tolerate the taint and become healthy. For example,
inspect `oc -n openshift-cnv get daemonset virt-handler -o yaml` and the
relevant storage DaemonSets. Configure any necessary placement/toleration
changes for these components. Do **not** remove the taint just to make an
unprepared component schedule.

## 5. Check guest-to-KMS connectivity

Check Private Google Access on the worker subnet, and ensure outbound DNS and
TCP port 1688 to `kms.windows.googlecloud.com` are permitted. For example:

```bash
export WORKER_SUBNET="YOUR_WORKER_SUBNET"
export GCP_REGION="YOUR_REGION"

gcloud compute networks subnets describe "$WORKER_SUBNET" \
  --region="$GCP_REGION" --project="$GCP_PROJECT" \
  --format='value(privateIpGoogleAccess)'
```

If it is `False`, arrange for Private Google Access to be enabled through the
cluster's approved networking process. Test connectivity **inside a Windows
guest** in the next step. A successful TCP connection does not prove that
activation succeeded or that the KMS request is attributed to the licensed
worker. If activation fails, work with the network owners to check NAT,
routing, firewall rules, and whether KMS sees traffic from the licensed
worker.

## 6. Create and place a Windows VM

Use the normal OpenShift Virtualization VM workflow and an existing,
appropriately prepared Windows Server 2025 Datacenter boot source. The
DataSource, in this example, must be available **in the VM namespace**; using
one in another namespace requires the usual CDI clone permissions. Choose the
Hyperdisk StorageClass for its root disk.

Before starting the VM, ensure its `spec.template.spec` has a Node selector
and toleration for the licensed workers and uses the pod network with
masquerade:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        workload.example.com/windows-payg: "true"
      tolerations:
        - key: workload.example.com/windows-payg
          operator: Equal
          value: "true"
          effect: NoSchedule
      evictionStrategy: None
      networks:
        - name: default
          pod: {}
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
```

These fields are **part of the VM manifest**, not a complete VM manifest:
retain the boot source's appropriate disks, EFI/Windows settings, CPU, and
memory. The selector keeps the VM on these workers; the toleration permits it
past their taint. Toleration alone does **not** restrict a workload to these
workers, and any other workload granted this toleration could also run there.
Use admission/RBAC controls if scheduling must be strictly limited to
authorized Windows workloads. For the
single-node `ReadWriteOnce` example, `evictionStrategy: None` avoids assuming
live migration is available. HA or live migration requires appropriately
licensed destinations and migration-capable storage.

Verify the VM is running on a labeled, license-verified worker:

```bash
oc get vmi -n <windows-vm-namespace> -o wide
```

## 7. Activate Windows and verify the result

Open an Administrator PowerShell session in the VM:

```powershell
DISM /online /Get-CurrentEdition
Test-NetConnection kms.windows.googlecloud.com -Port 1688
```

The edition must be `ServerDatacenter` before activation. If using a
`ServerDatacenterEval` boot source, follow Microsoft's edition-conversion
procedure and reboot first. Use the appropriate published Windows Server 2025
Datacenter [KMS client key](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys):

```powershell
cscript //nologo C:\Windows\System32\slmgr.vbs /ipk D764K-2NDRG-47T6Q-P8T8W-YP6DF
cscript //nologo C:\Windows\System32\slmgr.vbs /skms kms.windows.googlecloud.com:1688
cscript //nologo C:\Windows\System32\slmgr.vbs /ato
cscript //nologo C:\Windows\System32\slmgr.vbs /dli
```

The final check is **`License Status: Licensed`**, on a VM running on a
verified PAYG worker. KMS renewal must remain possible while it runs.

## 8. Maintain licensed boot images after cluster upgrades

The special MachineSets are **excluded from automatic boot-image updates**,
not from in-place RHCOS updates on their running workers. A MachineSet's boot
image is used when it creates a **future** worker, for example when the pool
scales out or recovers from a failure. Keeping the licensed image current
does not require replacing the workers created during initial setup.

After an ordinary worker MachineSet has received its updated boot image,
repeat the following for its **corresponding** dedicated MachineSet:

1. Read that MachineSet's **new** boot image reference as in step 1. Set a
   new `IMAGE_REV` and `LICENSED_IMAGE` for this version; record its
   associated OpenShift release/RHCOS version.
2. Create a **new, versioned** copy using step 2, carrying forward any
   licenses on the source image and adding the Windows PAYG license.
   Verify both licenses on the new image.
3. Update the boot disk `image` reference on the corresponding dedicated
   MachineSet.
   Keep its **MachineSet exclusion label** and Node placement label. This
   changes the image used by **subsequently created workers**; it does not
   reimage or replace running workers.
4. Verify both licenses on the next worker provisioned by the dedicated
   MachineSet (for example during a planned scale-out), then test guest
   placement and activation there. Plan any intentional node rotation
   separately around VM availability, storage, and metal capacity.
5. Advance the cluster-wide manual skew-enforcement version only when it
   accurately describes the **oldest boot image still in use**. Do not set
   it to the newest ordinary worker image while licensed MachineSets still
   reference an older one.

Perform boot-image refresh and skew checks *before* subsequent upgrades. Do
not delete an image while a MachineSet still needs it to provision new
workers.

## 9. Off-ramp

Stop or relocate Windows VMs before reducing the dedicated MachineSet
replicas. Verify that removed GCE workers' **licensed boot disks were
deleted**; a retained boot disk can continue to incur license charges. Do not
remove the `Partial` exclusion while any special MachineSet still uses a
licensed custom image. Reassess the cluster-wide skew configuration when no
manually maintained boot images remain.

## References

- [Google Cloud: manage licenses](https://docs.cloud.google.com/compute/docs/licenses/manage)
- [Google Cloud: license changes and restrictions](https://docs.cloud.google.com/compute/docs/licenses/license-changes-and-restrictions)
- [OpenShift: boot-image management](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/machine_configuration/mco-update-boot-images)
- [OpenShift: `MachineConfiguration` API](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/operator_apis/machineconfiguration-operator-openshift-io-v1)
- [Windows PAYG activation reference](https://cloud.redhat.com/experts/osd/gcp-windows-payg-activation/)

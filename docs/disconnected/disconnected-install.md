# Disconnected Installation Guide

This guide covers installing OSAC on OpenShift clusters that operate in
disconnected (air-gapped) or restricted-network environments where direct
access to the default Red Hat operator catalogs is unavailable.

## Overview

OSAC depends on several operators delivered through the Operator Lifecycle
Manager (OLM). In a connected environment the `osac-deps` and the `osac-infra` Helm chart
installs OLM `Subscription` resources. These OLM `Subscription` resources reference the default
`redhat-operators` CatalogSource in the `openshift-marketplace` namespace.

### Before you start

You must be aware about the [Disconnected Environment Terms](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/disconnected_environments/about-disconnected-environments).

This document is validated against a restricted environment.

OSAC installation is configurable to provide core components in conjunction with
rest of the services like VMaaS,BMaaS,CaaS etc. Depending upon your choice of
services, toggling and customization of Helm values must be done.

For Ansible Automation Platform, obtain a subscription manifest in the
Subscription Allocations section of Red Hat Subscription Management.  This 
document will refer to it by license.zip file.

Even for a disconnected OpenShift installation, you must obtain and use
a pull secret from Red Hat Cloud along with the pull secret to pull from the
openshift mirror registry.

#### Example networking setup

```mermaid
flowchart TB
    subgraph HOST["HOST: sno.on.baremetal.redhat.com (10.6.76.15)"]
        direction TB
        OCP_LABEL["OCP 4.22 Single-Node OpenShift Host"]

        subgraph BRIDGE["Linux Bridge: br-sno (192.168.100.1/24)<br/><i>Software-only — NO physical NIC slaved</i>"]
            direction LR

            subgraph BASTION["bastion-vm"]
                direction TB
                B_ETH1["eth1: 192.168.100.2<br/>(bridge: br-sno)"]
                B_ETH0["eth0: 10.0.2.x<br/>(masq/NAT)"]
            end

            subgraph REGISTRY["registry-vm"]
                direction TB
                R_ETH1["eth1: 192.168.100.3<br/>(bridge: br-sno)"]
                R_ETH0["eth0: 10.0.2.x<br/>(masq/NAT)"]
            end

            subgraph OSAC["osac-sno-vm"]
                direction TB
                O_ETH0["eth0: 192.168.100.10<br/>MAC: 52:54:00:10:09:83<br/>(bridge: br-sno)"]
                O_NOTE["[ Single NIC Only ]"]
            end
        end

        subgraph K8S_NET["Cluster & Virtualization Networking Overview"]
            direction LR
            NAD["<b>NAD: br-lab-nad</b><br/>Bridge CNI on br-sno<br/><code>macspoofchk: false</code> | <code>promiscMode: true</code>"]
            OVN["<b>OVN Pod Network</b><br/>10.128.0.0/14<br/>(virt-launcher pods)"]
            SVC["<b>ClusterIP Network</b><br/>172.30.0.0/16<br/>(K8s Services)"]
        end
    end

    %% Styling
    style HOST fill:#1e1e2e,stroke:#89b4fa,stroke-width:2px,color:#cdd6f4
    style BRIDGE fill:#313244,stroke:#a6adc8,stroke-width:1px,color:#cdd6f4
    style BASTION fill:#45475a,stroke:#f9e2af,stroke-width:1px,color:#cdd6f4
    style REGISTRY fill:#45475a,stroke:#f9e2af,stroke-width:1px,color:#cdd6f4
    style OSAC fill:#45475a,stroke:#a6e3a1,stroke-width:1px,color:#cdd6f4
    style K8S_NET fill:#313244,stroke:#cba6f7,stroke-width:1px,color:#cdd6f4
    style OCP_LABEL fill:none,stroke:none,color:#94e2d5
```

In a disconnected environment , these mandatory checks must pass

1. Successfully mirror all the container images.
2. CRIO daemon and kubelet on the osac vm must be able to resolve the openshift mirror registry.


## Required Operator Packages

The following operator packages must be available in the mirrored catalog:

| Operator | Package Name | Default Channel |
|----------|-------------|-----------------|
| Ansible Automation Platform | `ansible-automation-platform-operator` | `stable-2.6-cluster-scoped` |
| AMQ Streams (Kafka) | `amq-streams` | `stable` |
| cert-manager | `openshift-cert-manager-operator` | `stable-v1` |
| OpenShift Virtualization (CNV) | `kubevirt-hyperconverged` | `stable` |
| LVM Storage | `lvms-operator` | `stable-4.22` |
| Multicluster Engine | `multicluster-engine` | `stable-2.17` |
| MetalLB | `metallb-operator` | `stable` |

> **Note:** Not every operator is required for every deployment. Only the
> operators enabled in your `values.yaml` need to be present in the
> mirrored catalog.

## Mirroring with oc-mirror

Use `oc-mirror` to build a mirror of the required Openshift catalogs,
Red Hat container images & non Red Hat Container images as well as
OSAC container images . Below is a working  `ImageSetConfiguration` tested in a
disconnected environment.
>**Note**:At the time of testing, at the least 400GiB storage space is required
>for this mirrored registry.

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
archiveSize: 8
mirror:
  platform:
    architectures:
      - amd64
    channels:
      - name: stable-4.22
        minVersion: 4.22.6
        maxVersion: 4.22.6
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.22
      packages:
        - name: kubevirt-hyperconverged
          channels:
            - name: stable
        - name: lvms-operator
          channels:
            - name: stable-4.22
        - name: metallb-operator
          channels:
            - name: stable
        - name: ansible-automation-platform-operator
          defaultChannel: stable-2.6-cluster-scoped
          channels:
            - name: stable-2.6-cluster-scoped
              maxVersion: '2.6.0+0.1787258256' # osac-installer/docs/cve-2026-75884-security-exception.md
        - name: openshift-cert-manager-operator
          channels:
            - name: stable-v1
        - name: amq-streams
          channels:
            - name: stable
  additionalImages:
    # --- osac-deps and osac-infra. ---
    - name: ghcr.io/openbao/openbao:2.6.2 # Bundled OpenBao (Vault-compatible) secret store for dev/CI environments.
    - name: quay.io/jetstack/trust-manager:v0.20.0 # Helm chart osac-deps trust-manager
    - name: quay.io/jetstack/trust-pkg-debian-bookworm:20230311-deb12u1.1 # Helm chart osac-deps trust-manager
    - name: quay.io/keycloak/keycloak:26.6.4 # Helm chart osac-deps keycloak
    - name: quay.io/openshift/origin-cli:4.22 # Consumed by OSAC dependencies
    - name: quay.io/sclorg/postgresql-18-c10s@sha256:6be2c9d855f06fb665257a6b0911676a38d740be7022cc61acee1c99a832b1b2 # Helm osac-deps bundled-postgres
    # --- OSAC core components ---
    - name: ghcr.io/osac-project/osac-operator:0.0.18
    - name: ghcr.io/osac-project/fulfillment-service:0.0.111
    - name: ghcr.io/osac-project/envoy:v1.33.0
    - name: ghcr.io/osac-project/osac-aap:0.0.19
    - name: ghcr.io/osac-project/osac-ui:0.0.10
    - name: ghcr.io/osac-project/bare-metal-fulfillment-operator:0.0.18
    # --- OSAC metering (conditional: metering.enabled) ---
    - name: ghcr.io/osac-project/metering-service:0.0.7
    - name: ghcr.io/osac-project/metering-echo-adapter:0.0.7
    - name: ghcr.io/osac-project/metering-m360-adapter:0.0.7
    # --- OSAC CSI driver (conditional: csiDriver.enabled) ---
    - name: ghcr.io/osac-project/osac-csi-driver:0.0.7
    # --- CSI sidecars (used by csi-driver controller and node daemonset) ---
    - name: registry.k8s.io/sig-storage/csi-provisioner:v5.1.0
    - name: registry.k8s.io/sig-storage/csi-attacher:v4.7.0
    - name: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.12.0

```

Run the mirror:

```bash
oc-mirror --v2 --config=imageset-config.yaml --workspace \
file://$(pwd)/oc-mirror-workspace docker://registry.example.com:8443/
```
>**Tip**: In dev setup, using `-dest-tls-verify=false   --parallel-images 1   --parallel-layers 1`
> avoids TLS certification and registry db lockup issues.

Expected Result:

```bash
....
481 / 481 (1h7m41s) [====================================================] 100 %
2026/10/01 01:53:58  [INFO]   : === Results ===
2026/10/01 01:53:58  [INFO]   :  ✓  192 / 192 release images mirrored successfully
2026/10/01 01:53:58  [INFO]   :  ✓  266 / 266 operator images mirrored successfully
2026/10/01 01:53:58  [INFO]   :  ✗  23 / 23 additional images mirrored:
2026/10/01 01:53:58  [INFO]   : Generating pinned configurations
2026/10/01 01:53:58  [INFO]   : Pinned ISC written to: $(pwd)/oc-mirror-workspace/working-dir/isc_pinned_2026-10-01T05:53:58Z.yaml
2026/10/01 01:53:58  [INFO]   : Pinned DISC written to: $(pwd)/oc-mirror-workspace/working-dir/disc_pinned_2026-10-01T05:53:58Z.yaml
2026/10/01 01:53:58  [INFO]   : Generating cluster resources
2026/10/01 01:53:58  [INFO]   : Generating IDMS file
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/idms-oc-mirror.yaml file created
2026/10/01 01:53:58  [INFO]   : Generating ITMS file
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/itms-oc-mirror.yaml file created
2026/10/01 01:53:58  [INFO]   : Pinned DISC written to: $(pwd)/oc-mirror-workspace/working-dir/disc_pinned_2026-10-01T05:53:58Z.yaml
2026/10/01 01:53:58  [INFO]   : 📄 Generating cluster resources...
2026/10/01 01:53:58  [INFO]   : 📄 Generating IDMS file...
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/idms-oc-mirror.yaml file created
2026/10/01 01:53:58  [INFO]   : 📄 Generating ITMS file...
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/itms-oc-mirror.yaml file created
2026/10/01 01:53:58  [INFO]   : 📄 Generating CatalogSource file...
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/cs-redhat-operator-index-v4-22.yaml file created
2026/10/01 01:53:58  [INFO]   : 📄 Generating ClusterCatalog file...
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/cc-redhat-operator-index-v4-22.yaml file created
2026/10/01 01:53:58  [INFO]   : 📄 Generating Signature Configmap...
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/signature-configmap.json file created
2026/10/01 01:53:58  [INFO]   : $(pwd)/oc-mirror-workspace/working-dir/cluster-resources/signature-configmap.yaml file created
2026/10/01 01:53:58  [INFO]   : mirror time     : 1h8m8.261079897s
2026/10/01 01:53:58  [INFO]   : 👋 Goodbye, thank you for using oc-mirror

```

As a OSAC CSA, you must be **authorized** to apply the generated IDMS 
`ImageDigestMirrorSet` file, ITMS(`ImageTagMirrorSet`) file, `CatalogSource` file,
`ClusterCatalog` file and Signature Configmap.

Dependency relationship between ImageSetConfiguration, OSAC Deps,OSAC Infra & OSAC.
Assuming OSAC v0.0.21.

```mermaid
flowchart TB
    classDef iscStyle fill:#1565c0,color:#fff,stroke:#0d47a1,stroke-width:2px
    classDef depsStyle fill:#e65100,color:#fff,stroke:#bf360c
    classDef infraStyle fill:#2e7d32,color:#fff,stroke:#1b5e20
    classDef osacStyle fill:#6a1b9a,color:#fff,stroke:#4a148c
    classDef imgStyle fill:#37474f,color:#e0e0e0,stroke:#263238
    classDef secStyle fill:#455a64,color:#e0e0e0,stroke:#37474f

    ISC(["ImageSetConfiguration<br/>mirror.openshift.io/v2alpha1"]):::iscStyle

    ISC -->|"platform"| PLAT["OpenShift 4.22.6 amd64"]

    ISC -->|"operators"| OPCAT

    subgraph OPCAT["Operators — redhat-operator-index:v4.22"]
        OP1["openshift-cert-manager-operator<br/>channel: stable-v1"]
        OP2["ansible-automation-platform-operator<br/>channel: stable-2.6-cluster-scoped<br/>maxVersion: 2.6.0"]
        OP3["lvms-operator<br/>channel: stable-4.22"]
        OP4["metallb-operator<br/>channel: stable"]
        OP5["kubevirt-hyperconverged<br/>channel: stable"]
        OP6["amq-streams<br/>channel: stable"]
    end

    ISC -->|"additionalImages"| ADDIMG

    subgraph ADDIMG["Additional Images"]
        subgraph DIMGS["osac-deps / osac-infra"]
            I_TM["trust-manager:v0.20.0<br/>+ trust-pkg-debian-bookworm"]:::imgStyle
            I_KC["keycloak:26.6.4"]:::imgStyle
            I_PG["postgresql-18-c10s"]:::imgStyle
            I_OB["openbao:2.6.2"]:::imgStyle
            I_CL["origin-cli:4.22"]:::imgStyle
        end
        subgraph CIMGS["OSAC Core"]
            I_OP["osac-operator:0.0.18"]:::imgStyle
            I_FS["fulfillment-service:0.0.111"]:::imgStyle
            I_EN["envoy:v1.33.0"]:::imgStyle
            I_AA["osac-aap:0.0.19"]:::imgStyle
            I_UI["osac-ui:0.0.10"]:::imgStyle
            I_BM["bare-metal-fulfillment-operator:0.0.18"]:::imgStyle
        end
        subgraph MOIMGS["Metering + CSI — conditional"]
            I_MET["metering-service · echo-adapter · m360-adapter :0.0.7"]:::imgStyle
            I_CSI["osac-csi-driver:0.0.7<br/>csi-provisioner · attacher · registrar"]:::imgStyle
        end
    end

    OPCAT ==>|"OLM Subscriptions"| DEPS
    I_TM -->|"vendored chart image"| DEPS
    I_CL -->|"hook jobs"| DEPS

    subgraph DEPS["osac-deps — Phase 1a: Install Operators & Register CRDs"]
        D_CM["cert-manager Operator<br/>→ Certificate, ClusterIssuer CRDs"]:::depsStyle
        D_AAP["AAP Operator<br/>→ AutomationController CRDs"]:::depsStyle
        D_LVMS["LVMS Operator<br/>→ LVMCluster CRD"]:::depsStyle
        D_MLB["MetalLB Operator<br/>→ IPAddressPool, L2Adv CRDs"]:::depsStyle
        D_CNV["CNV Operator<br/>→ HyperConverged CRD"]:::depsStyle
        D_AMQ["AMQ Streams<br/>→ Kafka CRD"]:::depsStyle
        D_TM["trust-manager vendored<br/>→ Bundle CRD"]:::depsStyle
    end

    DEPS ==>|"CRDs now available<br/>on the cluster"| INFRA
    I_KC -->|"image"| INFRA
    I_PG -->|"image"| INFRA
    I_OB -->|"image"| INFRA
    I_CL -->|"hook jobs"| INFRA
subgraph INFRA["osac-infra — Phase 1b: Create CRD Instances & Shared Services"]
        subgraph TLS["TLS and Trust"]
            CI["ClusterIssuer<br/>default-ca"]:::infraStyle
            CERTS["CA Certificates"]:::infraStyle
            BUN["trust-manager Bundle"]:::infraStyle
            CABUN["ca-bundle ConfigMap"]:::infraStyle
        end
        subgraph SVC["Shared Services"]
            KCL["Keycloak<br/>+ osac realm"]:::infraStyle
            PGS["Bundled PostgreSQL<br/>dev/CI only"]:::infraStyle
            OBV["OpenBao / Vault<br/>dev/CI only"]:::infraStyle
        end
        subgraph CRS["Operand Custom Resources"]
            LVC["LVMCluster"]:::infraStyle
            HCV["HyperConverged"]:::infraStyle
            MBP["MetalLB IPAddressPool<br/>+ L2Advertisement"]:::infraStyle
            KFK["Kafka"]:::infraStyle
        end
        subgraph SECS["Shared Secrets"]
            FCC["fulfillment-controller-credentials"]:::secStyle
            KCS["keycloak-client-secrets"]:::secStyle
        end
    end

    D_CM -.->|"CRDs"| CI
    D_CM -.->|"CRDs"| CERTS
    D_TM -.->|"CRD"| BUN
    D_LVMS -.->|"CRD"| LVC
    D_CNV -.->|"CRD"| HCV
    D_MLB -.->|"CRD"| MBP
    D_AMQ -.->|"CRD"| KFK

    INFRA ==>|"secrets, certificates<br/>endpoints, CRs"| PHASE2
    CIMGS ==>|"core images"| PHASE2
    MOIMGS -.->|"conditional"| PHASE2

    subgraph PHASE2["osac — Phase 2: OSAC Platform"]
        PP1["osac-operator"]:::osacStyle
        PP2["fulfillment-service + envoy"]:::osacStyle
        PP3["osac-aap"]:::osacStyle
        PP4["osac-ui"]:::osacStyle
        PP5["bare-metal-fulfillment-operator"]:::osacStyle
        PP6["metering · CSI driver"]:::osacStyle
    end
```

### Helm Charts
The helm charts osac-infra & osac-deps helm chart are helper charts.
The `osac` helm chart must be pinned to specific release tag( for eg, 0.0.21 )
> [!IMPORTANT]
> Existing version of oc-mirror does not support mirroring of the
> osac helm chart for 2 reasons . 1. No support for `oci:` protocol[^1]
> 2. No support for passing values to the helm chart [^2]

#### Installing OSAC on a Disconnected Cluster via a Bastion Laptop

In disconnected environments where cluster nodes cannot reach external
registries, OSAC can be installed using an internet-connected workstation
(laptop) as a bridge. The workstation pulls Helm charts from the upstream
registry and submits rendered manifests to the cluster API server through
an SSH tunnel — no Helm chart mirroring is required.

```mermaid
flowchart LR
    %% Subgraphs for visual grouping
    subgraph Workstation[" Local Workstation "]
        direction TB
        WS[" Workstation<br/><i>(Internet Access)</i> "]
        WS_CLI[" Tools & Config:<br/>• helm CLI<br/>• kubeconfig "]
        WS_HELM[" Pulls Helm charts from<br/><b>ghcr.io</b> "]
    end

    subgraph DMZ[" Bastion Host / DMZ "]
        BM[" Bastion / BM Host<br/><code>10.6.76.15</code> "]
    end

    subgraph Cluster[" Air-Gapped OpenShift Cluster "]
        direction TB
        OCP[" Disconnected OpenShift<br/><code>192.168.100.0/24</code> "]
        REG[" Mirror Registry<br/><i>(Image Source)</i> "]
    end

    %% Network Connections & Tunnels
    WS ==>|" SSH Tunnel / sshuttle VPN "| BM
    BM ==>|" Internal Network "| OCP
    OCP -.->|" Pulls images "| REG

    %% Styling
    classDef default fill:#f9f9f9,stroke:#333,stroke-width:1px;
    classDef workstation fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef bastion fill:#fff3e0,stroke:#f57c00,stroke-width:2px;
    classDef cluster fill:#e8f5e9,stroke:#388e3c,stroke-width:2px;

    class WS,WS_CLI,WS_HELM workstation;
    class BM bastion;
    class OCP,REG cluster;
```

#### Disconnected Installation specific values

The customer guide and the helm deployment guide in this repository provides
helm commands  to discover the helm values. This document will provide information on
distinct Helm values making disconnected installation smoother.
In both the connected and disconnected environments, the following shell variables
are essential. At the time of writing, `DOMAIN`, `OCP_VERSION` and `KC_HOSTNAME`
must exist.

#### Overriding

The `osac-deps` chart exposes two values that control which CatalogSource
every operator Subscription references:

| Value | Default | Description |
|-------|---------|-------------|
| `catalogSource` | `redhat-operators` | Name of the OLM `CatalogSource` CR |
| `catalogSourceNamespace` | `openshift-marketplace` | Namespace where the CatalogSource exists |
| `cliImage` | `quay.io/openshift/origin-cli:4.20.0` | Name of the Openshift CLI used by OSAC installation specific hooks |

##### Helm install / upgrade command line based parameters

Pass the overrides directly:

```bash
helm upgrade --install osac-deps osac-installer/charts/osac-deps \
--set catalogSource=my-mirror-catalog \
--set catalogSourceNamespace=openshift-marketplace
```

##### Values file

Alternatively, create a custom values file:

```yaml
# custom-values.yaml
catalogSource: my-mirror-catalog
catalogSourceNamespace: openshift-marketplace
```

Then reference it during install:

```bash
helm upgrade --install osac-deps osac-installer/charts/osac-deps \
-f custom-values.yaml
```

For first time experience of installation, I recommend manual installation over
helm based installation. It becomes easier to determine the root cause of errors.
Order of installation is determined based upon the Helm weight and type.
osac-deps


##### CaaS Cluster Considerations
Check with the CaaS team.

## Troubleshooting

### Subscription stuck in "UpgradePending" or no InstallPlan created

Verify the CatalogSource is healthy:

```bash
oc get catalogsource -n openshift-marketplace
```

The `READY` column should show `true` and `STATUS` should be `READY`.

### Operator package not found

Confirm the package exists in your mirrored catalog:

```bash
oc get packagemanifests -l "catalog=<your-catalog-name>"
```

If the package is missing, update your `ImageSetConfiguration` to include
it and re-run `oc-mirror`.

### Image pull errors

Ensure the `ImageContentSourcePolicy` or `ImageDigestMirrorSet` is applied
and that the cluster nodes can reach the mirror registry. Check node-level
pull status:

```bash
oc get machineconfigpool
oc debug node/<node-name> -- chroot /host podman pull <image>
```

#### Footnotes
[^1]: [RFE-8713 — “oc-mirror fails to render Helm charts containing required values during image discovery”](https://redhat.atlassian.net/browse/RFE-8713)
  [^2]: [RFE-8748: Support for Helm mirroring avilable in OCI Compliant registries](https://redhat.atlassian.net/browse/RFE-8748) 
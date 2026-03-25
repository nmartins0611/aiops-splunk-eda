# Predictive AIOps with Splunk ITSI and Event-Driven Ansible

Closed-loop AIOps that detects network throughput anomalies using Splunk ITSI
predictive analytics (MLTK Kalman Filter) and automatically remediates them
through Event-Driven Ansible and Ansible Automation Platform.

## Architecture

```
                         ┌─────────────────────────────────┐
                         │         Splunk ITSI              │
                         │                                  │
                         │  MLTK Forecast (Kalman / LLP5)   │
                         │         ▼                        │
                         │  KPI Breach → Notable Episode    │
                         │         ▼                        │
                         │  Rules Engine → Webhook Action   │
                         └────────────────┬────────────────┘
                                          │ POST (itsi_notable:group)
                                          ▼
                         ┌─────────────────────────────────┐
                         │     EDA Controller               │
                         │                                  │
                         │  Rulebook: network_load.yml      │
                         │  Source:   ansible.eda.webhook    │
                         │            0.0.0.0:5000           │
                         │                                  │
                         │  Match: sourcetype + policy_id   │
                         │  Throttle: 15 min per episode    │
                         └────────────────┬────────────────┘
                                          │ run_job_template
                                          ▼
                         ┌─────────────────────────────────┐
                         │     AAP Controller               │
                         │                                  │
                         │  Job Template: Network loadbalance│
                         │  Playbook: remediate_network_    │
                         │            throughput.yml         │
                         └────────────────┬────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
           ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
           │ Remediate    │    │ Comment & Close   │    │ Re-enable    │
           │ (scale infra)│    │ ITSI Episode      │    │ Demo Injector│
           └──────────────┘    └──────────────────┘    └──────────────┘
```

## How It Works

1. **Detect** — Splunk ITSI monitors the *Network Throughput — Bytes In* KPI on
   the *On-Prem DB* service. The MLTK `predict` command (Kalman Filter / LLP5)
   forecasts expected values. When actuals breach the upper confidence band, the
   KPI health score drops to Critical and ITSI groups the events into a Notable
   Episode.

2. **Trigger** — The ITSI Rules Engine fires a webhook action that POSTs the
   episode payload (`itsi_notable:group`) to EDA Controller on port 5000.

3. **Match** — The `network_load.yml` rulebook matches events with the correct
   `sourcetype` and `itsi_policy_id`, throttling duplicate firings to once per
   episode every 15 minutes.

4. **Remediate** — EDA launches the `Network loadbalance` job template in AAP
   Controller, which runs `remediate_network_throughput.yml` against the `itsi`
   host group. The playbook:
   - Disables the demo spike injector (simulates infrastructure scaling)
   - Waits for KPIs to normalize
   - Comments on the ITSI episode with remediation details
   - Closes the episode (status resolved, severity info)
   - Re-enables the injector so the demo loop continues

## Repository Structure

```
aiops-splunk-eda/
├── README.md
├── collections/
│   └── requirements.yml          # Ansible collection dependencies
├── extensions/
│   └── eda/
│       └── rulebooks/
│           └── network_load.yml  # EDA rulebook — webhook listener + rules
└── playbooks/
    └── remediate_network_throughput.yml  # Remediation playbook
```

## Components

| Component | Role | Detail |
|-----------|------|--------|
| **Splunk ITSI** | Predictive analytics & episode management | Kalman Filter forecast, KPI health scoring, Notable Episodes |
| **MLTK** | Machine learning in SPL | `predict` command with LLP5 algorithm for time-series forecasting |
| **EDA Controller** | Event-driven automation | Listens for webhooks, matches conditions, triggers AAP jobs |
| **AAP Controller** | Orchestration | Runs remediation playbook as a job template |
| **`splunk.itsi`** | Ansible Content Collection | Episode comments and status updates via `itsi_add_episode_comments`, `itsi_update_episode_details` |
| **`ansible.eda`** | Ansible Content Collection | `webhook` source plugin for receiving events |

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Ansible Automation Platform | 2.5+ | Controller + EDA Controller |
| Splunk Enterprise | 9.x | With ITSI 4.19+ and MLTK 5.x |
| `splunk.itsi` collection | latest | For episode management modules |
| `ansible.eda` collection | latest | For the webhook event source |

### Splunk ITSI Configuration

Before this automation will fire, the following must be configured in ITSI:

1. **Service**: e.g. *On-Prem DB* with a throughput KPI
2. **KPI**: *Network Throughput — Bytes In* with adaptive thresholds or MLTK
   forecast-based alerting
3. **Correlation Search**: detects when the KPI breaches its predicted range
4. **Notable Episode Group Policy**: groups related events into episodes
5. **Rules Engine Action**: sends a webhook POST to `http://<eda-host>:5000`
   when an episode is created or severity changes

### AAP Configuration

| Object | Value |
|--------|-------|
| **Project** | Git repo pointing to this repository |
| **Inventory** | Host group `itsi` with Splunk connection details |
| **Credential** | Splunk HTTP API credentials (`ansible_user`, `ansible_httpapi_pass`) |
| **Job Template** | Name: `Network loadbalance`, Playbook: `playbooks/remediate_network_throughput.yml` |
| **EDA Rulebook Activation** | Source: `extensions/eda/rulebooks/network_load.yml`, linked to the AAP credential + project |

### Inventory Variables

The playbook expects these variables on the `itsi` host:

```yaml
ansible_host: splunk.example.com
ansible_httpapi_port: 8089
ansible_user: admin
ansible_httpapi_pass: "{{ vault_splunk_password }}"
ansible_connection: httpapi
ansible_network_os: splunk.es.splunk
```

---

## Getting Started

### 1. Clone the repo

```bash
git clone https://github.com/nmartins0611/aiops-splunk-eda.git
cd aiops-splunk-eda
```

### 2. Install Ansible collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 3. Configure AAP Controller

Create the following objects in AAP:

- **Project** — SCM Type: Git, URL: `https://github.com/nmartins0611/aiops-splunk-eda.git`
- **Inventory** — add a host group `itsi` with the Splunk connection variables above
- **Credential** — type: Machine or custom, storing Splunk admin credentials
- **Job Template** — name: `Network loadbalance`, inventory: `itsi`, project: this repo, playbook: `playbooks/remediate_network_throughput.yml`

### 4. Activate the EDA Rulebook

In EDA Controller:

- Create a **Rulebook Activation** using `extensions/eda/rulebooks/network_load.yml`
- Link the AAP Controller credential so `run_job_template` can launch jobs
- Verify the webhook listener is running on port `5000`

### 5. Configure Splunk ITSI

Set up the Rules Engine action to POST to `http://<eda-controller>:5000/endpoint`:

```
URL:          http://<eda-controller-host>:5000/endpoint
Method:       POST
Content-Type: application/json
```

### 6. Test the integration

Send a test payload to the EDA webhook to verify the pipeline end-to-end:

```bash
curl -X POST http://<eda-controller-host>:5000/endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "sourcetype": "itsi_notable:group",
    "itsi_policy_id": "ee2d56fe-1ef8-11f1-9a43-06dfc3703af7",
    "itsi_group_id": "test-episode-001",
    "title": "Network Throughput - Bytes In anomaly on On-Prem DB",
    "severity": "critical"
  }'
```

Check the EDA Controller UI for the matched rule and the AAP Controller for a
launched `Network loadbalance` job.

---

## Customising the Rulebook

### Matching Your ITSI Policy ID

The rulebook filters on a specific `itsi_policy_id`. Update this value to match
your ITSI Notable Event Aggregation Policy:

```yaml
condition: >-
  event.payload.sourcetype is defined
  and event.payload.sourcetype == "itsi_notable:group"
  and event.payload.itsi_policy_id is defined
  and event.payload.itsi_policy_id == "<your-policy-id>"
```

Find your policy ID in ITSI under **Configuration → Notable Event Aggregation
Policies** or by running:

```spl
| rest /servicesNS/nobody/SA-ITOA/notable_event_aggregation_policy
| table title policy_id
```

### Adjusting the Throttle

The default throttle prevents the same episode from re-triggering remediation
within 15 minutes. Adjust `once_within` to match your KPI normalization window:

```yaml
throttle:
  group_by_attributes:
    - event.payload.itsi_group_id
  once_within: 15 minutes
```

---

## Customising the Playbook

The demo playbook uses a Splunk saved search (the "spike injector") to simulate
the remediation action. In production, replace the disable/enable tasks with
your actual infrastructure remediation — for example:

| Environment | Module | Example |
|-------------|--------|---------|
| Cisco routers | `cisco.ios.ios_config` | Increase interface bandwidth |
| F5 load balancer | `f5networks.f5_modules.bigip_pool_member` | Add pool members |
| AWS | `amazon.aws.ec2_instance` | Upgrade instance type |
| VMware | `vmware.vmware_rest.vcenter_vm` | Add vCPU / memory |
| Kubernetes | `kubernetes.core.k8s` | Scale deployment replicas |

The four-phase pattern stays the same regardless of the remediation action:

1. **Fix** — apply the infrastructure change
2. **Wait** — pause for KPIs to normalize
3. **Document** — comment on the ITSI episode
4. **Close** — resolve the episode

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| EDA not receiving events | Verify webhook listener is running: `curl http://<eda-host>:5000/endpoint` should return 200 |
| Rule not matching | Check `sourcetype` and `itsi_policy_id` in the payload match the rulebook condition |
| Job template not launching | Verify EDA has a valid AAP credential and the template name matches exactly (`Network loadbalance`) |
| Episode not closing | Confirm `episode_id` is populated — check that `ansible_eda.event.payload.itsi_group_id` exists in the event |
| Splunk REST calls failing | Verify `ansible_host`, `ansible_httpapi_port`, and credentials on the `itsi` inventory host |
| Throttle blocking re-runs | Wait 15 minutes or change `once_within` in the rulebook |

---

## Solution Guide

For a comprehensive walkthrough covering this use case plus two additional
Splunk AIOps patterns (RHEL server remediation, network OSPF remediation), see
the [AIOps with Splunk and Event-Driven Ansible](https://github.com/nmartins0611/solution-guides/blob/main/README-AIOps-Splunk-ITSI.md) solution guide.

---

## License

GPL-3.0

# Security Response Automation

Cloud Functions to take automated actions on threat and vulnerability findings.

## Note

This project is currently under development and is not yet ready for users. Stay tuned! If you do decide to use this library early and have any questions please reach out to tomfitzgerald@google.com for help.

## Getting Started

This repository contains Cloud Functions to take automated actions on findings from Event Threat Detection and Security Health Analytics (SHA). For example, if SHA alerts you that a Google Cloud Storage bucket is open you may want to close it, or perhaps leave it alone if its meant to be public. The logic and the framework to express such automation is the purpose of SRA!

### Configuration

Before installation we'll configure our Cloud Functions in `settings.json`. Within this file we'll restrict our Functions to only take actions if the affected resource is within a set of resource IDs. Each Function that considers resources will support the following resources:

- Project IDs `folder_ids`: The Function will execute if the affected project ID is within this set.
- Folder IDs `project_ids`: Take the action if the affected project ID has an ancestor of a folder ID within this set.
- Organization ID `organization_id`: Take the action if the affected project ID is within this organization ID.

**Google Cloud Storage**

***Close open buckets***

This Cloud Function will automatically close public buckets found by Security Health Analytics that match the criteria you specify. Depending on which resources you specify will determine which projects are enforced.

For example, if you wanted to only close buckets in the folder **development** you'll want to find that folders ID in [Cloud Resource Manager](https://console.cloud.google.com/cloud-resource-manager) and place into the `folder_ids` array.

- Configured in settings.json under the `close_bucket` key.

**Revoke IAM grants**

This Cloud Function responds to Event Threat Detection's Anomalous IAM grant detector sub rule, external account added sub-rule. Within this configuration you can specify a list of domain names that you wish to disable. This way you can control which members are removed.

- Configured in settings.json under the `revoke_iam` key.
- `remove_list` An array of strings containing domain names to be matched against the members added. If there is a match, the member will be removed from the resource if that are within the below resources.

**Remove Public IPs from GCE Instance**

This Cloud Function will automatically remove public IPs found by Security Health Analytics that match the criteria you specify.
Depending on which resources you specify will determine which projects are enforced.

- Configured in settings.json under the `remove_public_ip` key.

**Disable Kubernetes Dashboard addon**

This Cloud Function will automatically disable Kubernetes Dashboard addon found by Security Health Analytics.
Depending on which resources you specify will determine which projects are enforced.

- Configured in settings.json under the `disable_dashboard` key.

### Installation

Following these instructions will deploy all SRA Cloud Functions. Before you get started be sure
you have (at least) **Go version 1.11 installed**.

```shell
$ gcloud auth application-default login
$ terraform init

// Install all Functions.
$ terraform apply

// Install a single Function.
$ terraform apply --target module.revoke_iam_grants
```

TIP: Instead of entering variables every time you can create `terraform.tfvars`
file and input key value pairs there, i.e.
`automation-project="aerial-jigsaw-235219"`.

If at any point you want to revert the changes we've made just run `terraform destroy .`

**CSCC Notifications**

Security Health Analytics requires CSCC notifications to be setup. This requires your account to be added to a early access group, please ping tomfitzgerald@google.com to be added. You can then create a new notification config that will send all CSCC findings to a Pub/Sub topic.

```shell
$ export PROJECT_ID=ae-threat-detection \
  SERVICE_ACCOUNT_EMAIL=automation-service-account@$PROJECT_ID.iam.gserviceaccount.com \
  ORGANIZATION_ID=154584661726 \
  TOPIC_ID=cscc-notifications-topic

$ ./enable-cscc-notfications.sh
```

### Reinstalling a Cloud Function

Terraform will create or destroy everything by default. To redeploy a single Cloud Function you can do:

```shell
$ zip -r ./deploy/functions.zip . -x *deploy* -x *.git* -x *.terraform*
$ terraform apply .
```

Then visit Cloud Console, Cloud Functions, click the Function name then edit. Finally hit deploy.

### Test

```shell
$ go test ./...
```

## CSCC notifications setup

- Need to grant automation account the proper permissions. Below example shown if adding via the
  config service account. Note the granting account must have organization admin to grant this
  role.
- Make sure to edit `enable-cscc-notifications.sh` and fill in your variables to match your
  environment.

```shell
./enable-cscc-notifications.sh
```

### Logging

Each Cloud Function logs its actions to the below log location. This can be accessed by visiting
StackDriver and clicking on the arrow on the right hand side then 'Convert to advanced filter'.
Then paste in the below filter making sure to change the project ID to the project where your
Cloud Functions are installed.

`logName="projects/{{ project_id }}/logs/security-response-automation"`

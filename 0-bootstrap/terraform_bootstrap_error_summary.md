# Terraform Bootstrap Error Summary

During the deployment of the `0-bootstrap` stage for the GCP landing zone, several cascading issues were identified and resolved across different resources. This document outlines the problems and their solutions, organized by component based on the finalized state.

---

## 1. Cloud Storage State Bucket Issues

### 1.1 `Error 400: Service account ... does not exist`
- **Impacted Modules:** `seed_bootstrap`, `cloudbuild_bootstrap`
- **Cause:** Cloud Storage uses a dedicated, Google-managed service account to handle encryption/decryption keys. This account is not created the instant a project is generated. The Terraform code attempted to assign KMS roles to it by manually constructing its email address before GCP had provisioned it.
- **Solution:** Configured Terraform to force early service account creation using the `data.google_storage_project_service_account.gcs_account` data source. This guarantees the API retrieves (and automatically creates if missing) the account before attaching IAM roles.

### 1.2 `Error 403: Permission denied on Cloud KMS key`
- **Impacted Modules:** `seed_bootstrap`, `cloudbuild_bootstrap`
- **Cause:** Because the IAM role assignment failed in the previous step, the Cloud Storage service account never received the necessary `roles/cloudkms.cryptoKeyEncrypterDecrypter` permission. Terraform then attempted to create the bucket simultaneously without waiting.
- **Solution:** Enforced execution order by adding `depends_on = [google_project_iam_binding.gs_encrypt_decrypt]` to the `google_storage_bucket.org_terraform_state` resource to ensure permissions exist before bucket creation.

---

## 2. Cloud Build Service Account Issues

### 2.1 Default Cloud Build Account `NOT_FOUND` / Transition to Custom Service Account
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** The `cloudbuild_project` utilizes the `terraform-google-project-factory` module (and under the hood, `core-project-factory`). This module defaults to disabling the default compute service account if not explicitly configured otherwise. Because the default service account was disabled, it caused subsequent steps relying on that default Cloud Build identity to fail with 'account not found' and cascading provisioner failures.
- **Solution:** Rather than re-enabling the default service account (which has overly broad permissions and violates the principle of least privilege), we transitioned entirely to a dedicated service account: `google_service_account.cloudbuild_sa.email`. We replaced all references to `${module.cloudbuild_project.project_number}@cloudbuild.gserviceaccount.com` with this custom identity, ensuring we maintain full lifecycle control and strict least-privilege scoping over the builds.

### 2.2 Custom Account Propagation (`400 Bad Request`)
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** Terraform successfully created our custom `cloudbuild_sa` account, but GCP's IAM API is eventually consistent. Attempting to attach multiple role bindings immediately resulted in a "does not exist" error due to global replication delays.
- **Solution:** Added a `30s` `time_sleep.wait_for_sa_propagation` buffer after the service account creation. All dependent IAM member resources explicitly use `depends_on = [time_sleep.wait_for_sa_propagation]` to wait for the identity to propagate before binding roles.

### 2.3 Missing Log Writer Permissions
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** A warning indicating the custom service account could not write build logs to Cloud Logging.
- **Solution:** Explicitly granted the `roles/logging.logWriter` role to the custom service account (`google_project_iam_member.cloudbuild_runner_log_writer`) to ensure output streaming.

---

## 3. `gcloud builds submit` and Docker Failures

### 3.1 `INVALID_ARGUMENT` for Logging Bucket
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** Modern Cloud Build security policies require explicitly defining logging behavior when using a custom service account.
- **Solution:** Placed `options: logging: CLOUD_LOGGING_ONLY` into the `cloudbuild.yaml` configuration.

### 3.2 `403 Forbidden` for GCS Source Staging and Dependency ordering
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** By default, `gcloud builds submit` attempts to create and use a default staging bucket. Our restricted custom service account lacked permission to do so. Moreover, the provisioner frequently ran before the artifacts bucket and service account were fully registered in Terraform.
- **Solution:** 
  - Defined the explicit staging directory using `--gcs-source-staging-dir=gs://${google_storage_bucket.cloudbuild_artifacts.name}/source`.
  - Enforced strict execution ordering for `null_resource.cloudbuild_terraform_builder` using `depends_on = [module.cloudbuild_project, google_service_account.cloudbuild_sa, google_storage_bucket_iam_member.cloudbuild_artifacts_iam]`.

### 3.3 Docker Build Caching and Expired Dependencies
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** The internal Docker build failed silently in Cloud Build. The `Dockerfile` used an expired `ubuntu_18_0_4` image which caused `apt-get` to fail on outdated GPG keys. Additionally, a monolithic `RUN` command broke Docker layer caching, causing massive delays on rebuilds.
- **Solution:** 
  - Upgraded base image to `ubuntu:24.04`.
  - Fixed expired Google Cloud SDK repository keys using modern keyring conventions.
  - Removed the deprecated `terraform-validator` (now bundled into `google-cloud-cli-terraform-tools`).
  - Separated the `apt-get` OS dependencies, the `google-cloud-cli` installation, and the `curl -O` Terraform version download into distinct discrete `RUN` layers to optimize Docker build cache. Rebuilds now leverage `CACHED` layers effectively.

### 3.4 Terraform "Deposed Object" Bug
- **Impacted Modules:** `cloudbuild_bootstrap`
- **Cause:** Attempting to re-run Terraform after the `local-exec` provisioner failed resulted in a core Terraform crash: "Attempt to restore non-existent deposed object". This is a known bug where a `null_resource` state becomes corrupted if it fails mid-creation.
- **Solution:** Manually cleared the tainted resource from the local file state using `terraform state rm module.cloudbuild_bootstrap.null_resource.cloudbuild_terraform_builder`.

### 3.5 Cloud Source Repositories API Disabled (`403 SERVICE_DISABLED`)
- **Impacted Modules:** `cloudbuild_bootstrap`, `0-bootstrap`
- **Cause:** The `0-bootstrap/main.tf` configuration attempts to provision a standalone Cloud Source Repository (`google_sourcerepo_repository.gcp_policies`). However, Google Cloud Source Repositories is officially **deprecated** and no longer allows new repositories to be created. As a result, the upstream `cloudbuild` module rightfully removed `sourcerepo.googleapis.com` from its internal list of enabled APIs.
- **Solution:** Removed the API entirely and commented out the local `google_sourcerepo_repository.gcp_policies` resource block in `0-bootstrap/main.tf` since the service can no longer be provisioned.

---

## 4. GitHub Actions Runner (`runner-mig`) Issues

### 4.1 Autoscaler Name Regex Failure
- **Impacted Modules:** `runner-mig`
- **Cause:** The `google_compute_region_autoscaler` relies on the `project_name` string to construct its resource name (e.g. `${var.project_name}-runner-vm-autoscaler`). Because `project_name` was left completely blank (`""`) in `terraform.tfvars`, the resulting name started with a hyphen (`-runner-vm-autoscaler`), which violates GCP naming regex rules.
- **Solution:** Populated the `project_name` variable in `terraform.tfvars` with a valid string (e.g., `"cdcvnlz"`).

### 4.2 Error resolving image name `ubuntu-os-cloud/ubuntu-2004-lts`
- **Impacted Modules:** `runner-mig`, `0-bootstrap`
- **Cause:** In `gh-runner.tf`, the upstream inputs were mapped incorrectly to the child module. The discrete image string `source_image` was accidentally assigned the value of the OS family, and `source_image_family` was assigned the Google project. The resulting image lookup failed because GCP interpreted the family string as an explicit disk name.
- **Solution:** 
  - Corrected the mappings in `gh-runner.tf` to pass the correct standard variables (`source_image_family = var.source_image_family` and `source_image_project = var.source_image_project`) directly to the GitHub Runner Managed Instance Group.
  - Took advantage of this fix to simultaneously upgrade the GitHub runner OS base image. Bumped the default OS base image family in `0-bootstrap/variables.tf` from the aging `"ubuntu-2004-lts"` to the modern `"ubuntu-2404-lts-amd64"`.

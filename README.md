# Workload Identity Federation with Gitlab

## Example GitLab token

https://docs.gitlab.com/ee/ci/cloud_services/index.html#how-it-works

```json
{
  "jti": "c82eeb0c-5c6f-4a33-abf5-4c474b92b558",
  "iss": "https://gitlab.example.com",
  "aud": "https://gitlab.example.com",
  "iat": 1585710286,
  "nbf": 1585798372,
  "exp": 1585713886,
  "sub": "project_path:mygroup/myproject:ref_type:branch:ref:main",
  "namespace_id": "1",
  "namespace_path": "mygroup",
  "project_id": "22",
  "project_path": "mygroup/myproject",
  "user_id": "42",
  "user_login": "myuser",
  "user_email": "myuser@example.com",
  "pipeline_id": "1212",
  "pipeline_source": "web",
  "job_id": "1212",
  "ref": "auto-deploy-2020-04-01",
  "ref_type": "branch",
  "ref_protected": "true",
  "environment": "production",
  "environment_protected": "true"
}
```

## Terraform

```hcl
locals {
  project_id = "MY_PROJECT_ID"
}

resource "google_service_account" "gitlab" {
  project    = local.project_id
  account_id = "gitlab"
}

resource "google_project_iam_member" "gitlab_role" {
  project = local.project_id
  role    = "roles/viewer"
  member  = google_service_account.gitlab.member
}

module "gl_oidc" {
  source            = "terraform-google-modules/github-actions-runners/google//modules/gh-oidc"
  project_id        = local.project_id
  pool_id           = "gitlab-pool"
  provider_id       = "gitlab-provider"
  issuer_uri        = "https://gitlab.127.0.0.1.nip.io/"
  allowed_audiences = ["https://gitlab.127.0.0.1.nip.io"]

  # https://docs.gitlab.com/ee/ci/cloud_services/index.html#how-it-works
  attribute_mapping = {
    "google.subject"         = "assertion.sub"
    "attribute.user_login"   = "assertion.user_login"
    "attribute.user_email"   = "assertion.user_email"
    "attribute.project_path" = "assertion.project_path"
  }

  sa_mapping = {
    "gitlab" = {
      sa_name = google_service_account.gitlab.id
      #attribute = "attribute.project_path/mygroup/myproject"
      #attribute = "attribute.user_login/myuser"
      attribute = "attribute.user_email/myuser@example.com"
    }
  }
}

output "pool_name" {
  value = module.gl_oidc.pool_name
}

output "provider_name" {
  value = module.gl_oidc.provider_name
}

```

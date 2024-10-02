# GitHub Actions Runner Controller on Google Kubernets Engine

## Setup GitHub Application for ARC

1. Go to [Developer Settings > GitHub Apps](https://github.com/settings/apps)
![image](./images/SkzpPT9AA.png)

### Create a GitHub APP
2. Create a GitHub App that is owned by an organization. For more information, see "[Registering a GitHub App](https://docs.github.com/en/apps/creating-github-apps/creating-github-apps/creating-a-github-app)". Configure the GitHub App as follows.
    
    1.  For "Homepage URL," enter `https://github.com/actions/actions-runner-controller`.
        
    2.  Under "Permissions," click **Repository permissions**. Then use the dropdown menus to select the following access permissions.
        
        -   **Administration**: Read and write
            
            **Note**: `Administration: Read and write` is only required when configuring Actions Runner Controller to register at the repository scope. It is not required to register at the organization scope.
            ![image](./images/Sy6V_pcAC.png)


            
        -   **Metadata**: Read-only
            ![image](./images/SJIIOp50C.png)

            
    3.  Under "Permissions," click **Organization permissions**. Then use the dropdown menus to select the following access permissions.
        
        -   **Self-hosted runners**: Read and write
            ![image](./images/ry83uTqRA.png)


3.  After creating the GitHub App, on the GitHub App's page, note the value for "App ID". You will use this value later.
![image](./images/ry_AOaq00.png)

    ```APP ID=1014082```

    
4.  Scroll down you'll see under "Private keys", click **Generate a private key**, and save the `.pem` file. You will use this key later.
![image](./images/SJZDFTc0A.png)
![image](./images/rkAJq6900.png)

And, browser will download a **"xxx.pem"**.

5.  In the menu at the top-left corner of the page, click **Install app**, and next to your organization, click **Install** to install the app on your organization.
![image](./images/SJtbqpqC0.png)
![image](./images/HyRz5690R.png)



6.  After confirming the installation permissions on your organization, note the app installation ID. You will use it later. You can find the app installation ID on the app installation page, which has the following URL format:
    
    `https://github.com/organizations/ORGANIZATION/settings/installations/INSTALLATION_ID`
    
In this example, [https://github.com/settings/installations/55521982](https://github.com/settings/installations/55521982), Installation ID is `55521982`


## Setup GCP Project
### Create a Project

```bash
$ gcloud projects create --name="Devfest 2024 Demo" 
```

```bash=
No project ID provided.

Use [devfest-2024-demo-437413] as project ID (Y/n)?  Y

Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/devfest-2024-demo-437413].
Waiting for [operations/cp.6504422382618409515] to finish...done.
Enabling service [cloudapis.googleapis.com] on project [devfest-2024-demo-437413]...
Operation "operations/acat.p2-643933963932-92a1c279-58bd-4ad6-bf32-8c38da03be6b" finished successfully.
```
project ID is `devfest-2024-demo-437413`

```bash
$ export PROJECT_ID=devfest-2024-demo-437413
```

### Settings for GCP Project

#### Set Quota Project

```bash
$ gcloud auth application-default set-quota-project $PROJECT_ID
```

#### Set Default Project
```bash
$ gcloud config set project $PROJECT_ID
```

### Enable Require GCP APIs

```bash
$ gcloud services enable \
    cloudresourcemanager.googleapis.com \
    storage-api.googleapis.com \
    iam.googleapis.com \
    container.googleapis.com \
    serviceusage.googleapis.com
```

If you got the error like this, please go to https://console.cloud.google.com/billing/linkedaccount , link a billing account for this working project.


ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project '643933963932' is not found. Billing must be enabled for activation of service(s) 'container.googleapis.com,container.googleapis.com,artifactregistry.googleapis.com,compute.googleapis.com,compute.googleapis.com,compute.googleapis.com,containerregistry.googleapis.com,dns.googleapis.com,dns.googleapis.com' to proceed.


### Setup GCP Crendentilas

```bash
$ gcloud auth application-default login
```

## Prepate a DEMO Repository 

### Fork DEMO Repository

Go to https://github.com/jianan1104/devfest-2024-gh-arc-gke-demo.git ,and fork it.


## Deploy GitHub Actions Runner Controller Using by Terraform

### Clone the Repository


``` bash
$ git clone https://github.com/terraform-google-modules/terraform-google-github-actions-runners.git
```


```
cd ./terraform-google-github-actions-runners/examples/gh-runner-gke-dind
```

### Edit main terraform file


Set value for `gh_app_id`, `gh_app_installation_id`, `gh_app_private_key` and `gh_config_url`.


- Use any text editor to open `*.pem` file to get raw text of private key.
- `gh_config_url` could be a repository URL or organization URL, depends on what account you installed APP.


Remove `enable_private_nodes`  for not support value for current GCP provider.
Add `zones` or `region` if you want your cluster at specific location.



```tf=
/**
 * Copyright 2020 Google LLC
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

module "runner-gke" {
  source  = "terraform-google-modules/github-actions-runners/google//modules/gh-runner-gke"
  version = "~> 4.0"

  project_id             = var.project_id
  create_network         = true
  network_name           = "runner-network-dind"
  subnet_name            = "runner-subnet-dind"
  cluster_suffix         = "dind"
  gh_app_id              = "1014082"
  gh_app_installation_id = "55521982"
  gh_app_private_key     = <<EOT
-----BEGIN RSA PRIVATE KEY-----

YOUR_PRIVATE_KEY_IN_XXX.pem

-----END RSA PRIVATE KEY-----
EOT
  gh_config_url          = "https://github.com/jianan1104/devfest-2024-gh-arc-gke-demo"
  arc_container_mode     = "dind"
  enable_private_nodes   = true // <-- Remove this line for current gcp provider error
  zones                  = ["asia-east1-a"] // Optional
  region                 = "asia-east1"     // Optional
  min_node_count         = 10               // Optional
  max_node_count         = 20               // Optional
}

```

### Execute Terraform Command to Build Up

```bash
$ terraform init
$ terraform apply -var="project_id=$PROJECT_ID"
```

Successful Output:

```bash
cluster_name = "gh-runner-dind"
location = "asia-east1-a"
project_id = "devfest-2024-demo-437413"
service_account = "tf-gke-gh-runner-dind-y9nl@devfest-2024-demo-437413.iam.gserviceaccount.com"
```


## Test with DEMO Repository 

### Go to Your DEMO Repository

Go to https://github.com/jianan1104/devfest-2024-gh-arc-gke-demo.git


### Click `Actions` Tab and Run Workflow

![image](./images/Sk1g8yo0C.png)

![image](./images/S1IZI1jC0.png)




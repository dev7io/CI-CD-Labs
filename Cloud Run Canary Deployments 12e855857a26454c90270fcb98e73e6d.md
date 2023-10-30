# Cloud Run Canary Deployments

Overview

Task 1. Preparing your environment
Task 2. Creating your Cloud Run service
Task 3. Enabling Dynamic Developer Deployments
Task 4. Automating canary testing
Task 5. Releasing to Production

# **Overview**

In this lab you will learn how to implement a deployment pipeline for Cloud Run that executes a progression of code from developer branches to production with automated canary testing and percentage based traffic management. It is intended for developers and DevOps engineers who are responsible for creating and managing CI/CD pipelines to Cloud Run.

Many organizations use robust release pipelines to move code into production. Cloud Run provides unique traffic management capabilities that let you implement advanced release management techniques with little effort.

# Objectives

- Create your Cloud Run service.
- Enable developer branch.
- Implement canary testing.
- Rollout safely to production.

# **Task 1. Preparing your environment**

1. In Cloud Shell, create environment variables to use in this lab:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=
gcloud config set compute/region $REGION
```

1. Enable the following APIs with the code below:
- Cloud Resource Manager
- GKE
- Cloud Source Repositories
- Cloud Build
- Container Registry
- Cloud Run

```bash
gcloud services enable \
cloudresourcemanager.googleapis.com \
container.googleapis.com \
sourcerepo.googleapis.com \
cloudbuild.googleapis.com \
containerregistry.googleapis.com \
run.googleapis.com
```

1. Grant the Cloud Run Admin role (roles/run.admin) to the Cloud Build service account:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role=roles/run.admin
```

1. Grant the IAM Service Account User role (roles/iam.serviceAccountUser) to the Cloud Build service account for the Cloud Run runtime service account:

```bash
gcloud iam service-accounts add-iam-policy-binding \
$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
--member=serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com \
--role=roles/iam.serviceAccountUser
```

1. If you haven't used Git in Cloud Shell previously, set the `user.name` and `user.email` values that you want to use (it's not necessary to have an existing account on GitHub):

```bash
git config --global user.email "[YOUR_EMAIL_ADDRESS]"
git config --global user.name "[YOUR_USERNAME]"
```

1. Clone and prepare the sample repository:

```bash
git clone https://github.com/GoogleCloudPlatform/software-delivery-workshop --branch cloudrun-progression-csr cloudrun-progression
cd cloudrun-progression/labs/cloudrun-progression
rm -rf ../../.git
```

1. Using **nano**, **vi** or any editor replace the `REGION` in `branch-cloudbuild.yaml` , `master-cloudbuild.yaml` and `tag-cloudbuild.yaml` files with the pre-populated `REGION` in **Step 1**.
2. Replace the placeholder values in the sample repository with your PROJECT_ID:

```bash
sed "s/PROJECT/${PROJECT_ID}/g" branch-trigger.json-tmpl > branch-trigger.json
sed "s/PROJECT/${PROJECT_ID}/g" master-trigger.json-tmpl > master-trigger.json
sed "s/PROJECT/${PROJECT_ID}/g" tag-trigger.json-tmpl > tag-trigger.json
```

1. Store the code from the sample repository in Google Source Repository:

```bash
gcloud source repos create cloudrun-progression
git init
git config credential.helper gcloud.sh
git remote add gcp https://source.developers.google.com/p/$PROJECT_ID/r/cloudrun-progression
git branch -m master
git add . && git commit -m "initial commit"
git push gcp master
```

# **Task 2. Creating your Cloud Run service**

In this section, you build and deploy the initial production application that you use throughout this lab.

1. In Cloud Shell, build and deploy the application, including a service that requires authentication. To make a public service use the `-allow-unauthenticated` flag as [explained in the Cloud Run documentation](https://cloud.google.com/run/docs/authenticating/public).

```bash
gcloud builds submit --tag gcr.io/$PROJECT_ID/hello-cloudrun
gcloud run deploy hello-cloudrun \
--image gcr.io/$PROJECT_ID/hello-cloudrun \
--platform managed \
--region $REGION \
--tag=prod -q
```

The output looks similar to the following:

```bash
Deploying container to Cloud Run service [hello-cloudrun] in project [sdw-mvp6] region REGION
✓ Deploying new service... Done.
✓ Creating Revision...
✓ Routing traffic...
Done.
Service [hello-cloudrun] revision [hello-cloudrun-00001-tar] has been deployed and is serving 100 percent of traffic.
Service URL: https://hello-cloudrun-apwaaxltma-uc.a.run.app
The revision can be reached directly at https://prod---hello-cloudrun-apwaaxltma-uc.a.run.app
```

The output includes the service URL and a unique URL for the revision. Your values will differ slightly from what's indicated here.

1. After the deployment is complete, view the newly deployed service by selecting **Cloud Run** from the Cloud Console menu, choosing the **hello-cloudrun service**, and selecting the **Revisions** page.
2. In Cloud Shell, view the authenticated service response:

```bash
PROD_URL=$(gcloud run services describe hello-cloudrun --platform managed --region $REGION --format=json | jq --raw-output ".status.url")
echo $PROD_URL
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $PROD_URL
```

# **Task 3. Enabling Dynamic Developer Deployments**

In this section, you enable developers with a unique URL for development branches in Git. Each branch is represented by a URL identified by the branch name. Commits to the branch trigger a deployment, and the updates are accessible at that same URL.

1. In Cloud Shell, set up the trigger:

```bash
gcloud beta builds triggers create cloud-source-repositories --trigger-config branch-trigger.json
```

1. To review the trigger, select **Cloud Build** from the Cloud Console menu and select **Triggers**.
2. In Cloud Shell, create a new branch:

```bash
git checkout -b new-feature-1
```

1. Open the sample application using your favorite editor or using the Cloud Shell IDE:

```bash
edit app.py
```

1. You can now browse or modify the code with the editor. In the sample application (**~/cloudrun-progression/labs/cloudrun-progression/app.py**), modify line 24 to indicate `v1.1` instead of `v1.0`:

```python
@app.route('/')
def hello_world():
return 'Hello World v1.1'
```

1. To return to your terminal, click **Open Terminal**.
2. In Cloud Shell, commit the change and push to the remote repository:

```bash
git add . && git commit -m "updated" && git push gcp new-feature-1
```

1. To review the build in progress, go back to the **Cloud Build** page and view the current build running on your new branch.
2. After the build completes, to review the revision, go to **Cloud Run** from the Cloud Console menu, choose the **hello-cloudrun service**, and select the **Revisions** page.
3. In Cloud Shell, get the unique URL for this branch:

```bash
BRANCH_URL=$(gcloud run services describe hello-cloudrun --platform managed --region $REGION --format=json | jq --raw-output ".status.traffic[] | select (.tag==\"new-feature-1\")|.url")
echo $BRANCH_URL
```

1. Access the authenticated URL:

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $BRANCH_URL
```

The updated response output looks like the following:

`Hello World v1.1`

# **Task 4. Automating canary testing**

When code is released to production, it's common to release to a small subset of live traffic before migrating all traffic to the new code base.

In this section, you implement a trigger that is activated when code is committed to the main branch. The trigger deploys the code to a unique canary URL and routes 10% of all live traffic to it.

1. In Cloud Shell, set up the branch trigger:

```bash
gcloud beta builds triggers create cloud-source-repositories --trigger-config master-trigger.json
```

1. To review the new trigger, go to the **Cloud Build > Triggers** page.
2. In Cloud Shell, merge the branch to the main line and push to the remote repository:

```bash
git checkout master
git merge new-feature-1
git push gcp master
```

1. To review the build in progress, go back to the **Cloud Build** page and view the current build.
2. After the build completes, to review the new revision, go to **Cloud Run**, choose the **hello-cloudrun service** , and select the **Revisions** page. Note that 90% of the traffic is routed to prod, 10% to canary, and 0% to the branch revisions.

[https://cdn.qwiklabs.com/ofBtRhde8X03IPSk4EP5LmHY9IUSvMEDgknkkq2SIG4%3D](https://cdn.qwiklabs.com/ofBtRhde8X03IPSk4EP5LmHY9IUSvMEDgknkkq2SIG4%3D)

1. Review the key lines of `master-cloudbuild.yaml` that implement the logic for the canary deploy.

Lines 39-44 deploy the new revision and use the tag flag to route traffic from the unique canary URL:

```bash
gcloud run deploy ${_SERVICE_NAME} \
--platform managed \
--region ${_REGION} \
--image gcr.io/${PROJECT_ID}/${_SERVICE_NAME} \
--tag=canary \
--no-traffic
```

Line 61 adds a static tag to the revision that notes the Git short SHA of the deployment:

```bash
gcloud beta run services update-traffic ${_SERVICE_NAME} --update-tags=sha-$SHORT_SHA=$${CANARY} --platform managed --region ${_REGION}
```

Line 62 updates the traffic to route 90% to production and 10% to canary:

```bash
gcloud run services update-traffic ${_SERVICE_NAME} --to-revisions=$${PROD}=90,$${CANARY}=10 --platform managed --region ${_REGION}
```

1. In Cloud Shell, get the unique URL for the canary revision:

```bash
CANARY_URL=$(gcloud run services describe hello-cloudrun --platform managed --region $REGION --format=json | jq --raw-output ".status.traffic[] | select (.tag==\"canary\")|.url")
echo $CANARY_URL
```

1. Review the canary endpoint directly:

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $CANARY_URL
```

1. To see percentage-based responses, make a series of requests:

```bash
LIVE_URL=$(gcloud run services describe hello-cloudrun --platform managed --region $REGION --format=json | jq --raw-output ".status.url")
for i in {0..20};do
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $LIVE_URL; echo \n
done
```

# **Task 5. Releasing to Production**

After the canary deployment is validated with a small subset of traffic, you release the deployment to the remainder of the live traffic.

In this section, you set up a trigger that is activated when you create a tag in the repository. The trigger migrates 100% of traffic to the already deployed revision based on the commit SHA of the tag. Using the commit SHA ensures the revision validated with canary traffic is the revision utilized for the remainder of production traffic.

1. In Cloud Shell, set up the tag trigger:

```bash
gcloud beta builds triggers create cloud-source-repositories --trigger-config tag-trigger.json
```

1. To review the new trigger, go to the [Cloud Build Triggers page](https://console.cloud.google.com/cloud-build/triggers) in the Cloud console.
2. In Cloud Shell, create a new tag and push to the remote repository:

```bash
git tag 1.1
git push gcp 1.1
```

1. To review the build in progress, go to the [Cloud Build Builds page](https://console.cloud.google.com/cloud-build/builds) in the Cloud console.
2. After the build is complete, to review the new revision, go to the **Cloud Run**, choose the **hello-cloudrun service**, and select the **Revisions** page in the Cloud console. Note that the revision is updated to indicate the prod tag and it is serving 100% of live traffic.

[https://cdn.qwiklabs.com/wxg0Z4wZdd4XN9QI3JD5ju7NimPSrL7h1GSmKLlI%2B2s%3D](https://cdn.qwiklabs.com/wxg0Z4wZdd4XN9QI3JD5ju7NimPSrL7h1GSmKLlI%2B2s%3D)

1. In Cloud Shell, to see percentage-based responses, make a series of requests:

```bash
LIVE_URL=$(gcloud run services describe hello-cloudrun --platform managed --region $REGION --format=json | jq --raw-output ".status.url")
for i in {0..20};do
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" $LIVE_URL; echo \n
done
```

1. Review the key lines of `tag-cloudbuild.yaml` that implement the production deployment logic.

Line 37 updates the canary revision adding the prod tag. The deployed revision is now tagged for both prod and canary:

```bash
gcloud beta run services update-traffic ${_SERVICE_NAME} --update-tags=prod=$${CANARY} --platform managed --region ${_REGION}
```

Line 39 updates the traffic for the base service URL to route 100% of traffic to the revision tagged as prod:

```bash
gcloud run services update-traffic ${_SERVICE_NAME} --to-revisions=$${NEW_PROD}=100 --platform managed --region ${_REGION}
```

# **Congratulations!**

Now you can use Cloud Build to create and rollback continuous integration pipelines with Cloud Run on Google Cloud!

[Cloud Run Canary Deployments | Google Cloud Skills Boost](https://www.cloudskillsboost.google/catalog_lab/5448)
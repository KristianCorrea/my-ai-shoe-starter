# TODO: Replace with your team name

## Our Team

TODO: Replace with your team members

## How to run the streamlit app

### Running the Streamlit app for development

Make sure you have installed the correct packages.

```shell
$ pip install -r requirements.txt
```

Change into this directory. You can check that you're in the correct directory by running `ls`. 
You should see the correct files printed out.

```shell
$ ls
app.py             data_fetcher.py       internals.py  modules_test.py  requirements.txt
custom_components  data_fetcher_test.py  modules.py    README.md        run-streamlit.sh
```

Run the following command to run the app locally. Note that if you make changes, you just
need to refresh the server and the new changes should appear (you do not need to rerun
the following command while you are actively making changes).

```shell
$ streamlit run app.py
```

### Using Docker to run the Streamlit app

Docker simply creates a virtual environment for only your app. This is what we will use to
deploy our app continuously.

Fortunately, we already have a script that builds and starts Docker for us. Run the 
following command to build the container and start the server locally.

```shell
$ ./run-streamlit.sh
```

## Setting up GitHub Actions for CI/CD

The GitHub Actions are already mostly configured for you. They are split
into two files:

* `.github/workflows/cloud-run.yml`: workflow for automatic deployment
* `.github/workflows/python-checks.yml`: workflow for continuous integration (testing)

You won't need to modify `python-checks` at all, but you will need to change
the environment variables in `cloud-run.yml` to work for your team's GCP project.

### Deploy manually on the command-line

First, deploy your project manually from the command line.

1. Set up your Google Cloud Project by following the "Before you begin" section of 
https://cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-python-service?hl=en. **Each student can do this with their own Cloud project to start.**

2. Build the image using the following commands, replacing the project ID and service 
name with your info. Your service name is the name of your webapp or project and can 
be whatever you want!

```shell
PROJECT_ID=your-cloud-project-name
SERVICE_NAME=your-team-service-name
gcloud builds submit --tag gcr.io/${PROJECT_ID}/${SERVICE_NAME}
```

3. Deploy the image that you just built by copying the following command. If successful, you should see a URL in the
output on the command-line. Follow the link and you should see your app!

```shell
gcloud run deploy ${SERVICE_NAME} \
    --image gcr.io/${PROJECT_ID}/${SERVICE_NAME}:latest \
    --region us-central1 \
    --allow-unauthenticated
```

### Set up automatic deployment in GitHub Actions

After a successful deployment, we can modify the `./github/workflows/cloud-run.yml`
file. Additional information on each step is in the [documentation](https://github.com/google-github-actions/auth?tab=readme-ov-file#workload-identity-federation-through-a-service-account)
 but you should be able to follow the steps below. **These steps should be done as a team, but only ONE person needs to run the commands.**

1. Decide on ONE team member's Cloud Project to use for the team project. That team member needs to go to **IAM & Admin** for their Cloud Project, click on **Grant Access** and add each team member's techexchange email as a "New Principle". For "Role", under **Basic** there should be **Editor**. Once every team member is an Editor for the project, every team member should be able to see the Cloud Project in the dropdown menu on the Google Cloud Console.

2. Create a Workload Identity Pool and a Provider for GitHub. Replace `your-common-cloud-project-name` with the team's Cloud Project name, `your-team-service-name` with a name for your team webapp, and `your-course-github-org` with the name of the GitHub organization for your class. You do NOT need to change `OIDC_NAME=project-repo`. Then run the following commands (once you run the first commands setting the PROJECT_ID and other bash variables, you can copy and paste the next commands).

```shell
PROJECT_ID=your-common-cloud-project-name
SERVICE_NAME=your-team-service-name
OIDC_NAME=project-repo
GITHUB_ORG=your-course-github-org
```

```shell
gcloud iam workload-identity-pools create "github" \
    --project="${PROJECT_ID}" \
    --location="global" \
    --display-name="GitHub Actions Pool"
```

```shell
gcloud iam workload-identity-pools providers create-oidc "${OIDC_NAME}" \
    --project="${PROJECT_ID}" \
    --location="global" \
    --workload-identity-pool="github" \
    --display-name="My GitHub repo Provider" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
    --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
    --issuer-uri="https://token.actions.githubusercontent.com"
```

3. Get the full path name for the Workload Identity Provider from gcloud.
You will use the output from this command as the `WORKLOAD_IDENTITY_PROVIDER` 
field in `.github/workflows/cloud-run.yml`. You've already set the `OIDC_NAME` and the `PROJECT_ID` as bash variables so you can copy and paste this command.

```shell
gcloud iam workload-identity-pools providers describe "${OIDC_NAME}" \
    --project="${PROJECT_ID}" \
    --location="global" \
    --workload-identity-pool="github" \
    --format="value(name)"
```

4. Authorize your service account. Use the output from the first command as the
`WORKLOAD_IDENTITY_POOL_ID` in the second command and your team's GitHub repo
name as the `GITHUB_REPO_NAME`. Also replace `GITHUB_ORG` with the name of the GitHub organization for your course.

```shell
gcloud iam workload-identity-pools describe "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)"
```

```shell
WORKLOAD_IDENTITY_POOL_ID=paste-output-from-prev-command
PROJECT_NUMBER=yourcloudprojectnumber
GITHUB_ORG=your-course-github-org
GITHUB_REPO_NAME=your-github-team-repo-name
```

```shell
gcloud iam service-accounts add-iam-policy-binding "${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO_NAME}"
```

5. In IAM in the Cloud Console, double check that the following roles are attached 
to the service account, which is the `PROJECT_NUMBER-compute@developer.gserviceaccount.com` from the previous command.

    - Cloud Build Service Account

6. Update the environmental variables (all of the TODOs) in 
`.github/workflows/cloud-run.yml`, commit and push your changes, 
and click on "Actions" in your team's repository on GitHub to see
the workflow running!

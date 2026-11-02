  You can use one GitHub Actions workflow to deploy both parts of your application:

  - Frontend: Build your React.js app and upload the production files to AWS S3.
  - Backend: Deploy your FastAPI application to an EC2 instance running Gunicorn behind Nginx.
  - Trigger: Automatically deploy when you push to `main`.

  The workflow below assumes your repository contains a `frontend/` React app and a `backend/` FastAPI app, and that Nginx and the Python environment are already configured on EC2.

  ## 1. GitHub Actions workflow

  Create `.github/workflows/deploy.yml`:

  yaml
  ```

  name: Deploy React and FastAPI

  on:
    push:
      branches:
        - main

    workflow_dispatch:

  permissions:
    contents: read

  concurrency:
    group: production-deployment
    cancel-in-progress: false

  jobs:
    build-frontend:
      name: Build and Deploy React to S3
      runs-on: ubuntu-latest

      defaults:
        run:
          working-directory: frontend

      steps:
        - name: Checkout Code
          uses: actions/checkout@v4

        - name: Setup Node.js
          uses: actions/setup-node@v4
          with:
            node-version: 22
            cache: npm
            cache-dependency-path: frontend/package-lock.json

        - name: Install Dependencies
          run: npm ci

        - name: Build React App
          run: npm run build
          env:
            VITE_API_URL: ${{ vars.VITE_API_URL }}

        - name: Configure AWS Credentials
          uses: aws-actions/configure-aws-credentials@v4
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ${{ secrets.AWS_REGION }}

        - name: Deploy Frontend to S3
          run: |
            aws s3 sync dist/ \
              s3://${{ secrets.AWS_S3_BUCKET }}/ \
              --delete \
              --cache-control "public, max-age=31536000, immutable" \
              --exclude "index.html"

            aws s3 cp dist/index.html \
              s3://${{ secrets.AWS_S3_BUCKET }}/index.html \
              --cache-control "no-cache, no-store, must-revalidate" \
              --content-type "text/html"

    deploy-backend:
      name: Deploy FastAPI to EC2
      runs-on: ubuntu-latest

      steps:
        - name: Checkout Code
          uses: actions/checkout@v4

        - name: Configure SSH
          env:
            EC2_PRIVATE_KEY: ${{ secrets.EC2_SSH_PRIVATE_KEY }}
            EC2_KNOWN_HOSTS: ${{ secrets.EC2_KNOWN_HOSTS }}
          run: |
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            printf '%s\n' "$EC2_PRIVATE_KEY" > ~/.ssh/deploy_key
            chmod 600 ~/.ssh/deploy_key
            printf '%s\n' "$EC2_KNOWN_HOSTS" > ~/.ssh/known_hosts
            chmod 600 ~/.ssh/known_hosts

        - name: Copy Backend to EC2
          env:
            EC2_HOST: ${{ secrets.EC2_HOST }}
            EC2_USER: ${{ secrets.EC2_USER }}
          run: |
            rsync -az --delete \
              -e "ssh -i ~/.ssh/deploy_key" \
              --exclude ".env" \
              --exclude "venv/" \
              --exclude ".venv/" \
              --exclude "__pycache__/" \
              --exclude ".git/" \
              backend/ \
              "${EC2_USER}@${EC2_HOST}:/opt/myapp/backend/"

        - name: Install Dependencies and Restart API
          env:
            EC2_HOST: ${{ secrets.EC2_HOST }}
            EC2_USER: ${{ secrets.EC2_USER }}
          run: |
            ssh -i ~/.ssh/deploy_key \
              "${EC2_USER}@${EC2_HOST}" << 'EOF'
              set -e
              cd /opt/myapp/backend
              /opt/myapp/venv/bin/pip install -r requirements.txt
              sudo systemctl restart fastapi
              sudo systemctl is-active --quiet fastapi
              EOF
  ```

  Important: This example assumes Vite, which generates the frontend build in `frontend/dist/`. If you use Create React App, change `dist/` to `build/`. It also assumes your FastAPI application is in `backend/` and the EC2 deployment user can write to `/opt/myapp/backend/`.

  ## 2. Configure the EC2 FastAPI service

  Your backend directory could look like this:

  text
  ```
  backend/
  ├── app/
  │   ├── __init__.py
  │   └── main.py
  └── requirements.txt
  ```

  In `requirements.txt`, ensure you have:

  text
  ```
  fastapi
  uvicorn
  gunicorn
  ```

  On EC2, create the application environment and directory:

  bash
  ```
  sudo mkdir -p /opt/myapp/backend
  sudo chown -R $USER:$USER /opt/myapp

  python3 -m venv /opt/myapp/venv
  ```

  Create `/etc/systemd/system/fastapi.service`:

  ini
  ```
  [Unit]
  Description=FastAPI Gunicorn Service
  After=network.target

  [Service]
  User=ubuntu
  Group=www-data
  WorkingDirectory=/opt/myapp/backend
  ExecStart=/opt/myapp/venv/bin/gunicorn \
      -w 2 \
      -k uvicorn.workers.UvicornWorker \
      app.main:app \
      --bind 127.0.0.1:8000

  Restart=always
  RestartSec=3

  [Install]
  WantedBy=multi-user.target
  ```

  Replace `ubuntu` with your EC2 deployment username if different. The example assumes your application object is named `app` in `app/main.py`.

  Enable the service:

  bash
  ```
  sudo systemctl daemon-reload
  sudo systemctl enable --now fastapi
  sudo systemctl status fastapi
  ```

  If you use a `.env` file, create it directly on EC2 and configure the systemd service to load it. Do not commit it to GitHub.

  ## 3. Configure Nginx

  Example Nginx configuration at `/etc/nginx/sites-available/fastapi`:

  nginx
  ```
  server {
      listen 80;
      server_name api.example.com;

      location / {
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

  Enable and reload it:

  bash
  ```
  sudo ln -s /etc/nginx/sites-available/fastapi \
    /etc/nginx/sites-enabled/fastapi

  sudo nginx -t
  sudo systemctl reload nginx
  ```

  Replace `api.example.com` with your API domain. Configure HTTPS using a valid TLS certificate before exposing the API publicly.

  ## 4. Add GitHub secrets and variables

  Go to GitHub → Repository → Settings → Secrets and variables → Actions.

  | Name | Type | Purpose |
  | --- | --- | --- |
  | `AWS_ACCESS_KEY_ID` | Secret | IAM access key |
  | `AWS_SECRET_ACCESS_KEY` | Secret | IAM secret key |
  | `AWS_REGION` | Secret | For example, `ap-south-1` |
  | `AWS_S3_BUCKET` | Secret | Frontend bucket name |
  | `EC2_HOST` | Secret | EC2 public IP or DNS |
  | `EC2_USER` | Secret | SSH username, e.g. `ubuntu` |
  | `EC2_SSH_PRIVATE_KEY` | Secret | Private SSH deployment key |
  | `EC2_KNOWN_HOSTS` | Secret | Verified EC2 SSH host key |
  | `VITE_API_URL` | Variable | For example, `https://api.example.com` |

  The EC2 security group must allow SSH from your deployment runner and HTTP/HTTPS traffic as required. For stronger security, consider AWS OIDC for S3 and a self-hosted runner or another restricted deployment path for EC2.

  ## 5. Deployment flow

  Push to `main`

  React frontend

  Install → Build → Upload to S3

  FastAPI backend

  SSH → Sync files → Install dependencies → Restart Gunicorn

  Both deployment jobs run independently.

  One final detail: the workflow assumes your EC2 user has passwordless `sudo` permission to restart the `fastapi` service. Configure a narrowly scoped sudoers rule for that command, and ensure the directory ownership matches the SSH deployment user. If the frontend uses a different API URL variable or build directory, update those values accordingly.



  =======================




  Let's learn GitHub Actions step by step, using your React + FastAPI deployment project as the example. We'll start with the basics and gradually understand every line of your workflow.

  # Learning roadmap

  1\. What is GitHub Actions?

  Understand CI/CD and automation.

  Start here

  2\. Workflow YAML structure

  Learn name, on, jobs, steps, and uses.

  3\. Triggers and events

  Push, pull requests, and manual runs.

  4\. Jobs and runners

  Understand Ubuntu runners and job execution.

  5\. Steps, actions, and commands

  Checkout, npm, SSH, and AWS CLI.

  6\. Secrets and variables

  Protect credentials and configure deployments.

  7\. Deploy React to S3

  Build, upload, and manage caching.

  8\. Deploy FastAPI to EC2

  SSH, rsync, Gunicorn, systemd, and Nginx.

  ## Lesson 1: What is GitHub Actions?

  GitHub Actions is a tool that automatically runs tasks when something happens in your GitHub repository.

  For example, without GitHub Actions, deploying your portfolio might require you to:

  1. Build your React application manually.
  2. Upload the generated files to S3.
  3. Connect to EC2 using SSH.
  4. Update your FastAPI code.
  5. Restart Gunicorn.

  With GitHub Actions, you push your code to GitHub, and these tasks can run automatically.

  ### Understand CI/CD

  - CI — Continuous Integration: Automatically build and test your code when changes are pushed.
  - CD — Continuous Delivery/Deployment: Automate preparing or releasing your application. Continuous deployment means changes are released automatically when the required checks pass.

  ### How it works

  You write code

  React + FastAPI

  Push to GitHub

  git push origin main

  GitHub Actions starts

  Reads your YAML workflow

  React

  Build → S3

  FastAPI

  Deploy → EC2

  ## Your first workflow

  Let's create the smallest useful workflow.

  File location: `.github/workflows/hello.yml`

  yaml
  ```

  name: My First Workflow

  on:
    push:
      branches:
        - main

  jobs:
    hello:
      runs-on: ubuntu-latest
      steps:
        - name: Print Message
          run: echo "Hello from GitHub Actions!"
  ```

  What each part means:

  | YAML | Meaning |
  | --- | --- |
  | `name` | Name shown in the GitHub Actions interface |
  | `on` | Event that triggers the workflow |
  | `push` | Run when code is pushed |
  | `branches: main` | Only trigger for pushes to `main` |
  | `jobs` | Collection of jobs to execute |
  | `hello` | Identifier for this job |
  | `runs-on` | Machine environment that runs the job |
  | `steps` | Tasks performed by the job |
  | `run` | Shell command executed by the runner |

  The runner is a temporary virtual machine hosted by GitHub in this example. It executes your commands; it is not your EC2 server.

  ### Try it yourself

  1. Create `.github/workflows/hello.yml` in your repository.
  2. Commit and push it to `main`.
  3. Open the repository's Actions tab.
  4. Select My First Workflow and open its run.
  5. Expand the `Print Message` step and find the output.

  You should see:

  text
  ```
  Hello from GitHub Actions!
  ```

  ## Quick check

  Question 1: In your workflow, what is the purpose of `runs-on: ubuntu-latest`?

  a It selects the machine environment that runs the job.

  It selects the EC2 instance where FastAPI runs.

  It selects the AWS region for the S3 bucket.

  Correct\!

  GitHub provides an Ubuntu runner to execute your workflow's steps.

  Continue to Lesson 2

  Next, we'll learn the workflow's YAML structure in detail, including the difference between `uses:` and `run:` and why indentation matters.


  ============================


  # Lesson 2: GitHub Actions YAML Structure

  Today, we'll understand the six main parts of a GitHub Actions workflow: `name`, `on`, `jobs`, `steps`, `uses`, and `run`.

  By the end, you'll be able to read your React + S3 and FastAPI + EC2 deployment workflow and understand what each line does.

  ## 1. First, understand the complete structure

  A GitHub Actions workflow is a YAML file stored inside your repository.

  File path:

  text
  ```
  .github/
  └── workflows/
      └── deploy.yml
  ```

  Here is a simple example containing all six concepts.

  yaml
  ```

  name: Portfolio Deployment

  on:
    push:
      branches:
        - main

  jobs:
    deploy:
      runs-on: ubuntu-latest

      steps:
        - name: Checkout Code
          uses: actions/checkout@v4

        - name: Print Message
          run: echo "Starting deployment"
  ```

  Think of this file as a set of instructions that tells GitHub what to do, when to do it, and how to execute it.

  ## 2. `name` — Give your workflow a name

  yaml
  ```
  name: Portfolio Deployment
  ```

  The `name` field gives the workflow a readable name in the GitHub Actions interface.

  For your project, you could use:

  yaml
  ```
  name: Deploy React and FastAPI
  ```

  When you open your repository's Actions tab, GitHub displays this name.

  Important distinction:

  - Workflow name: `Deploy React and FastAPI`
  - Job name: `deploy`
  - Step name: `Checkout Code`

  These names describe different levels of the workflow.

  ## 3. `on` — Decide when the workflow runs

  The `on` field specifies the events that trigger your workflow.

  For example:

  yaml
  ```
  on:
    push:
      branches:
        - main
  ```

  Let's break it down:

  | Line | Meaning |
  | --- | --- |
  | `on:` | Define workflow triggers |
  | `push:` | Trigger when code is pushed |
  | `branches:` | Restrict the trigger to specific branches |
  | `- main` | Only pushes to `main` trigger this workflow |

  Suppose you execute:

  bash
  ```
  git add .
  git commit -m "Update portfolio"
  git push origin main
  ```

  GitHub detects the push and starts the workflow.

  A push to another branch, such as `development`, will not trigger this workflow through the configuration above.

  You can also enable manual execution:

  yaml
  ```
  on:
    push:
      branches:
        - main
    workflow_dispatch:
  ```

  Now the workflow can run either when you push to `main` or when you manually start it from the Actions tab.

  Remember: `on` defines when the workflow starts, not what the workflow does.

  ## 4. `jobs` — Divide the work into jobs

  A job is a group of steps that execute together on a runner.

  Your application has two deployment tasks:

  - Build and upload React files to S3.
  - Deploy FastAPI code to EC2.

  You can represent them as separate jobs:

  yaml
  ```
  jobs:
    build-frontend:
      runs-on: ubuntu-latest
      steps:
        - name: Build React
          run: echo "Building React"

    deploy-backend:
      runs-on: ubuntu-latest
      steps:
        - name: Deploy FastAPI
          run: echo "Deploying FastAPI"
  ```

  Here, `build-frontend` and `deploy-backend` are job identifiers.

  By default, these jobs can run in parallel. That means the backend deployment doesn't automatically wait for the frontend deployment.

  If the backend job must wait for the frontend job, you can add:

  yaml
  ```
  deploy-backend:
    needs: build-frontend
  ```

  This makes `deploy-backend` depend on `build-frontend`.

  A useful detail: each job normally receives its own runner. Files created in one job aren't automatically available in another job. If you need to transfer build artifacts between jobs, you must explicitly share them.

  ## 5. `steps` — Define the tasks inside a job

  A step is an individual task within a job. Steps run sequentially, in the order you list them.

  For example:

  yaml
  ```
  steps:
    - name: Checkout Code
      uses: actions/checkout@v4

    - name: Install Dependencies
      run: npm ci

    - name: Build React
      run: npm run build
  ```

  The sequence is:

  Step 1: Checkout Code

  Get the repository files

  Step 2: Install Dependencies

  Install Node.js packages

  Step 3: Build React

  Generate production files

  If a step fails, later steps in the same job are normally skipped unless you explicitly configure different execution conditions.

  For React, the build produces files such as `index.html`, JavaScript bundles, CSS, and assets. For FastAPI, steps might install Python packages, copy code over SSH, and restart a systemd service.

  ## 6. `uses` — Run an existing action

  The `uses` keyword tells GitHub Actions to execute a reusable action.

  Example:

  yaml
  ```
  - name: Checkout Code
    uses: actions/checkout@v4
  ```

  What does this mean?

  - `name`: A descriptive label for the step.
  - `uses`: Select an existing action.
  - `actions/checkout`: The action that checks out your repository.
  - `@v4`: The major version reference for that action.

  GitHub-hosted runners are temporary, so you generally need to check out your repository before using its source files.

  Another example from your deployment:

  yaml
  ```
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: ${{ secrets.AWS_REGION }}
  ```

  Here, `with` supplies inputs to the action. The action configures AWS credentials so later commands can access AWS.

  The `${{ secrets.AWS_ACCESS_KEY_ID }}` expression reads a secret configured in your repository's GitHub settings. You should never hardcode your AWS secret key into the workflow.

  Important: `uses` isn't limited to AWS or GitHub's own actions. It can reference reusable actions published by other developers or organizations, so review third-party actions and pin them appropriately for production security.

  ## 7. `run` — Execute a shell command

  The `run` keyword executes shell commands on the runner.

  Example:

  yaml
  ```
  - name: Print Message
    run: echo "Hello GitHub Actions"
  ```

  This prints a message in the workflow logs.

  For your React deployment:

  yaml
  ```
  - name: Install Dependencies
    run: npm ci

  - name: Build React
    run: npm run build
  ```

  These are ordinary shell commands, not GitHub Actions themselves.

  For your FastAPI deployment, you might use:

  yaml
  ```
  - name: Restart FastAPI
    run: |
      ssh ubuntu@YOUR_EC2_HOST \
        "sudo systemctl restart fastapi"
  ```

  The `|` symbol allows a multiline shell script. The SSH command connects to EC2 and requests a service restart. In a real workflow, you must configure SSH authentication and host verification first.

  ### Difference between `uses` and `run`

  | `uses` | `run` |
  | --- | --- |
  | Executes an existing action | Executes shell commands |
  | Used for checkout and credential setup | Used for npm, AWS CLI, SSH, and scripts |
  | Inputs are commonly provided through `with` | Commands can use shell syntax and environment variables |

  A useful rule: use `uses` when you want a reusable action; use `run` when you want to execute a command or script.

  ## 8. Put everything together

  Here's a complete small workflow that uses all six concepts:

  yaml
  ```

  name: React Build Example

  on:
    push:
      branches:
        - main

  jobs:
    build:
      runs-on: ubuntu-latest

      steps:
        - name: Checkout Code
          uses: actions/checkout@v4

        - name: Setup Node.js
          uses: actions/setup-node@v4
          with:
            node-version: 22

        - name: Install Dependencies
          run: npm ci

        - name: Build React
          run: npm run build
  ```

  This example assumes your React project and `package-lock.json` are in the repository root. If your React project is in `frontend/`, you would configure the job's working directory or explicitly run commands from that directory.

  The workflow checks out your code, sets up Node.js, installs packages, and builds React. It does not deploy anything to S3 yet.

  ## 9. Test your understanding

  ## Lesson 2 quiz

  5/5

  1. Which keyword defines when a workflow starts?

  jobs

  on

  uses

  steps

  Correct

  The on section defines triggering events such as pushes and pull requests.

  2. What does \`runs-on: ubuntu-latest\` specify?

  The EC2 deployment server

  The AWS region

  The runner environment for the job The runner environment for the job

  The Git branch

  Correct

  GitHub provisions an Ubuntu runner to execute that job.

  3. Which keyword runs a command such as \`npm ci\`?

  uses

  name

  run run

  on

  Correct

  The run keyword executes shell commands.

  4. Which keyword invokes \`actions/checkout@v4\`?

  run

  uses uses

  jobs

  with

  Correct

  The uses keyword references an existing action.

  5. By default, what is the relationship between two independent jobs?

  They always run sequentially

  They run only on separate pushes

  They can run in parallel They can run in parallel

  They share one runner and filesystem

  Correct

  Independent jobs may run concurrently; use needs to express a dependency.

  Check answers again

  ## What comes next?

  Lesson 3 will cover GitHub Actions triggers and events in more detail, including `push`, `pull_request`, `workflow_dispatch`, branch filters, and how to run your deployment only when the appropriate code changes.


  =========================


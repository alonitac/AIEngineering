# The Yolo Service - AWS Deployment with CI/CD

## Background

In this project you'll apply cloud networking, Linux services, and CI/CD principles together in a realistic multi-environment deployment on AWS.

You'll build isolated dev and prod networks, run the Yolo service as a Linux service on each, and wire up automated deployment pipelines that ship code to the right environment - dev or prod - based on which branch was pushed.

## Part I: Build a multi-environment network

![poly_yolo_aws][poly_yolo_aws]


Create a VPC with **4 public subnets** split across two environments:

| Environment | Subnet name   | CIDR           |
|-------------|---------------|----------------|
| Dev         | dev-subnet-1  | 10.0.0.0/24    |
| Dev         | dev-subnet-2  | 10.0.1.0/24    |
| Prod        | prod-subnet-1 | 10.0.128.0/24  |
| Prod        | prod-subnet-2 | 10.0.129.0/24  |

Make sure:

- All four subnets share a single **Internet Gateway**. The Internet Gateway is **not** configured in the default route table. Create a custom route table, add a `0.0.0.0/0` route pointing to the Internet Gateway, and associate all four subnets to it.
- Each dev subnet is in a **different Availability Zone**. Same for the prod subnets.
- Dev and prod subnets should not be able to talk to each other. There are multiple ways to achieve this.
- **No need** to create a NAT gateway in this project.

## Part II: Launch the Yolo service

1. Create a `t3.small` EC2 instance in one of the **dev** subnets and another `t3.small` instance in one of the **prod** subnets.

   For each instance, allocate an **Elastic IP** and associate it. Elastic IPs are static - they don't change when you stop and start an instance, which is important for the CD pipeline in Part III.


2. SSH into your instances and launch the Yolo app as a linux service:
  - Install required packages.
  - Clone the repo.
  - Create a Python virtual environment and install dependencies
  - Create a systemd service file that runs the app on startup


## Part III: Continuous Deployment pipeline

Set up a GitHub Actions workflow that automatically deploys the Yolo app to the correct instance on every push:

- push to `dev` - deploy to the **dev** instance
- push to `main` - deploy to the **prod** instance

1. Store the following secrets in your GitHub repository

   | Secret name             | Value                                           |
   |-------------------------|-------------------------------------------------|
   | `DEV_INSTANCE_IP`       | Elastic IP of the dev instance                  |
   | `PROD_INSTANCE_IP`      | Elastic IP of the prod instance                 |
   | `DEV_INSTANCE_SSH_KEY`  | Full contents of the `.pem` key file for dev    |
   | `PROD_INSTANCE_SSH_KEY` | Full contents of the `.pem` key file for prod   |

> [!NOTE]
> When pasting the `.pem` file, include the full text - the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines are part of the key.

2. Create `.github/workflows/deploy.yaml` in your repository (this is just an example - you can structure your workflow entirely different if you want):

   ```yaml
   name: Deploy Yolo

   on:
     push:
       branches:
         - main
         - dev

   jobs:
     deploy-dev:
       if: github.ref == 'refs/heads/dev'
       runs-on: ubuntu-latest
       steps:
         - name: Deploy to Dev instance
           uses: appleboy/ssh-action@v1
           with:
             host: ${{ secrets.DEV_INSTANCE_IP }}
             username: ubuntu
             key: ${{ secrets.DEV_INSTANCE_SSH_KEY }}
             script: |
               # YOUR COMMANDS HERE...

     deploy-prod:
       if: github.ref == 'refs/heads/main'
       runs-on: ubuntu-latest
       steps:
         - name: Deploy to Prod instance
           uses: appleboy/ssh-action@v1
           with:
             host: ${{ secrets.PROD_INSTANCE_IP }}
             username: ubuntu
             key: ${{ secrets.PROD_INSTANCE_SSH_KEY }}
             script: |
                # YOUR COMMANDS HERE...
   ```

3. Test the dev pipeline: make a small harmless change directly on the `dev` branch (add a comment in `app.py`, for example), push it, and watch the **Actions** tab. The `deploy-dev` job should trigger and deploy to your dev instance.

   SSH into the dev instance afterward and confirm the service restarted cleanly:

   ```bash
   journalctl -u yolo.service
   ```

## Part IV: The full dev - prod flow

Now you'll practice the complete feature-branch workflow end-to-end, and implement **graceful termination** functionality for the Yolo app along the way.

#### Step 1 - Protect the main branch

Before writing any code, configure a branch ruleset so that PRs into `main` can only be merged after CI passes:

1. In your repository on GitHub, go to **Settings** - **Rules** - **Rulesets** - **New branch ruleset**.
2. Set the target branch to `main`.
3. Under **Branch rules**, enable **Require a pull request before merging**, **Block force pushes** and **Require status checks to pass**, then add your CI job (the `test` job from `.github/workflows/test.yaml`).
4. Save the ruleset.

From now on, the **Merge pull request** button on any PR into `main` will be disabled until all required checks pass.

#### Step 2 - Create a feature branch

```bash
git checkout main
git pull origin main
git checkout -b feature/graceful-termination
```

#### Step 3 - Implement graceful termination

Implement as taught in the [Processes tutorial](../tutorials/linux_processes.md).

When systemd stops the service (e.g. during a CD pipeline), it sends `SIGTERM` first. Instead of dying immediately, the app shuts down gracefully, logs a shutdown message, and exits. You'll see this in the logs in the next step.

Commit your changes.

#### Step 4 - Merge into dev and verify the deployment

1. Merge your feature branch locally into `dev` and push:

   ```bash
   git checkout dev
   git merge feature/graceful-termination
   git push origin dev
   ```

2. Watch the **Actions** tab - the `deploy-dev` job should trigger automatically.

3. SSH into the dev instance and tail the service logs to confirm the graceful shutdown message appears when the service restarts:

   ```bash
   journalctl -u yolo.service
   ```

   You should see your graceful shutdown log line before the service comes back up.

#### Step 5 - Open a PR into main

Now that you tested your feature in the dev environment, it's time to merge it into prod:

1. Push your feature branch:

   ```bash
   git push origin feature/graceful-termination
   ```

2. On GitHub, open a Pull Request from `feature/graceful-termination` into `main`.
3. Wait for the CI checks to pass. If they fail, read the logs carefully, fix the issue, and push again.
4. Once CI passes, merge the PR.


> [!NOTE]
> Don't forget to stop your EC2 instances when you're done to avoid unnecessary charges.

## Good Luck


[poly_yolo_aws]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/yolo_network_fursa.png
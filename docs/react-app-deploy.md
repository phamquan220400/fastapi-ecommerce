# React on S3 + CloudFront deployment

This GitHub Actions workflow builds a React single-page app, publishes its output to S3, and clears the CloudFront cache. It runs after pushes to `main` that change files under `fastcomm/`; you can also start it manually on `main` regardless of which files changed.

## Add it to your app

Copy [`.github/workflows/deploy-react-app.yml`](../.github/workflows/deploy-react-app.yml) into the same path in your React repository. The workflow uses `npm ci` and `npm run build`, so the repository needs a committed `package-lock.json` and a `build` script in `package.json`.

The default build output directory is `dist` (Vite). For Create React App, set `BUILD_DIR` to `build`.

The workflow currently runs npm from the repository root. This repository's `fastcomm/` app is a Next.js project in a subdirectory and is not configured for static export; deploying it to S3 requires adapting the workflow's build paths and configuring a compatible static export first.

## AWS setup

1. Create an S3 bucket for the built site. Keep S3 Block Public Access enabled.
2. Create a CloudFront distribution with that bucket as a private S3 origin and Origin Access Control (OAC). Grant the distribution access to read the bucket objects.
3. Configure the distribution's custom error responses for a client-side React router: map origin `403` and `404` responses to `/index.html` with viewer response code `200`.
4. Add GitHub as an IAM OIDC identity provider if your AWS account does not already have one: provider URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
5. Create an IAM role trusted by that provider. Restrict the `sub` claim to your repository and the `main` branch. Replace `YOUR_GITHUB_OWNER` and `YOUR_REPOSITORY` in this trust policy:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_OWNER/YOUR_REPOSITORY:ref:refs/heads/main"
           }
         }
       }
     ]
   }
   ```

6. Attach the permissions in [`react-deploy-permissions.json`](../terraform/deploy/react-deploy-permissions.json) to that role. Replace the bucket, AWS account, and CloudFront distribution placeholders.
7. In GitHub, open **Settings → Secrets and variables → Actions → Variables** and add:

   | Variable | Value |
   | --- | --- |
   | `AWS_ROLE_ARN` | ARN of the IAM role from step 5 |
   | `AWS_REGION` | Region containing the S3 bucket (optional; defaults to `us-east-1`) |
   | `S3_BUCKET` | S3 bucket name |
   | `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront distribution ID |
   | `BUILD_DIR` | Build output path (optional; defaults to `dist`) |
   | `NODE_VERSION` | Node.js major version (optional; defaults to `22`) |

No long-lived AWS access keys are stored in GitHub. The workflow receives short-lived credentials through OIDC. The role is scoped to the `main` branch; manual runs must also select `main`.

## Cache behavior

Uploaded files receive a one-hour cache lifetime, while `index.html` is set to revalidate on every request. The workflow then requests a CloudFront invalidation for `/*` so the new deployment is visible promptly.

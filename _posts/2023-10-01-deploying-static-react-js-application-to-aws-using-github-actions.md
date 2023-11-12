---
layout: default
title: Deploying a Static React.JS application to AWS using Github Actions
category: reactjs
tags:
- react.js
- webpack
- AWS
- github
- javascript
---

A static web application is an application delivered directly to an end user’s browser without any server-side modification. While a web server is still used, it primarily serves application files directly from disk. For example, an incoming request to `https://mywebsite.com/` would result in returning the `index.html` file from the server root directory. If the returned HTML document references any additional assets (e.g images, css, javascript files), additional requests to the web server would be made, and those files would be resolved from their relative location on the filesystem.

In this post, we’ll look at deploying a static site created with [Create React App](https://create-react-app.dev/) to AWS via Github Actions. We'll require the following AWS services:

- Amazon S3 to store our application assets
- CloudFront to serve our application over HTTP
- IAM to provide a role for deploying our application. The IAM role should have write permissions to the S3 bucket and `createInvalidation` permissions to our CloudFront distribution.

Configuring the AWS services is outside the scope of this article. I highly recommend leveraging the [CloudPosse AWS CloudFront/S3 CDN module](https://github.com/cloudposse/terraform-aws-cloudfront-s3-cdn) or reading through this [excellent blog post](https://medium.com/runatlantis/hosting-our-static-site-over-ssl-with-s3-acm-cloudfront-and-terraform-513b799aec0f) and configuring the resources via Terraform manually.

Let’s get started with our `build.yml` definition. This file will live in the `.github/workflows/` directory, and should be committed into your repository (and not .gitignored). Let’s start by giving our job a name and defining when it’s run.

```
name: Build and publish

on:
  push:
    tags:
      - '*'
```

For the above configuration, our job will be run anytime a new tag is published. Git tags are a great way to manage code releases. Ideally artifacts are not modified in-between environments. Instead, a single artifact progresses from int → stg → prd. Unfortunately, this isn’t always possible for client/static applications. Static applications typically inject configuration during their build stage. For example, the HTTP endpoint for your API generally differs between pre-production and production. Additionally, pre-production static sites can opt to enable debugging aids such as sourcemaps, which would typically be disabled for production.

With these limitations, we’ll opt for rebuilding the application for each environment, and use git tags to identify which environment is being targeted. Let’s take a look at the entire build definition:

```
{% raw %}
name: Build and publish

on:
  push:
    tags:
      - '*'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set TARGET_ENV
        run: |
          if [[ '${{ github.ref_name }}' =~ -(int|prd)$ ]]; then
            TARGET_ENV=${BASH_REMATCH[1]}
            echo "TARGET_ENV=$TARGET_ENV" >> $GITHUB_ENV
          else
            echo "Invalid tag"
            exit 1;
          fi

      - name: Install dependencies
        uses: actions/setup-node@v2
        with:
          node-version: '16.x'
          cache: 'npm'
      - run: npm install

      - name: Build project
        run: |
          cp ".env.${TARGET_ENV}" .env
          npm run build

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-region: ${{ fromJSON(secrets.AWS_CREDENTIALS)[env.TARGET_ENV].region }}
          aws-access-key-id: ${{ fromJSON(secrets.AWS_CREDENTIALS)[env.TARGET_ENV].accessKeyId }}
          aws-secret-access-key: ${{ fromJSON(secrets.AWS_CREDENTIALS)[env.TARGET_ENV].secretAccessKey }}

      - name: Deploy
        run: |
          aws s3 sync build/ s3://${{ fromJSON(vars.BUILD_CONFIG)[env.TARGET_ENV].bucket }}
          aws cloudfront create-invalidation --distribution-id ${{ fromJSON(vars.BUILD_CONFIG)[env.TARGET_ENV].distributionId}}--paths "/"
{% endraw %}
```

Our build has a total of six steps.

- Pull source code
- Parse our git tag and set `TARGET_ENV` variable. This is used later to look up configuration values stored in our project settings.
- Here, we run `npm install` to install our application dependencies. We are not running this with the `--production` flag since we require dev dependencies for the next step. For a static application, another option is to move build-deps into the `dependencies` section in your `package.json` and save `devDependencies` for local development tools. Doing this allows you to run `npm install --production` to avoid installing those packages during the build stage.
- Build/compile our application code. The first line of this step is especially important. `cp ".env.${TARGET_ENV}" .env` Here, we are selecting the `.env.${env}` file matching the target environment and renaming it to `.env`. If you are using `REACT_APP_${var}` style environment variables in your application, Create React App supports using `.env` files to configure values for injecting during the webpack execution. For more information, check the [official create react app docs](https://create-react-app.dev/docs/adding-custom-environment-variables/#what-other-env-files-can-be-used).
- Configure our AWS credentials. This step makes use of a repository secret, which is configured by visiting `settings > secrets and variables > Actions`. We create a new secret named `AWS_CREDENTIALS` with the following format:

```
{
  "int": {
    "region": "...",
    "accessKeyId": "...",
    "secretAccessKey": "..."
  },
  "prd": {...}
}
```

- The last step in our build is to deploy our artifact to S3 and invalidate our CloudFront cache. Like our previous step, we are also making use of a JSON value to define our build parameters. Instead of a secret this time, we store this value as a “variable” in our repository settings. Our new variable is called `BUILD_CONFIG` and has the following format:

```
{
  "int": {
    "bucket": "...",
    "distributionId": "..."
  },
  "prd": {}
}
```

And that’s it! By using Github secrets & variables, we are able to keep our build definition generic and reusable for other applications. Additionally, adding environments or updating build parameters doesn’t require committing code, and can be done by updating our repository settings. By leveraging `.env` files, we're able to modify our artifact for a given environment, enabling things like source maps in pre-production (e.g by adding `GENERATE_SOURCEMAP=true` in .env.int) while ensuring our production build is optimized.
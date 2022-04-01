# CI/CD For deploying GCF

While preparing for a new enterprise grade project we had a new challenge. The project will benefit from Google Cloud Platform and will contain 500+ Google Cloud Functions as a part of serverless components.

During early meetings for our source control architecture, we decided to contain all Google Cloud Functions into single repo (Mono-Repo). The challenge was how to implement CI/CD pipeline for the functions.

I decided to take care of the issue and automate the build pipeline, my criteria were as follow:
1.	Build/Deploy must be triggered through pushing to GitHub
2.	Only changed functions will be rebuilt and deployed to optimize resource allocation and reduce total build time.
3.	All functions must remain in a single GitHub repository.
4.	The process must be totally transparent, so developers remain in focus.

So, to meet the criteria and accomplish the task, I wrote down the needs:
1.	Process start trigger
2.	Identify changed functions
3.	Loop through changed functions and deploy one-by-one

My solution was to depend on GitHub actions, which is very good tool for CI/CD automation with broad spectrum of flexibility. Here are the steps:

## Step 1: Structure Mono Repo
1.	On my development WSL created directory to host all of project functions and acts as Mono-Repo
2.	Initiated git repository, created `.gitignore` file and other required files.
3.	Created separate directory for each function, the convention used:
    - each function is in a sub-directory
    - directory name must start with `func`, followed by function name.
    - function entry point (.js file) and exported function must be same as folder name

## Step 2: Build the trigger
As my repo is ready, I created new repository in our GitHub organizations account, and added it as remote to the local mono repo, then head to Actions tab to create new CI/CD action. Action triggered for each new push to main branch, see the code
```yaml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
```

## Step 3: Identify Changed Functions
This is core of the solution! To identify changed functions, I used `git diff` command. `git diff` is part of Git plumbing tools. It can be used to get changed files or directories between two commits, branches, etc. So, I used `git diff`  and piped results into series of manipulations to get directories only, filter for directories contains `func` word (to follow the convention) and convert final result into `JSON` like array of elements. The result will be like this
```json
[“func1”, “func3”, “fun10,”, …]
```
GitHub actions has a nice feature called `matrix` that holds elements you can loop on. To this step, I used the matrix to store JSON array.

## Step 4: Loop through change functions and deploy on-by-one
Once I have the matrix, it was easy to run [google-github-actions/deploy-cloud-functions@v0.1.2](https://github.com/google-github-actions/deploy-cloud-functions) action to deploy changed functions as GitHub actions will loop through each element of the matrix and deploy it one by one.

The full `yaml` file for GitHub Action is
```yaml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.getfile.outputs.matrix }}
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      - name: get-files
        id: getfile
        run: |
          echo "::set-output name=matrix::[$(git diff --name-only ${{ github.event.before }} ${{ github.sha }} | cut -d/ -f1 | sort -u | sed -n 's/^/"/;s/$/"/;s/\(func\)/\1/p' | sed -e ':a;N;$!ba;s/\n/, /g')]"
          
      - name: echo output
        run: |
          echo ${{ steps.getfile.outputs.matrix }}
      
  deploy:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      # This needs to match the first job's name and output parameter
      matrix: 
        func: ${{ fromJson(needs.build.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v2
      - id: 'runbuild'
        uses: google-github-actions/deploy-cloud-functions@v0.1.2
        with:
          credentials: ${{ secrets.SERVICE_ACCOUNT_KEY }} 
          project_id: ${{ secrets.PROJECT_ID }}
          name: ${{ matrix.func }}
          source_dir: "${{ matrix.func }}"
          entry_point: ${{ matrix.func }}
          runtime: nodejs16
          region: ${{ secrets.DEPLOY_REGION }}
          timeout: 60
          max_instances: 10
          # env_vars: # optional
```
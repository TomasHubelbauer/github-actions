# GitHub Actions

## Status Badge

See [github-actions-badge](https://github.com/TomasHubelbauer/github-actions-badge)
for GitHub Actions workflow status image badge MarkDown link syntax.

## Read Workflow

This workflow script proceeds to validate the repository without pushing any
artifacts to it.

```yml
name: github-actions
on:
  push:
    branches:
    # Limit to the `master` branch
    - master
jobs:
  github-actions:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Run the workflow
      run: |
        set -x
        # Run the script
        npm install
        npm start
```

## Write Workflow

This workflow scripts executes a command and then commits its outputs to the
repository associated with the workflow.

```yml
name: github-actions
on:
  push:
    branches:
    # Limit to the `master` branch
    - master
jobs:
  github-actions:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Run the scraper
      run: |
        set -x
        # Configure Git for the push from the workflow to the repository
        git config --global user.email "tomas@hubelbauer.net"
        git config --global user.name "Tomas Hubelbauer"
        # Check out the `master` branch because by default GitHub Actions checks out detached HEAD
        git checkout master
        # Run the command
        ./command.sh
        # Authenticate with GitHub using the out of the box workflow integration PAT
        git remote set-url origin https://tomashubelbauer:${{secrets.GITHUB_TOKEN}}@github.com/${{github.repository}}
        # Add the command output to the commit
        git add output/*
        # Reset unstaged changes to prevent `git commit` from yelling if there's e.g. `package-lock.json` or caches
        git checkout -- .
        # Commit the added changes to the repository associated with this workflow
        git commit -m "Commit changes from the workflow"
        # Rebase if the branch has meanwhile changed (fail if there are automatically irresolvable merge conflicts)
        git pull --rebase
        # Pus the commit to the repository associated with this workflow
        git push
```

## GitHub Pages Deployment

This GitHub Actions workflow builds a CRA application placed in a `cra` directory
and places the result into a `docs` directory from which GitHub Pages are hosted.

It requires a custom PAT to invoke the GitHub API and deploy the GitHub Pages,
because the integration PAT doesn't cause GitHub Pages to build on push and using
the custom PAT to push would cause an infinite GitHub Actions chain on the push.

It is imporant to set the `homepage` field of the CRA `package.json` so that the
built site has correct relative URLs since it is going to be hosted on a path
relative to the GitHub Pages domain.

```yml
name: cd
on:
  push:
    branches:
    # Limit to the `master` branch
    - master
  schedule:
    # Run hourly
    - cron:  '0 * * * *'
jobs:
  cd:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Generate GitHub Pages
      run: |
        set -x
        # Configure Git identity for the push from the workflow to the repository
        git config --global user.email "tomas@hubelbauer.net"
        git config --global user.name "Tomas Hubelbauer"
        # Check out `master` - by default GitHub Actions checks out detached HEAD
        git checkout master
        # Delete the existing `docs` directory as it will be re-generated
        rm -rf docs
        # Build the Create React App application for the GitHub Pages deployment
        cd cra
        npm install
        npm run build
        # Move the `build` directory to be the `docs` directory for GitHub Pages
        mv build ../docs
        # Stage the generated GitHub Pages directory
        git add docs
        # Reset unstaged changes if there are any outside of `docs` to prevent commit failure
        git checkout -- .
        # Exit if there are no new changes (due to scheduled run or non-code changes)
        if git diff-index --quiet HEAD --; then
          exit
        fi
        # Commit the changes to the Git repository to deploy to GitHub Pages
        git commit -m "Deploy to GitHub Pages"
        # Authenticate with GitHub using the default integration PAT (this one won't deploy GitHub Pages)
        git remote set-url origin https://tomashubelbauer:${{secrets.GITHUB_TOKEN}}@github.com/${{github.repository}}
        # Rebase before pushing to integrate fast forward changes if there are any
        git pull --rebase
        # Push the changes to GitHub where it will not deploy GitHub Pages due to the use of the integration PAT
        git push
        # Enqueue a GitHub Pages deployment using the API with the custom PAT (the integration PAT cannot call the API)
        curl -f -X POST -H "Authorization: token ${{secrets.GITHUB_PAGES_PAT}}" -H "Accept: application/vnd.github.mister-fantastic-preview+json" "https://api.github.com/repos/${{github.repository}}/pages/builds"
```

## Scheduled Runs

Add this to make the workflow run daily:

```yml
on:
  push:
    branches:
    # Limit to the `master` branch
    - master
  schedule:
    # Run daily
    - cron:  '0 0 * * *'
```


**Scheduled runs do not have access to the cache.**

## Email Notifications

There is no built in way to send email notifications from the workflow runs.
One has to code that up themselves. I've wrapped the implementation in a repo:

**To be able to clone the repo, one has to use a custom PAT, since it is private.**

```yml
# Send myself an email
git clone https://TomasHubelbauer:${{secrets.GITHUB_ACTIONS_PAT}}@github.com/TomasHubelbauer/self-email.git
chmod +x ./self-email/self-email.sh
echo "Hi." >> email.eml
cat email.eml | ./self-email/self-email.sh
```

## Cache

It is possible to cache things across workflow runs:

Change the Cron to `0 * * * *` to run hourly.

https://help.github.com/en/actions/automating-your-workflow-with-github-actions/caching-dependencies-to-speed-up-workflows

However, scheduled runs do not have access to the cache.

## To-Do

### Learn about artifacts and demonstrate them

https://help.github.com/en/actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts

### Go through the docs and document interesting features

### See if the workflow files can be simplifed like with Azure Pipelines

Go directly to the steps element and see if that works.

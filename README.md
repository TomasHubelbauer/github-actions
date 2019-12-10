# GitHub Actions

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

## Cache

It is possible to cache things across workflow runs:

Change the Cron to `0 * * * *` to run hourly.

https://help.github.com/en/actions/automating-your-workflow-with-github-actions/caching-dependencies-to-speed-up-workflows

However, scheduled runs do not have access to the cache.

## To-Do

### Learn about artifacts and demonstrate them

https://help.github.com/en/actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts

### Go through the docs and document interesting features

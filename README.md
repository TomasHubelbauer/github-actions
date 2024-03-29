# GitHub Actions

[**WEB**](https://tomashubelbauer.github.io/github-actions)

## Status Badge

See [github-actions-badge](https://github.com/TomasHubelbauer/github-actions-badge)
for GitHub Actions workflow status image badge MarkDown link syntax.

## Workflow File Location

`.github/workflows/$name.yml` - the `$name` can be whatever you like.

## Manual Runs

https://docs.github.com/en/actions/managing-workflow-runs/manually-running-a-workflow

## Read Workflow

This workflow script proceeds to validate the repository without pushing any
artifacts to it.

```yml
name: main
on: push

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
    - name: Check out the main branch
      uses: actions/checkout@v3
      with:
        ref: main

    - name: Run the workflow
      run: |
        # Print shell commands as they execute
        set -x
        
        # Run the script
        node .
```

## Write Workflow

**Note:** Writing back to the repository is also possible with REST (sole file)
and GraphQL (multiple files). Prefer this for new integrations!

https://github.com/TomasHubelbauer/github-actions-push-api

This workflow scripts executes a command and then commits its outputs to the
repository associated with the workflow.

```yml
name: main
on: push

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
    - name: Check out the main branch
      uses: actions/checkout@v3
      with:
        ref: main

    - name: Run the workflow
      run: |
        # Fail on error
        set -e
        
        # Print shell commands as they execute
        set -x
        
        # Configure Git for the push from the workflow to the repository
        # These credentials will make the commit associate with the GitHub Actions service account
        git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
        git config --global user.name "github-actions[bot]"
        
        # Run the script
        node .
        
        # Stage the Git index changes resulting from the CI script
        git add *
        
        # Reset unstaged changes so that Git commit won't fail (e.g.: package-lock.json, temporary files, …)
        git checkout -- .
        
        # Bail if there are no changes to commit and hence no GitHub Pages to build
        if git diff-index --quiet HEAD --; then
          exit
        fi
        
        # Commit the staged changes to the workflow repository
        git commit -m "Commit generated content"
        
        # Rebase if the branch has changed meanwhile or fail on automatically irresolvable conflicts
        git pull --rebase
        
        # Push the commit to the workflow repository
        # This will not cause an infinite loop, GitHub knows it is from the agent and will not run the workflow again
        git push
```

An alternative way to do this is to buy in to the proprietary GitHub Actions
syntax and get the benefit of the nicer UI. I would only do this for simple
scripts that are easy to convert back to Bash with just copy-paste if needed.

```yml
name: main
on:
  push:
  schedule:
    - cron: "0 0 * * *"

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      # We need to check out first so that we have a baseline to make a diff of
      - name: Check out the main branch
        uses: actions/checkout@v3
        with:
          ref: main

      # Set up Git identity before making any changes
      - name: Commit and push the change to the GitHub repository from the agent
        run: |
          # Configure Git for the push from the workflow to the repository
          # (This is needed even with the workflow PAT)
          # These credentials will make the commit associate with the GitHub Actions service account
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git config --global user.name "github-actions[bot]"

      # Carry out the workflow work
      - name: Run the workflow script or do some other stuff in more steps
        run: node .

      - name: Stage the changes resulting from the above steps
        run: git add *

      - name: Bail if there are no changes staged to commit
        id: bail
        continue-on-error: true
        run: |
          git status
          if git diff-index --quiet HEAD --; then
            echo "::set-output name=bail::true"
          else
            echo "::set-output name=bail::false"
          fi

      - name: Commit the staged changes to the workflow repository
        if: ${{steps.bail.outputs.bail == 'false'}}
        run: git commit -m "Capture workflow changes"

      - name: Rebase if the branch has changed meanwhile or fail on conflicts
        if: ${{steps.bail.outputs.bail == 'false'}}
        run: git pull --rebase

      - name: Push the commit to the workflow repository
        if: ${{steps.bail.outputs.bail == 'false'}}
        run: git push
```

## GitHub Pages Deployment Workflow

This GitHub Actions workflow builds on top of the write one above. GitHub Pages
can be hosted either from the root of the repository or the `docs` folder, so the
above workflow needs to be updated to write the contents the GitHub Pages should
have to the right directory first.

It requires a custom PAT to invoke the GitHub API and deploy the GitHub Pages,
because the integration PAT doesn't cause GitHub Pages to build on push and using
the custom PAT to push would cause an infinite GitHub Actions chain on the push.

In case of CRA, it is imporant to set the `homepage` field of the CRA `package.json`
so that the built site has correct relative URLs since it is going to be hosted on
a path relative to the GitHub Pages domain unless a custom domain is configured.

```yml
        … the workflow script so far …

        # Enqueue and monitor a GitHub Pages deployment using the GitHub API and the custom PAT
        # (The out-of-the-box PAT is an integration PAT - not privileged to make GitHub Pages API calls)
        
        # Authorize using the custom PAT which is privileged to call the GitHub Pages API
        authorization="Authorization: token ${{secrets.GITHUB_PAGES_PAT}}"
        
        pagesBuildsUrl="https://api.github.com/repos/${{github.repository}}/pages/builds"
        pagesBuildsJson="pages-builds.json"

        curl -s -f -X POST -H "$authorization" $pagesBuildsUrl > $pagesBuildsJson
        status=$(jq '.status' $pagesBuildsJson | tr -d '"')
        echo $status
        if [ "$status" != "queued" ]
        then
          exit 1
        fi

        pagesBuildsLatestUrl=$(jq '.url' $pagesBuildsJson | tr -d '"')
        pagesBuildsLatestJson="pages-builds-latest.json"

        rm $pagesBuildsJson
        while true
        do
          sleep 5
          curl -s -f -H "$authorization" $pagesBuildsLatestUrl > $pagesBuildsLatestJson
          status=$(jq '.status' $pagesBuildsLatestJson | tr -d '"')
          echo $status
          if [ "$status" = "built" ]
          then
            rm $pagesBuildsLatestJson
            exit
          fi
        done
```

## Scheduled Runs

Add this to make the workflow run daily:

```yml
on:
  push:
  schedule:
    - cron: "0 0 * * *"
```

Change to this to limit to the main branch:

```yml
on:
  push:
    branches:
    - main
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

## Notes

Unlike Azure Pipelines at which GitHub Actions are based on, the `steps` object
is required in GitHub Actions and cannot be skipped to go straight to the step.

## To-Do

### Learn about artifacts and demonstrate them

https://help.github.com/en/actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts

### Go through the docs and document interesting features

### Replace the Write Workflow section with a link to the experimental repo

https://github.com/TomasHubelbauer/github-actions-push-api

All of the new write workflows should be based on this and the old ones changed
to this.

### Validate whether the integration PAT token with Actions identity emits Pages

I am not sure what change, but I think Pages either get deployed on commits from
the workflow in general, or maybe only when using the integration PAT or maybe
even when impersonating the Actions service account (with the integration PAT?).

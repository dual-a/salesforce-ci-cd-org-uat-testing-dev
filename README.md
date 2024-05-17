# CI/CD for Org Development Model

This is a sample sfdx project that showcases how to use Github Actions for CI/CD in an org development model (no packages, no scratch orgs). Scratch orgs expire—they have a default expiration of 7 days and a max lifespan of 30 days, so we're using sandboxes in this pipeline.

Check out the `pr-develop-branch.yml` file under `.github/workflows` for a detailed explanation of the operations that are being automated.

#  Personal changes compared to forked repository

> [!IMPORTANT]
> I've made some changes to the forked repository. I've taken a different sandbox strategy that I'm more familiar with compared to the one in the original workflow. My sandbox strategy uses the following flow:

1. Production
2. UAT/Staging
3. Testing/QA
4. Individual developer sandboxes

Updated some of the .yml files to reflect the latest github actions, sfdx commands and plugin commands. This works better in my opinion.

## High level flow

**1-** We have a master branch that represents the metadata we are tracking. We also have a uat (or staging) branch that represents the uat enviroment which is how we're modeling our sandbox strategy.

**2-** At the beginning of a development sprint, we create a develop branch off of main, in the remote repository.  

**3-** Developers clone the repository (which is an sfdx project) to their local computer and authorize the project against their own sandbox (this is a one time step). This is an important and often overlooked feature of sfdx: you can authorize any sfdx project against any org, even if the project was created from a different org. 

**4-** Developers checkout the remote develop branch, and create a feature branch that corresponds to the bug or user story they are working on. 

**5-** Developers push their branch to the remote repository, and continue to push commits as they progress through the development process.

**6-** Once ready, the developer opens a pull request from the feature branch against the develop branch.

**7-** The pull request triggers a CI/CD job with Github Actions that will do the following:

Do a check-only deployment of only the new metadata or the existing metadata that has changed. The pr-develop-branch.yml file or the pr-develop-branch-no-codescan.yml file will trigger as well.

If the deployment passes, run the tests specified by the developer (explained below). 

Scan the apex code for any vulnerabilities or code smells. Log any issues directly in github. (scanning requires github enterprise if you're using [Code-QL] (https://codeql.github.com/) to scan as shown in this pipeline)

**8-** If the CI/CD job completes successfully, then the feature branch can be merged into testing.

**9-** The feature branch is merged into develop, and this triggers an actual deployment of the metadata into the testing org, and again runs the tests specified by the developer.

**10-** A pull request is created from the testing branch to the UAT branch, for UAT testing or staging.

**11-** At the end of the sprint, the UAT branch is merged into master, and this triggers a production deployment. 


## Deploying delta changes

We are using `sfdx-git-delta` to only deploy the metadata that has been changed (or created) by the developer. 

If you want to deploy the entire branch, simply use the `deploy` command against the `force-app` directory.

The plugin [sfdx-git-delta] (https://github.com/scolladon/sfdx-git-delta/tree/main) is an open source plugin that was created by [Sebastian] (https://github.com/scolladon) and allows us to compare the head of the branch to previous head. This allows us to take a delta of the code and reduce the resource allocation. Instead of having to deploy the whole repository each time, we reduce the workload to only files that were changed, added or deleted.

## Specify which tests to run

We allow the developer to specify which tests to run by using a special syntax in the pull request body

`Apex::[Class1,Class2,Class3...]::Apex`

Read `pr-develop-branch.yml` for a detailed description of how this works.

## Static code analysis

We are using the `sfdx scanner` to scan the code in the delta directory. 

We decided not to fail the entire job just because there are warnings. Instead, the warnings are logged directly in the PR for your team to review. We think this is better than failing the job because it allows your team to review the code and have a conversation about it. If the same warning keeps showing up every now and then, then it might be worth to configure the job to fail.


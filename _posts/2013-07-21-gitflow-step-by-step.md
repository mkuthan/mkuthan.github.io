---
title: "GitFlow step by step"
date: 2013-07-21
categories: [git, maven, jira]
---

Git Flow is a mainstream process for branch per feature development. Git Flow  is the best method I've found for 
managing project developed by small to medium project teams. Before you start reading this post you should read two 
mandatory lectures:

[Git Workflows by Atlassian](https://www.atlassian.com/git/workflows#!workflow-gitflow)

[Maven JGit-Flow Plugin](https://bitbucket.org/atlassian/jgit-flow/wiki/Home)

This blog post is a step by step instruction how to use Git Flow together with Maven build tool, continuous integration
server (e.g: Bamboo) and bug tracker (e.g: JIRA). If you are interested in how to automate the whole process, watch this
[Flow with Bamboo](https://www.youtube.com/watch?v=YIgX67c-2hQ) video. But I really recommend to start using Git Flow 
with pure git commands, when you understand the concept move to Git Flow, and then automate everything eventually.

## Start feature branch

1. Assign JIRA task to you.
2. Move JIRA tasks from "To Do" to "In Progress".
3. Create feature branch for the JIRA user story (if it is a first task of the user story). 
Feature branch must reflect JIRA issue number and have meaningful name, e.g: _PROJ-01_user_registration_.

        mvn jgitflow:feature-start
        
4. Verify that:
    * New local feature branch _feature/PROJ-01_user_registration_ is created.

5. Optionally push feature branch into remote repository.

        git push origin feature/PROJ-01_user_registration

6. Verify that:
    * The feature branch is pushed into remote repository.
    * New Bamboo build plan is created for the feature branch.

## Checkout the feature branch

1. Checkout the feature branch created by other developer (e.g for code review).

        git checkout feature/PROJ-01_user_registration
        
## Work on the feature branch
       
1. Periodically push changes to the remote repository.
 
        git push origin feature/PROJ-01_user_registration

2. Verify that:
    * Bamboo build plan for feature branch is green.
    
## Finish feature branch
    
1. Ensure your local develop branch is up to date.

        git checkout develop
        git pull origin develop

2. To avoid conflicts during finishing feature branch, ensure that all changes from develop are merged to the feature branch.

        git checkout feature/PROJ-01_user_registration
        git pull origin develop

3. Resolve all conflicts (if any) and commit changes.

        git commit -a -m "Conflicts resolved"

4. Finish the feature.

        mvn jgitflow:feature-finish

5. Push changes from develop into remote repository

        git push origin develop
        
6. Move JIRA task to "Done" category.

7. Verify that:
    * Feature branch is merged into develop branch.
    * Local feature branch is removed.
    * Bamboo build plan for develop is green.    
    
## Start release branch

1. Create release branch.

        mvn jgitflow:release-start

2. Verify that:
    * New local release branch release/version is created.
    * Work with release branch
    
## Work with release branch    
    
1. Clean the database (Database).

2. Run the application (Running Application) and perform exploratory tests.

3. Fix all issues (if any).

4. Commit changes to the release branch.

        git commit -a -m "Fixes release candidate"

## Finish release branch

1. Make sure your local master branch is up to date

        git fetch origin master

2. Finish the release branch

        mvn jgitflow:release-finish
    
3. Verify that:
    * Release branch is merged into local develop branch.
    * Project version is updated in local develop branch.

4. Push changes from develop into remote repository

        git push origin develop

5. Checkout master

        git checkout master

6. Verify that:
    * Release branch is merged into local master branch.
    * Project version is updated in local master branch.

7. Push changes from master into remote repository

        git push --tags origin master
    
7. Verify that:
    * Release tag is pushed to the remote repository.
    * Build plan on master is green and new version is deployed.
    
8. Delete released feature branches from remote repository.

        git push origin :feature/PROJ-01_user_registration
        
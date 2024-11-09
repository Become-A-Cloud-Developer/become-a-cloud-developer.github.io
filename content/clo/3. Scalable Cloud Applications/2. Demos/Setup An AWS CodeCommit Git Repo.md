+++
title = "Setup An AWS CodeCommit Git Repo"
weight = 7
date = 2024-05-20
draft = false
+++

# Tutorial - Setup An AWS CodeCommit Git Repo

Start using AWS CodeCommit, the Git code repository managed by AWS.

There are three steps to follow in order to setup your first AWS CodeCommit repo:

1. Apply permissions to your IAM user
2. Create AWS git credentials
3. Create a CodeCommit repo

**Tools and applications:**

- AWS Console (Web GUI)
- VS Code
- Terminal
- Git

**Reference**: [Setting Up with AWS CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up.html)

## Configure Git

If you haven´t done so already go ahead and configure your git information

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

Use `git config --list --show-origin` if you want to check the configuration afterwards

## Apply Permissions To Your IAM User

In order to use the AWS CodeCommit repo you have to have the permissions to do so. If you want full git access you can use the managed policy `AWSCodeCommitFullAccess`

**Alt 1: Change User**

1. Go to IAM - Users and select the user you want to give the permissions
2. In the tab "Permission" click "Add permissions" and select "Attach policies directly"
3. Filter on "codecommit" and choose the one that suits yuor needs. Click "Next".
4. Review and click "Add permission"

**Alt 2: Change User groups**

1. Go to IAM - User groups and select the user you want to give the permissions
2. In the tab "Permission" click "Add permissions"
3. Filter on "codecommit" and choose the one that suits yuor needs. Click "Attach policies".

## Create AWS Git Credentials

1. Login to the AWS console
2. Go to "Security credentials" in the account menu
3. Open the tab "AWS CodeCommit credentials"
4. Go to the section "HTTPS Git credentials for AWS CodeCommit" and click "Generate credentials"
5. Note down the credentials or download them to a CSV file. The password will never be available again! If you loose your password you have to re-generate the credentials

## Create A CodeCommit Repo

1. Go to te CodeCommit service. Click "Create Repository"
2. Copy the git clone command under "step 3: Clone the repository"
3. Create a ReadMe.md file by clicking "Create file". Fill in the information in the form and click "Commit changes". (It´s good make sure the repo isn´t empty when you clone it the first time to avoid problems)

**Alt 1: VS Code**

1. Open a directory in VS Code and then open a integrated terminal in that directory
2. Run the git clone command
3. Enter the credentials in the Command Palette of VS Code
4. Change the readme file in the VS Code editor
5. Commit and push the changes to your repo: 
	- use the VS Code GUI to do so
6. Verify in AWS CodeCommit that the changes where uploaded. Choose the "Commits" menu option in the left menu panel

**Alt 2: Terminal**

1. Open a directory in the terminal
2. Run the following git command in order to save the credentials

	```bash
	git config --global credential.helper store
	```
	
2. Run the git clone command
3. Enter the credentials in the terminal
4. Change the readme file (with nano or vim)
5. Commit and push the changes to your repo:

	```bash
	git add .
	git commit -m 'Change ReadMe'
	git push
	```

6. Verify in AWS CodeCommit that the changes where uploaded. Choose the "Commits" menu option in the left menu panel

> The credentials are saved in the file `~/.git-credentials`. On a Mac the credentials will usually be saved in the keychain as well. Search for `git` or `codecommit` in the keychain.app to find the entry.


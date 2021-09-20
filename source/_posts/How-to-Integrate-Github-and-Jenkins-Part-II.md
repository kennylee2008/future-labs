---
title: How to Integrate Github and Jenkins - Part II
date: 2021-08-12 00:53:45
tags:
  - Github
  - Jenkins
  - CICD
  - DevOps
  - SSH
  - Git
  - pipeline
  - Jenkinsfile
---

## Overview

This article is the second part of a series on integrating Jenkins and Github. It focuses on how to use the _git diff_ command in Jenkinsfile to get the differences between the Repository's current Commit and the last Commit, and use the **SSH Agent** to connect to a remote host to execute commands and upload files via _scp_.

<!-- more -->

## How to get the difference between two commits?

The _git diff_ command can be used to get the difference between two commits. For example.

```bash
% git diff --name-only --diff-filter=D <COMMIT_ID_1> <COMMIT_ID_2>
```

We need to know which files were deleted and which files were updated or added between commits. _git diff_ has a very rich set of parameters to use, so here are just a few of the ones we might want to use.

**--name-only**, this parameter is used to output the name of file only.Because we need to know the filename and the path,so we will use this parameter later.

**--diff-filter**, this parameter is used to query for the files we need. For example, _--diff-filter=D_, meaning deleted files. _--diff-filter=A_ means the file that was added. And if lowercase letters are used, it means exclude. For example _--diff-filter=d_, means all files except those deleted files.

We can start by running _git log_ to get a list of the commit history:

```bash
% git log

commit 882866463421d32f9920f7675fa1c3e51caaa1ba (HEAD -> main, origin/main)
Author: kennylee2008 <**@gmail.com>
Date:   Wed Aug 11 16:33:24 2021 +1200

    modify the color of the menus

commit 381d6a602ef1ae5181a72785a91495ebec347e61
Merge: 4c87bb0 13ec5c1
Author: kennylee2008 <**@gmail.com>
Date:   Wed Aug 11 16:32:04 2021 +1200

```

What files were removed from the latest commit compared to the previous commit?

```bash
% git diff --name-only --diff-filter=D 381d6a602ef1ae5181a72785a91495ebec347e61 882866463421d32f9920f7675fa1c3e51caaa1ba
```

What files have been added or updated in the latest commit compared to the previous commit?

```bash
% git diff --name-only --diff-filter=d 381d6a602ef1ae5181a72785a91495ebec347e61 882866463421d32f9920f7675fa1c3e51caaa1ba
```

## How to call Git commands in Jenkinsfile

You can use **Shell Script** to invoke the _git diff_ command and assign the output to the appropriate variables.

```java
script{
    //All modified and added files (none-deleted files)
    COMMIT_CHANGES = sh (
        returnStdout: true,
        script: "git diff --name-only --diff-filter=d $GIT_PREVIOUS_COMMIT $GIT_COMMIT"
    ).trim()

    //All deleted files
    COMMIT_DELETEDS = sh (
        returnStdout: true,
        script: "git diff --name-only --diff-filter=D $GIT_PREVIOUS_COMMIT $GIT_COMMIT"
    ).trim()
}
```

The COMMIT_CHANGES and COMMIT_DELETEDS, obtained in the above example, are both of type String. The $GIT_PREVIOUS_COMMIT/$GIT_COMMIT in the example are the two variables output by the Git plugin.

Since Jenkinsfile can be written using Groovy syntax, which is very similar to Java, programmers familiar with Java can easily master writing Jenkinsfile. For example, the following file is printed out to be deleted.

```java
// Converting strings to arrays
COMMIT_DELETED_SET = COMMIT_DELETEDS.split("\n");

//print out the files to be deleted one by one
for(int i=0; i<COMMIT_DELETED_SET.size(); i++){
	println(COMMIT_DELETED_SET[i]);
}
```

## Configure SSH Credential

You've got the list of difference files. Next, you'll need to connect to the remote host. To do this, we need to configure SSH in Jenkins first.

Generate a public-private key pair:

```bash
% ssh-keygen -t rsa -b 4096 -f "mysshkey"
```

Upload the public key to the target remote deployment host:

```bash
% ssh-copy-id -i mysshkey example@x.x.x.x
```

Configure SSH Credential in Jenkins:

![](2021-08-12T012517.png)
![](2021-08-12T012943.png)
The ID and Description are whatever you want, username is the login name of the remote host, and private key is the file you just generated. Passphrase is the password you entered when you used ssh-keygen to generate the public-private key pair (if you didn't set a password just now, just leave it blank).

## Execute commands on the remote host

You need to make sure that the **SSH Agent** Plugin is installed.

Before use the sshagent, you need to add the remote host public key to ~/.ssh/known_hosts:

```bash
sh """
    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
    [ ! -z \"\$(ssh-keygen -F x.x.x.x)\" ] || ssh-keyscan -t rsa,dsa x.x.x.x >> ~/.ssh/known_hosts
"""
```

It means:

- If the ~/.ssh directory is not exists, create it
- If the public key of the remote host x.x.x.x is not exists in knows_hosts, fetchs it and appends to known_hosts file.

Example:

```bash
sshagent(credentials: ['SSH_Credential_ID_You_Just_Created']) {

	sh """
		ssh example@x.x.x.x <<EOF
		cd /var/www/mywebsite
		rm need/to/be/deleted/file
		exit
		EOF
	"""
}

```

## Transferring files to a remote host

```bash
sshagent(credentials: ['SSH_Credential_ID_You_Just_Created']) {

	sh """
		scp local/file example@x.x.x.x:/remote/file/path
	"""
}
```

## A simple Jenkinsfile sample

```java
pipeline {
    agent any

    stages {
        stage('Deploy') {
            steps {
                script{
                    //All modified and added files (none-deleted files)
                    COMMIT_CHANGES = sh (
                        returnStdout: true,
                        script: "git diff --name-only --diff-filter=d $GIT_PREVIOUS_COMMIT $GIT_COMMIT"
                    ).trim()

                    //All deleted files
                    COMMIT_DELETEDS = sh (
                        returnStdout: true,
                        script: "git diff --name-only --diff-filter=D $GIT_PREVIOUS_COMMIT $GIT_COMMIT"
                    ).trim()
                }
                sh """
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    [ ! -z \"\$(ssh-keygen -F x.x.x.x)\" ] || ssh-keyscan -t rsa,dsa x.x.x.x >> ~/.ssh/known_hosts
                """

                script{

                    //Get the changed files set
                    COMMIT_CHANGE_SET = COMMIT_CHANGES.split("\n");
                    COMMIT_DELETED_SET = COMMIT_DELETEDS.split("\n");

                    sshagent(credentials: ['SSH_With_PrivateKey_to_Deploy_Machine']) {
                        for(int i=0; i<COMMIT_CHANGE_SET.size(); i++){
                            if(!COMMIT_CHANGE_SET[i].startsWith("directory/need/to/be/deploy")
                                && !COMMIT_CHANGE_SET[i].startsWith("another/directory/need/to/be/deploy")
                            ){
                                continue;
                            }
                            COMMIT_CHANGE_DIR = COMMIT_CHANGE_SET[i].substring(0, COMMIT_CHANGE_SET[i].lastIndexOf("/"));

                            //force create the directory and upload the file
                            sh """
                                ssh example@x.x.x.x 'mkdir -p /var/www/websites/example/$COMMIT_CHANGE_DIR'
                                scp ${COMMIT_CHANGE_SET[i]} example@x.x.x.x:/var/www/websites/example/$COMMIT_CHANGE_DIR
                            """

                        }

                        for(int i=0; i<COMMIT_DELETED_SET.size(); i++){
                            if(!COMMIT_DELETED_SET[i].startsWith("directory/need/to/be/deploy")
                                && !COMMIT_DELETED_SET[i].startsWith("another/directory/need/to/be/deploy")
                            ){
                                continue;
                            }
                            //delete the file
                            sh """
                                ssh example@x.x.x.x <<EOF
                                cd /var/www/websites/example
                                rm ${COMMIT_DELETED_SET[i]}
                                exit
                                EOF
                            """

                        }
                    }
                }
            }
        }
    }
}
```

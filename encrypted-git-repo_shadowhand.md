# Transparent Git Encryption

This document has been modified from its [original format][m1], which was
written by Ning Shang (<geek@cerias.net>). It has been updated and reformatted
into a [Markdown][m2] document by [Woody Gilk][m3] and [republished][m4].

## Description

When working with a remote git repository which is hosted on a
third-party storage server, data confidentiality sometimes becomes 
a concern. This article walks you through the procedures of setting 
up git repositories for which your local working directories are as 
normal (un-encrypted) but the committed content is encrypted.

## The Story
I use [git][1] and [Dropbox][2] as a reliable, highly available, cost
saving and distributed version control [solution][3], and have really
found it convenient and effective. One thing that is not addressed in
this solution is data privacy/confidentiality. As Dropbox is a third-party
data storage service with Amazon S3 as its backend data store, a paranoid
user like myself would always be concerned about the Dropbox hosted data
being disclosed to others, accidentally or deliberately. After all, putting
unconditional trust on a third-party provider never seems to be a perfect 
rescue.

User controlled end-to-end encryption solves the problem: before data is
pushed to the remote repository to store, it is encrypted with an encryption
key which is known only to the data owner itself. Management of the 
encryption key(s) and the encryption/decryption processes is always tedious
and easy to get wrong. In the following, we shall demonstrate how to use 
Git with encryption in a way transparent to the end user.

Before we start the demonstration, the following software packages need 
to be installed: git (version 1.7.1 for the demonstration), openssl [4]
(version 0.9.8o 01 Jun 2010 for the demonstration). The operating system
for the demonstration is Linux (Ubuntu 10.10).

The idea is to leverage git's smudge/clean filter, hinted by [this discussion][5],
in which GPG is proposed as the encryption method, we use OpenSSL's symmetric
key cipher as it is a better suitable solution.

## Setup

The procedures are as follows.

1) Before the git repository is created, in your home directory

    $ mkdir .gitencrypt
    $ cd !$

### Create three files

    $ touch clean_filter_openssl smudge_filter_openssl diff_filter_openssl 
    $ chmod 755 *

These files will be the clean/smudge/diff handler/hook for the git 
repository which we are going to work with.

The first file is `clean_filter_openssl`:

    #!/bin/bash

    SALT_FIXED=<your-salt> # 24 or less hex characters
    PASS_FIXED=<your-passphrase>

    openssl enc -base64 -aes-256-ecb -S $SALT_FIXED -k $PASS_FIXED

Here replace `<your-salt>` with a random hexadecimal string and replace 
`<your-passphrase>` with a passphrase you will use as a mater secret for
the symmetric key encryption/decryption. We are using AES-256 ECB mode
as the encryption algorithm, as it turns out a deterministic encryption
works best with git (we will explain later).

The next file is `smudge_filter_openssl`:

    #!/bin/bash

    # No salt is needed for decryption.
    PASS_FIXED=<your-passphrase>

    # If decryption fails, use `cat` instead. 
    # Error messages are redirected to /dev/null.
    openssl enc -d -base64 -aes-256-ecb -k $PASS_FIXED 2> /dev/null || cat

The last file is `diff_filter_openssl`:

    #!/bin/bash

    # No salt is needed for decryption.
    PASS_FIXED=<your-passphrase>

    # Error messages are redirect to /dev/null.
    openssl enc -d -base64 -aes-256-ecb -k $PASS_FIXED -in "$1" 2> /dev/null || cat "$1"

Files in the .gitencrypt directory should be locally kept and never shared
with anyone you do not want to have access to your data, as they contain
your decryption passphrase.

2) Change to the project directory where the git repository is to be
created. Suppose this directory is `~/myproj/`.

    $ git init

Now, create a `.gitattributes` file:

    $ touch .gitattributes

Add the following content to `.gitattributes`:

    * filter=openssl diff=openssl
    [merge]
        renormalize = true

In this file, the `filter` and `diff` attributes are assigned to drivers
named `openssl`, which should be defined in `.git/config` as follows.

    [filter "openssl"]
        smudge = ~/.gitencrypt/smudge_filter_openssl
        clean = ~/.gitencrypt/clean_filter_openssl
    [diff "openssl"]
        textconv = ~/.gitencrypt/diff_filter_openssl

3) Now `git add` relevant files to the staging area. When you do this,
the `clean` filter is applied to files in your working directory, i.e.,
it encrypts the files before they are checked into the staging area. Note 
that as a best practice, `.gitattributes` should not be added. At this time,
you can use `git diff` as usual, as the `diff` filter is properly 
configured to compare the difference of only plain text data (it first 
decrypts if needed).

4) Apply `git commit` to commit the changes to the repository.

5) Now suppose the repository `myproj` is connected to a remote repository
named Dropbox at `file://~/myproj-remote.git`, and you have pushed all
the committed changes to it. Suppose you want to create another git 
repository in directory `~/myproj-1`. First clone the remote repository
without checking out the HEAD.

    $ git clone -o Dropbox -n file://myproj-remote.git myproj-1
    $ cd myproj-1

Now create under `myproj-1` a file .gitattributes with the same content
as shown in Step 2. Then add/append the code snippet in `.git/config` in 
Step 2 to `myproj-1/.git/config`. Then reset the HEAD to check out all of
the files.

    $ git reset --hard HEAD

When the files are checked out, the `smudge` filter is automatically
applied, decrypting the files in the repository and putting the decrypted
files into your working directory. The reason non-deterministic encryption
(what GPG does) does not work very well here is because the same file 
is transformed to a different ciphertext each time it is encrypted, doing
a `git status` always shows the pulled files at modified, even though
a `git diff` shows no difference. Checking in such modified files only
replaces the old ciphertext with a new one which decrypts to the same 
file. If you work in two different local repositories synced to the same
remote, the push/pull process will never end even if nothing is changed
in your working directories. Using AES ECB mode with a fixed salt, although
not semantically secure, resolves this problem while providing reasonable
confidentiality.

From now on, you can work in the local repositories, push to or pull from 
the remote repository as usual, without noticing the encryption/decryption 
in the background.

I think this is cool.

[1]: http://git-scm.com/ "Git: the fast version control system"
[2]: http://www.dropbox.com/ "Dropbox"
[3]: http://syncom.wordpress.com/2010/05/06/%E7%89%88%E6%8E%A7/ "Version control with Git and Dropbox"
[4]: http://www.openssl.org/ "OpenSSL: Cryptography and SSL/TLS Toolkit"
[5]: http://git.661346.n2.nabble.com/Transparently-encrypt-repository-contents-with-GPG-td2470145.html "Web discussion: Transparently encrypt repository contents with GPG"

[m1]: http://syncom.appspot.com/papers/git_encryption.txt
[m2]: http://daringfireball.net/projects/markdown/syntax
[m3]: http://shadowhand.me/
[m4]: https://gist.github.com/873637


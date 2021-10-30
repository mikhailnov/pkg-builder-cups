# pkg-builder-cups

Build RPM packages from git automatically using [CUPS](https://openprinting.github.io/cups/) as a task server.

## What it will do

The goal of this work-in-progress project is to implement the following:

* source code is stored in a Git repository, e.g. in Gitea
* RPM spec file is stored in the root of the source code tree (later support of deb and other package formats may be added)
* when a commit is pushed into the git repo, a special script is called via a [webhook](https://docs.gitea.io/en-us/webhooks)
* rpmbuild is run via [mock](https://github.com/rpm-software-management/mock) or somehow else
* hash of current git commit is passed as a macro to rpmbuild (`rpmbuild --define 'pkg_git_head XXX'`) and can be used inside the spec (`%{pkg_git_head`), e.g. in the `Release` tag
* package is GPG and IMA signed
* package is published into the repository
* older builds of that package are deleted from the repo
* repository metadata is updated via `createrepo_c`
* build logs are copied to a special place and stored there

## Why CUPS?

CUPS is a lightweight task server which can actually be used not only for printing (upstream intends to drop support of custom backends and work with IPP only, but it will not be done in the nearest feature and probably an IPP to cupsbackend proxy will be implemented).

Other options like Jenkins will consume far too many resources (Jenkins is Java).

## Notes about architecture

### Database

We need a database to implement the following:

* connect RPM packages with build tasks to be able to find where a package appeared from
* delete left packages from previous builds of the same project and the same platform (delete older versions etc.)

Platform is a target distribution, e.g. rosa2021.1, el8, fc36, etc.

Our database implementation must meet the following requirements:

* avoid running a heavy database engine like MySQL or PostgreSQL
* keep history of changes
* be scalable, but we do not need to be ready neither for a really high load or a database cluster

Chosen implementation of the database:

* a git repository with csv-alike or other plain text files inside it
* each change is git-commited and git-pushed
* a web view is available via Gitea/Cgit-web/etc

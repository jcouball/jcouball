# Continuous Delivery for Ruby Gems

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Introduction](#introduction)
* [The original inspiration](#the-original-inspiration)
* [Additional changes I made](#additional-changes-i-made)
    * [Change tag format to "v1.0.0"](#change-tag-format-to-v100)
    * [Only create one tag per release](#only-create-one-tag-per-release)
    * [Use sentence case for commits listed in the CHANGELOG](#use-sentence-case-for-commits-listed-in-the-changelog)
* [Conclusion](#conclusion)
* [Additional considerations](#additional-considerations)
* [Continuous Delivery Implementation Runbook](#continuous-delivery-implementation-runbook)
    * [1. Add `.github/workflows/release.yml` to the project](#1-add-githubworkflowsreleaseyml-to-the-project)
    * [2. Add `.release-please-manifest.json` to the project](#2-add-release-please-manifestjson-to-the-project)
    * [3. Add `release-please-config.json` to the project](#3-add-release-please-configjson-to-the-project)
    * [4. Customize the `release` task in the project's `Rakefile`](#4-customize-the-release-task-in-the-projects-rakefile)
    * [5. Add the `AUTO_RELEASE_TOKEN` secret for the project in GitHub](#5-add-the-auto_release_token-secret-for-the-project-in-github)
    * [6. Create a PR with the changes above and merge it](#6-create-a-pr-with-the-changes-above-and-merge-it)
    * [7. Add a trusted publisher for the gem in RubyGems.org](#7-add-a-trusted-publisher-for-the-gem-in-rubygemsorg)
    * [8. Merge the release PR](#8-merge-the-release-pr)

## Introduction

I have been slowly inching toward implementing Continuous Delivery for my open source
gems for a long time now. What finally got me over the line was a blog post by my new
friend [Jonathan Gnagy](https://therubyist.org/about-the-author/) titled [Moving a
Ruby Gem's CI to GitHub
Actions](https://therubyist.org/2025/02/19/moving-a-ruby-gem-ci-to-github-actions/).

## The original inspiration

I was excited because Jonathan's post actually describes *How to implement Continuous
Delivery for Ruby gem projects* (I think that Jonathan buried the lede for this
article). The post uses two GitHub actions to achieve this result: the
[googleapis/release-please-action](https://github.com/googleapis/release-please-action)
and the [rubygems/release-gem](https://github.com/rubygems/release-gem) action.

I followed the instructions from Jonathan's blog (with a couple tweaks) for 13
different open source gems that I maintain. You can read Jonathan's post to see what
he recommended. See my [Continuous Delivery Implementation
Runbook](#continuous-delivery-implementation-runbook) (below) for how I implemented
these changes in my projects.

## Additional changes I made

The implementation was pretty smooth sailing with one exception: following Jonathan's
instructions resulted in two tags being created for each release where only one is
desired. You can see this in [Jonathan's metatron
project](https://github.com/jgnagy/metatron/tags).

### Change tag format to "v1.0.0"

The Release Please action creates a release tag and converts it to a GitHub release
complete with a description containing a list of changes.

By default, this action creates a release tag including the component name (aka gem
name). For the metatron project, the release tag would be "metatron/v0.11.1".

To align with my existing release process, I DO NOT want this tag to contain the gem
name. From the metatron example, I would want the release tag to be "v0.11.1".

This is easy enough to accomplish by adding the following to the Release Please
configuration file:

```json
"include-component-in-tag": false
```

### Only create one tag per release

Unfortunately, the `rubygems/release-gem` action ALSO tries to create a tag in this
format. This is why the metatron project is currently creating **two tags** for each
release.

Since the `rubygems/release-gem` action uses the `rake release` command under the
hood, it tries to create the same tag ("v0.11.1" for this example). Unfortunately,
this fails because the tag already exists having been created earlier by the Release
Please action. It would be nice if the release-gem action allowed you to specify a
different rake task for this action but it isn't really configurable at all. I am
planning on submitting a PR to add that.

As a workaround, I have redefined the rake `release` task to not create and push the
tag to GitHub. I replaced the `release` task with a task that just calls
`release:rubygem_push` by including this in my project's `Rakefile`:

```Ruby
require 'bundler'
require 'bundler/gem_tasks'

# Make it so that calling `rake release` just calls `rake release:rubygems_push` to
# avoid creating and pushing a new tag.

Rake::Task['release'].clear
desc 'Customized release task to avoid creating a new tag'
task release: 'release:rubygem_push'
```

I'll remove that customization if I can get my change to the `rubygems/release-gem`
accepted.

### Use sentence case for commits listed in the CHANGELOG

The last change I made was to add a plugin to sentence case the commit messages when
listing them in the change log. This change is not necessary, but it's my preference.
Add this section after the packages section:

```json
"plugins": [
  {
    "type": "sentence-case"
  }
],
```

## Conclusion

Now my release process is fully automated from commit-to-production (well, you have
to merge the release PR).

What I like about the Release Please action is that it will update the release PR as
you merge other PRs to master. This means that you don't have to have a release per
PR. Release Please will stack up all the changes into a single release PR even
updating the target release version number if you later push a feature or a breaking
change.

I strongly recommend that you also implement this workflow for your own gems.

## Additional considerations

I strongly recommend that your repository include a workflow to enforce that all
commit messages conform to [the Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
standard.

This is crucial because release-please uses these commit types (like fix:, feat:,
feat!: or BREAKING CHANGE:) to automatically determine the correct semantic version
bump (patch, minor, major) and to generate accurate CHANGELOG entries.

Stay tuned for a blog post about enforcing conventional commits for your project.

## Continuous Delivery Implementation Runbook

These are the steps I took to implement continuous delivery in my RubyGem projects.

### 1. Add `.github/workflows/release.yml` to the project

Replace `<gem-name>` with the name of your gem:

```yaml
---
name: Release Gem
description: |
This workflow creates a new release on GitHub and publishes the gem to
RubyGems.org.

The workflow uses the `googleapis/release-please-action` to handle the
release creation process and the `rubygems/release-gem` action to publish
the gem to rubygems.org

on:
push:
    branches: ["main"]

workflow_dispatch:

jobs:
release:
    runs-on: ubuntu-latest

    environment:
    name: RubyGems
    url: https://rubygems.org/gems/<gem-name>

    permissions:
    contents: write
    pull-requests: write
    id-token: write

    steps:
    - name: Checkout project
        uses: actions/checkout@v4

    - name: Create release
        uses: googleapis/release-please-action@v4
        id: release
        with:
        token: ${{ secrets.AUTO_RELEASE_TOKEN }}
        config-file: release-please-config.json
        manifest-file: .release-please-manifest.json

    - name: Setup ruby
        uses: ruby/setup-ruby@v1
        if: ${{ steps.release.outputs.release_created }}
        with:
        bundler-cache: true
        ruby-version: ruby

    - name: Push to RubyGems.org
        uses: rubygems/release-gem@v1
        if: ${{ steps.release.outputs.release_created }}
```

### 2. Add `.release-please-manifest.json` to the project

Replace `<last-release-version>` with the last released version of your gem (for
example, "0.1.0").

```json
{
".": "<last-release-version>"
}
```

If you have never released a version of this gem, just leave the json object empty:

```json
{}
```

### 3. Add `release-please-config.json` to the project

Make the following replacements:

* Replace `<commit-sha>` with the SHA of the last release (or leave off
`bootstrap-sha` if this gem has never been released).
* Replace `<gem-name>` with the name of your gem
* Replace `<path>` with the path to the gem's version file (for example,
`lib/metatron/version.rb`)

```json
{
    "bootstrap-sha": "<commit-sha>",
    "packages": {
        ".": {
        "release-type": "ruby",
        "package-name": "<gem-name>",
        "changelog-path": "CHANGELOG.md",
        "version-file": "<path>",
        "bump-minor-pre-major": true,
        "bump-patch-for-minor-pre-major": true,
        "draft": false,
        "prerelease": false,
        "include-component-in-tag": false
        }
    },
    "plugins": [
        {
        "type": "sentence-case"
        }
    ],
    "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json"
}
```

### 4. Customize the `release` task in the project's `Rakefile`

```Ruby
require 'bundler'
require 'bundler/gem_tasks'

# Make it so that calling `rake release` just calls `rake release:rubygems_push` to
# avoid creating and pushing a new tag.

Rake::Task['release'].clear
desc 'Customized release task to avoid creating a new tag'
task release: 'release:rubygem_push'
```

### 5. Add the `AUTO_RELEASE_TOKEN` secret for the project in GitHub

The release-please-action requires a GitHub Personal Access token in order to do its
work which includes updating the changelog, creating tags and creating pull requests.

The release workflow accesses this token in the `AUTO_RELEASE_TOKEN` secret.

* Create a PAT. Either create a classic token with repo access or a fine-grained
  token with the following permissions:
    * **Contents**: Read and Write
    * **Metadata**: Read
    * **Pull Requests**: Read and Write
* Add the secret by navigating to the project's GitHub page and then selecting
    Settings -> Secrets and variables -> Actions -> Repository secrets -> New
    repository secret
* Use `AUTO_RELEASE_TOKEN` for the secret name and the token you created as the
    secret value.

### 6. Create a PR with the changes above and merge it

Commit the changes above, create a PR, and merge it once the CI build is completed

* Commit the changes you made above (on a branch)
* Prefix the commit with 'fix:' so that a new release is created
* Create a PR
* Wait for the CI build to succeed
* Merge the PR

Verification:

* Verify that the release workflow runs and creates a new release PR

### 7. Add a trusted publisher for the gem in RubyGems.org

In order to publish the gem to rubygems.org, rubygems/release-gem action requires
the gem have trusted publishing configured on RubyGems.org.

* Login to Rubygems.org
* Go to the page on RubyGems for the gem being published
* In the "Links" section, click "Trusted publishers" and enter your password if
    prompted
* Click the "Create" button and enter the publisher information
* Trusted publisher type: Github Actions
* **Repository owner**: `<github user or organization name>`
* **Repository name**: `<github repository name>`
* **Workflow filename**: release.yml
* **Environment**: RubyGems (this must match what is in the `release.yml`)

### 8. Merge the release PR

Verification:

* Verify that the release workflow successfully pushes a new version of the gem to
    rubygems.org

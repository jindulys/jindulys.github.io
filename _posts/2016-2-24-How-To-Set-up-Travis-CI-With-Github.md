---
layout: post
title: How to Setup Travis CI with Github
---

Continuous Integration is the process of automatically building and running tests whenever a change is committed.

Travis sets up "hooks" with Github to automatically run tests at specified times. 

This repo is used to record the steps to setup Travis CI with one of my repo [GithubPilot](https://github.com/jindulys/GithubPilot), my Repo has already had some **Test Cases**.

# Step1

Go to [Travis-Ci](https://travis-ci.org) sign in with your github account

<img src="http://jindulys.github.io/images/TravisLogin.png" width="310px" height="190px" style="margin: 0 auto; display: block;"/>

Click your account and then find the project you want to turn on, just turn it on.
<img src="http://jindulys.github.io/images/TurnOnGithubPilot.png" width="360px" height="70px" style="margin: 0 auto; display: block;"/>

# Step2

Prepare your `.travis.yml` file.
 
          
     language: objective-cs
     osx_image: xcode7.2
     env:
       global:
       - LC_CTYPE=en_US.UTF-8
       - LANG=en_US.UTF-8
     matrix:
       - SCHEME="GithubPilot" SDK=iphonesimulator9.2

     before_install:
       - gem install cocoapods --no-rdoc --no-ri --no-document --quiet
       - gem install xcpretty --no-rdoc --no-ri --no-document --quiet

     script:
       - set -o pipefail
       - xcodebuild -version
       - xctool -workspace GithubPilot.xcworkspace -scheme "$SCHEME" -sdk "$SDK" ONLY_ACTIVE_ARCH=NO CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO

# Step3

At this time, probably you will hit an Error on Travis, read the log from Travis, you might know what you should do. Share the `Scheme`.

<img src="http://jindulys.github.io/images/TravisManageScheme.png" width="560px" height="260px" style="margin: 0 auto; display: block;"/>

<img src="http://jindulys.github.io/images/TravisShareScheme.png" width="560px" height="360px" style="margin: 0 auto; display: block;"/>

Pushed your changes.

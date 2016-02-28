---
layout: post
title: 信号量semaphore的一个小故事
comments: true
---

Dispatch semaphores是苹果提供的一种控制资源访问的机制。它提供了一种类似开关的机制，根据当前信号量的数量来决定阻塞一个任务，或者让任务继续进行。具体的说明可以参考苹果这个[文档](https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW24)。

本文讲述一个semaphore的实际使用案例，可以让大家通过例子看看怎么使用信号量来控制进度，与其它异步通信相比有什么不同。

# 栗子🌰的背景

我最近在写[Github API Swift 实现](https://github.com/jindulys/GithubPilot)时, 遇到一个有趣的API － List Stargazers。 这个API接口是用来调取一个代码仓库的所有star它的人。 如果这个仓库star的人很多，它的返回会以pagination的形式返回。

     GET /repos/:owner/:repo/stargazers

**API 返回**：

    Status: 200 OK
    Link: <https://api.github.com/resource?page=2>; rel="next",
          <https://api.github.com/resource?page=5>; rel="last"
    X-RateLimit-Limit: 5000
    X-RateLimit-Remaining: 4999

    [
      {
        "login": "octocat",
        "id": 1,
        "avatar_url": "https://github.com/images/error/octocat_happy.gif",
        "gravatar_id": "",
        "url": "https://api.github.com/users/octocat",
        "html_url": "https://github.com/octocat",
        "followers_url": "https://api.github.com/users/octocat/followers",
        "following_url": "https://api.github.com/users/octocat/following{/other_user}",
        "gists_url": "https://api.github.com/users/octocat/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/octocat/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/octocat/subscriptions",
        "organizations_url": "https://api.github.com/users/octocat/orgs",
        "repos_url": "https://api.github.com/users/octocat/repos",
        "events_url": "https://api.github.com/users/octocat/events{/privacy}",
        "received_events_url": "https://api.github.com/users/octocat/received_events",
        "type": "User",
        "site_admin": false
      }
    ] 


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
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


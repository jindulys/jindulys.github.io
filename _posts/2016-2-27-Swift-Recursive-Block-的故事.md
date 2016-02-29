---
layout: post
title: Swift Recursive Block的一个小故事
comments: true
---

本文讲述一个recursive block的实际使用案例，可以让大家通过例子看看为什么使用recursive block，以及如何用swift式的方式来实现。

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

我写这个API的时候，就按照API需要信息写了这个Routes函数，传入仓库主人用户名，传入仓库名，然后把page的信息以params的形式传入，函数会返回，这页stargazer数组（默认30个），或者error信息，还有nextpage的信息，代码如下：


```swift
/**
Users that stars a repo belongs to a user.
     
- parameter repo: repo name
- parameter name: owner
- parameter page: when repo has a lot of stargazers, pagination will be applied.
     
- returns: an RpcRequest, whose response result contains `[GithubUser]`, if pagination is applicable, response result contains `nextpage`.
*/
public func getStargazersFor(repo repo: String, owner: String, page: String = "1", defaultResponseQueue: dispatch_queue_t? = nil) -> RpcCustomResponseRequest<UserArraySerializer, StringSerializer, String> {
    precondition((repo.characters.count != 0 && owner.characters.count != 0), "Invalid Input")
        
    // Custom Response Handler to extract next `page`.
    let httpResponseHandler:((NSHTTPURLResponse?)->String?)? = { (response: NSHTTPURLResponse?) in
        if let nonNilResponse = response,
                link = (nonNilResponse.allHeaderFields["Link"] as? String),
                sinceRange = link.rangeOfString("page=") {
                    var retVal = ""
                    var checkIndex = sinceRange.endIndex
                    
                    while checkIndex != link.endIndex {
                        let character = link.characters[checkIndex]
                        let characterInt = character.zeroCharacterBasedunicodeScalarCodePoint()
                        if characterInt>=0 && characterInt<=9 {
                            retVal += String(character)
                        } else {
                            break
                        }
                        checkIndex = checkIndex.successor()
                    }
                    return retVal
        }
        return nil
    }
        
    return RpcCustomResponseRequest(client: self.client, host: "api", route: "/repos/\(owner)/\(repo)/stargazers", method: .GET, params: ["page":page], postParams: nil, postData: nil,customResponseHandler:httpResponseHandler, defaultResponseQueue: defaultResponseQueue, responseSerializer: UserArraySerializer(), errorSerializer: StringSerializer())
}
```

# 上栗子🌰

那现在需求来了，如果一个用户不想用这种一页一页请求的方式，他想要一次性取得这个仓库的所有stargazer，咋整呢？作为一个API Wrapper的提供者，你应该隐藏复杂的处理逻辑，尽量提供简单的接口给用户方便他调用。

## 最原始的解决思路

最原始的解决思路，虽然原始但是确实正确的形式，大概的样子应该是这么调用的。第一次调用 -> 得到返回信息 -> 提取需要的信息，内部发送第二次调用 -> 得到返回信息 -> ... -> 得到最后一页信息，通知程序得到所有结果，调用Callback。

直接点的写这种代码

```swift
client.stars.getStargazersFor(repo: "Yep", owner: "CatchChat", page: "1").response({ (nextPage, result, error) -> Void in
    if let users = result {
        print(users.count)
    }
                
    if let page = nextPage {
        print("Next page is:\(page)")
        client.stars.getStargazersFor(repo: "Yep", owner: "CatchChat", page: page).response({ (nextPage, result, error) -> Void in
        if let users = result {
            print(users.count)
        }
                        
        if let page = nextPage {
            print("Next page is:\(page)")
            client.stars.getStargazersFor(repo: "Yep", owner: "CatchChat", page: page).response({ (nextPage, result, error) -> Void in
                /*
                    . 真这么写你就输了
                    .
                    .
                */
            })
        }
        })
    }
})
```
这种思路的确是对的，我们就是想要这种recursive迭代的方式去请求下一个，因为下一个请求依赖前一个的返回。只是这么写显然不可行，重复的代码被一直嵌套，而且你不知道什么时候到头。

## 怎么改写能更加**Swifty** -- Recursive Block here to rescue

Swift里Block本身就是一等公民，我们可以用它做变量，封装一些必要的处理步骤在里面。我们这个场景就特别适合使用block变量的方式，因为一方面我们需要调用`getStargazersFor(repo:,owner:,page:)`来发起网络请求，另一方面，再网络请求返回后我们希望根据情况用得到的返回值来再一次调用同样的网络请求，或者结束整个block的执行。大概的模式是这样的：block调用 -> 网络请求 -> 网络返回 -> block调用 -> ... -> 网络返回 -> 满足终止条件，结束调用。

2014年WWDC大会上有个session叫做**"Advanced Swift"**其中提到一个技术叫做**Memoization**, 这个技术运用了swift functional特性来增强recursive call的效率, 通过在高阶函数内部增加额外的处理逻辑，这里增加了dictionary的cache功能，来达到提升效率的目的。而这个高阶特性的实现其事就依赖于recursive block。WWDC的代码片段如下，注意其中的result变量的使用：

<img src="http://jindulys.github.io/images/correctWWDC.png" width="610px" height="290px" style="margin: 0 auto; display: block;"/>

这里我们也写一个recursive block来完成上面类型的网络请求。对于Github API分页的结果返回，当你请求最后一页的时候，最后一页的下一页`nextpage`为 _"1"_，代码如下：

```swift
var aggregatedresult:[GithubUser] = []
// 1. 初始化一个block变量，并用dummy block赋值.
var recursiveBlock: (String, String, String) -> () = {(_, _, _) in }

// 2. recursiveBlock的实现，内部调用了自己
recursiveBlock = { repo, owner, page in
    client.stars.getStargazersFor(repo: repo, owner: owner, page: page).response({(nextPage, result, error) -> Void in
        if let users = result {
            print(users.count)
            self.myTestResult.appendContentsOf(users)
        }
        if let vpage = nextPage {
            print("Next page is:\(vpage)")
                if vpage == "1" {
                    print("Finished")
                } else {
                    // 3. 调用了自己，因为 `1`有声明这个变量,compiler就可以infer它的信息，所以能调用.
                    recursiveBlock(repo, owner, vpage)
                }
        }
    })    
}

recursiveBlock("Yep", "CatchChat", "1")
```
使用recursive block的关键点有两个，第一是你需要声明一个变量var，或是用optional的nil赋初始值，或是不用optional用dummy block来直接赋初始值。第二是把含有调用block自己的逻辑写在block实现代码中。这样我们就可以正常的回调了，代码如下：

```swift
/**
    Get all the stargazers belong to a owner's repo.
     
     - note: This request is time consuming if this repo is a quite popular one. but it will run on a private serial queue and will not block main queue.
     
     - parameter repo:              repo's name.
     - parameter owner:             owner's name.
     - parameter complitionHandler: callback that call on main thread.
*/
public func getAllStargazersFor(repo repo: String, owner: String, complitionHandler:([GithubUser]?, String?)-> Void) {
    var recursiveStargazers: (String, String, String) -> Void = {_, _, _ in }
    var retVal: [GithubUser] = []
    recursiveStargazers = {
        repo, owner, page in
        self.getStargazersFor(repo: repo, owner: owner, page: page).response {
            (nextPage, result, error) -> Void in
            guard let users = result, vpage = nextPage else {
                complitionHandler(nil, error?.description ?? "Error,Could not finish this request")
                return
            }

            retVal.appendContentsOf(users)
            if vpage == "1" {
                complitionHandler(retVal, nil)
            } else {
                recursiveStargazers(repo, owner, vpage)
            }
        }
    }
        
    recursiveStargazers(repo, owner, "1")
}
```


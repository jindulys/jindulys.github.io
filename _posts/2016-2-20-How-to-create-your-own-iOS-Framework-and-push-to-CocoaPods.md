---
layout: post
title: How to create your own iOS Framework and push to CocoaPods
---

> Frameworks are a collection of code and resources to encapsulate functionality that is valuable across projects. Frameworks work perfectly with extensions, sharing logic that can be used by both the main application, and the bundled extensions.

Let's escape the step of comparing `static library` with `framework` and jump into how to build a framework.

## Creation of a Framework

* Create a new Xcode project and select **iOS\Framework & Library\Cocoa Touch Framework**

	<img src="http://jindulys.github.io/images/XcodeFramework.png" width="500px" height="390px" style="margin: 0 auto; display: block;"/>

* When you select a location to save this project, it's preferred to save it to some convenience folder, so you could specify a local path when use Pod during your development of this framework.

	```
	pod 'MyPodName', :path=> '~/Path/To/Folder/Containing/Your/Pod'
	```
* Copy or write files that implement your framework's functionalities(Hard work here).

* Create a Git repo with your favourite one, Github usually.

* Create a **Podspec** for your Framework.

	   cd ~/Path/To/Folder/Containing/Your/Pod
	   pod spec create YourFrameworkName
	   open -a Xcode YourFrameworkName.podspec

* Replace everything in this podspec file with your own information

* Push your repo to _`Github`_

* Run _`pod spec lint`_ to validate your specifications

* Submit your Podspec to Trunk with _`pod trunk push NAME.podspec`_

* If your pod is still in progress, you could update your code by following
	   
	    git add .
	    git commit -m "Your description for this changes"
	    git tag 1.*.0
	    git push -u origin master --tags

	    /* Change your podspec as well */
	    pod spec lint
	    pod trunk push NAME.podspec

* After all of this your pod is ready for use, you could add _`pod 'YourFramework','~>1.0.0'`_ to your Podfile, and run _`pod install`_





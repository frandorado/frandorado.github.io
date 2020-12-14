---
layout: post
title:  Coding on iPad
author: frandorado
categories: [cloud]
tags: [coding, ipad, coder, cloud, coding, free, tablet, github, gitlab, codespaces, gitpod]
image: assets/images/posts/2020-05-16/header.png
toc: true
hidden: false
---

Since iOS 13 added the possibility of using keyboard and mouse, the iPad has become a good device in which to do programming. This post pretends to give some options based on Visual Studio Code that allows us coding in our iPads or tablets.

## Coder (Heroku)
Coder is an open source project in [Github](https://github.com/cdr/code-server/blob/master/README.md) that allows running a version of VS Code on a remote server. Heroku provides a free account to deploy this code server. You only need to follow next steps:

1. Create a free account in Heroku and install the client https://devcenter.heroku.com/articles/heroku-cli
2. Login with `heroku login` or `heroku login -i`
3. Create one application with `heroku create` or `heroku create APPLICATION_NAME`  where you could define your own application name. For this example we could assume the name will be **vscode-test**
4. Enter in Heroku -> Select the app -> Settings -> Config vars and add a new variable called `PASSWORD` with the value you want. This value allows you to login in the app when is deployed.
5. Next, login in docker registry
```
heroku container:login
```
6. Now, we are going to download a version of code-server ready to deploy in heroku
```
docker pull doradofran/coder-heroku
```
7. Tag and push it to docker registry in Heroku
```
docker tag doradofran/coder-heroku registry.heroku.com/vscode-test/web
``` 
```
docker push registry.heroku.com/vscode-test/web
```
9. And finally, we are ready to deploy the application 
```
heroku container:release web -a vscode-test
```
8. That’s all. Now visit **vscode-test.herokuapp.com** and enter the password you defined in the step 3

_Pros and cons:_
* (+) You could clone any repo
* (+) There is not time limitation
* (-) Not easy configuration
* (-) The free heroku account doesn’t allow https so when you start the application you will see a warning indicating that some functionalities don’t work correctly.
* (-) The changes you made in the container could lost since the free account uses share instances that are changing depending on the demand. In this case if you pull code from any repo, this wouldn’t be available in a future access to the vscode.

## Gitpod
Other option I found for coding on iPad is Gitpod. This tool works great and its free option is enough for small changes in personal projects. It connects with Github and Gitlab and doesn’t require any complex configuration. You only need:

1. Login with your GitHub/Gitlab account
2. Enter in `gitpod.io/#XXXXXXX` where you have to replace `XXXXXX` for the url of your github or gitlab repository.

_Pros and cons:_
* (+) Easy configuration
* (+) Keep your configuration and tools you installed.
* (+) Possibility to preview the changes
* (-) Only works with public repos in Github or Gitlab
* (-) Limited to 50 hours/month


## Github codespaces
Currently this option is only available in beta access by request. In the future this could be a good option in the case you use github.

## Conclusions
If you want an easy and fast way to coding on ipad Gitpod is the best option in my opinion. Gitpod keep the configuration, installed tools and allows to preview the result of your project in some cases. By other hand, Coder in Heroku disable the limitation of private or custom repositories that Gitpod has and doesn’t have any limitation related to the time of use. Against is more difficult to configure. If you skip this limitations with Heroku maybe use a non-free account could resolve this disadvantages.

## Shorcuts
All the options above are based in VS Code implementation. The ipad’s keyboard doesn’t contains some commonly used keys like ESC or SUPR. If you need to use this functionality then these are the shortcuts:

* ESC -> `cmd + .`
* SUPR -> `control + d`



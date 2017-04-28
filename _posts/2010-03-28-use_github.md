---
layout: post
title: githubを使ってみよう
date: 2010-03-28 15:40:15.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Git
meta:
  _edit_last: '1'
  _wp_old_slug: github%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%a6%e3%81%bf%e3%82%88%e3%81%86
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
私はいろいろ遅れてしまっている気がしますが、ようやくgithubにアカウント作りました。  
そもそも、私はGit使ったことないんです。

では、早速アカウント作りから。  
[github公式サイト](http://github.com/)にアクセスし、「Sign up now!」ボタンを押すとアカウント作成画面が出てきます。  
いくつかのプランが表示されるので、ひとまずFreeを選びました。  
ほかのはディスクスペースが大きかったり、Private Repositoriesを作れたりするようです。  
さらにユーザ名などなどを入れて作成完了です。

アカウントを作成したらCreate a Repositoryを押すとリポジトリを作成できます。  
そうか、ユーザごとにリポジトリが作成できるのか、そうだよね。  
終わると以下のような親切な案内が出るのでこれに従ってリポジトリを初期化します。  
[![]({{ site.baseurl }}/assets/github_createrepo-300x249.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/github_createrepo.png)

push(Subversionのcommitみたいなもん?)するにはssh用のpublic keyが必要なので以下の方法で生成します。
    
    
    
     % ssh-keygen -t rsa -P ""
    

~/.ssh/以下に鍵ペアが作成されますので、id_rsa.pubをAccount Settingから追加の処理をします。

これで見事Gitが使えるようになりました。  
ちなみに私はGitすらインストールできていなかったので、MacPortsでインストールしました。
    
    
    
    % sudo port install git-core
    

MacPortsは楽だなぁ。

ほかにgithubではWikiとかBTSの機能など豊富な機能があるようです。  
これからは積極的に使ってきたいと思います。  
ちなみに私のgithubは[ここ](http://github.com/hirokikana/)です。

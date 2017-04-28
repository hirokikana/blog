---
layout: post
title: Google App Engineを入門してみよう
date: 2010-03-27 18:29:45.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- GoogleAppEngine
- Python
meta:
  _edit_last: '1'
  _wp_old_slug: google-app-engine%e3%82%92%e5%85%a5%e9%96%80%e3%81%97%e3%81%a6%e3%81%bf%e3%82%88%e3%81%86
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
今さらですか?  
でも、ちゃんといろんなものを触ってみるのが大切だと思います。  
というわけでGoogle App Engineを入門してみます。

どうやら入門的なサイトを見てみると、Googleにスペース(アプリケーションのデプロイ先)を作成して、ローカルの開発環境で開発して上げるという流れのようだ。

まず、[このサイト](http://code.google.com/intl/ja/appengine/)で右のスタートガイドの登録というリンクをクリックして、登録。  
Googleのアカウントを入れたらSMSのアドレス、もとい携帯のアドレスを入れろと出た。  
届いたAuthentication Codeを入力するとアプリケーション作成画面になったので、適当なものをぽちぽちと入力。  
ひとまずデプロイ先ができました。  
以降は[Google app engine](appengine.google.com)にアクセスしてこの辺を制御するようです。

では次にSDKをダウンロード。  
もちろん選ぶは「Google App Engine SDK for Python」のMac OS X。  
JavaとPythonだったら私の頭的にはPythonのが断然理解が早いので、Pythonに。  
[![]({{ site.baseurl }}/assets/2010-03-26-1.01.20）-300x214.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/スクリーンショット（2010-03-26-1.01.20）.png)

インストールしたイメージを展開すると「GoogleAppEngineLauncher」というのをアプリケーションに配置します。  
最初に起動すると以下のようなメッセージが出ます。  
[![]({{ site.baseurl }}/assets/GoogleAppEngineLauncher_symlinkalert-300x183.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/GoogleAppEngineLauncher_symlinkalert.png)どうやらシンボリックリンクを張りたいらしいので「OK」を押します。

では早速Hello Worldを。  
GoogleAppEngineLauncherの左したの「＋」ボタンを押します。  
[![]({{ site.baseurl }}/assets/2010-03-26-1.09.54）-e1269681991349-113x300.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/GoogleAppEngineLauncher_window.png)  
するとダイアログが出るのでApplication Nameにさっきのアプリケーション作成画面で入力したアプリケーション名を入れて「Create」を押します。  
あとはPortで指定した番号にブラウザからアクセスすると「Hello World」が表示されます。  
[![]({{ site.baseurl }}/assets/GoogleAppEngineLauncher_createapp-300x146.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/GoogleAppEngineLauncher_createapp.png)はい、以上です。  
簡単過ぎですね。きっと進化したんですね、昔より。

さらにDeployボタンを押せばGoogleアカウントを入れるだけでデプロイ完了。  
すごい簡単なんですねぇ。  
[![]({{ site.baseurl }}/assets/2010-03-26-1.09.54）1-e1269682066190.png)](http://blog.hirokikana.com/wp-content/uploads/2010/03/GoogleAppEngineDeploy.png)

作成したディレクトリには設定ファイルとメインのソースが自動的に生成されてました。  
これだけではアレなんで、ひとまず参考にしたサイトをもとにmain.pyを修正して、Googleのアカウントと連携ちっくなものを。
    
    
    #!/usr/bin/env python
    import cgi
    import datetime
    import wsgiref.handlers
    from google.appengine.ext import db
    from google.appengine.api import users
    from google.appengine.ext import webapp
    
    class MyHandler(webapp.RequestHandler):
      def get(self):
         user = users.get_current_user()
         if user:
           greeting = ('こんにちは, %s! ([ログアウト](%s))' %
                       (user.nickname(), users.create_logout_url("/")))
         else:
           greeting = ('[ログイン ](%s)' %
                       users.create_login_url("/"))
         self.response.out.write("%s" % greeting)
    
    application = webapp.WSGIApplication([
        ('/', MyHandler)
        ], debug=True)
    
    def main():
      wsgiref.handlers.CGIHandler().run(application)
    
    if __name__ == '__main__':
      main()

ログインを押すとGoogleのログイン画面が表示されます。  
ローカル環境だとしょぼいやつですが、デプロイするとちゃんとまともなやつが表示されました。

これだけだとあれなので、次はちゃんと何か動くものを作ってみたいと思います。

---
layout: post
title: SeleniumとPhantomJSを使ったテストをJenkinsで動かす
date: 2014-06-02 01:18:57.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- Jenkins
- PhantomJS
- Selenium
meta:
  _edit_last: '1'
  _thumbnail_id: '512'
  _wp_old_slug: selenium%e3%81%a8phantomjs%e3%82%92%e4%bd%bf%e3%81%a3%e3%81%9f%e3%83%86%e3%82%b9%e3%83%88%e3%82%92jenkins%e3%81%a7%e5%8b%95%e3%81%8b%e3%81%99
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
Webアプリケーションを作るにあたり大切なのはテストとそれを継続的、しかも自動的に行うということであると思います。テストには様々な段階と形によるテストがあると思いますが、今回はSeleniumとPhantomJSを利用したテストをJenkinsによって動作させる方法について紹介します。

## はじめに

Seleniumはブラウザでの操作をコードにしそれを自動的に実行、結果出力に対するテストを行うためのツールです。SeleniumではFirefoxやChromeといった様々なブラウザで同一の動作をさせ、それを比較することもできます。しかし今回想定しているのはブラウザ間の差異を見るのではなく「想定した操作で想定されるレスポンスであるか」「変更により本来影響が無い箇所に影響がないか」や「そもそも表示されているのか」という部分についてチェックを行うことを想定しています。

また今回ブラウザとしてPhantomJSというGUIが必要無く、描画も行わないヘッドレスブラウザを利用します。描画は行いませんがスクリーンショットを撮影することができます。ヘッドレスブラウザを利用することでブラウザを立ち上げる分のリソースを節約し、高速にテストができることを期待しています。

今回検証した環境はMacOSX Mavericksが入ったMacBook Pro Retina 15(Mid 2012)上のIntelliJ 12を利用しました。

## PhantomJSのインストール

まず、PhantomJSを<http://phantomjs.org/download.html>からダウンロードします。Mac用のバイナリが用意されているのでそれをダウンロードし、解凍後任意のディレクトリにバイナリのphantomjsを配置します。今回は~/opt/bin/に配置しました。

[![phantomjs]({{ site.baseurl }}/assets/phantomjs-300x84.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/phantomjs.png)

## Mavenのインストール

テストプロジェクトでは言語はJava、そしてMavenを使って管理をします。Mavenのインストールは今回は一番手軽なhomebrewを利用しました。下記コマンドを実行しmvn --versionでインストールしたMavenのバージョン情報が返ってくればインストールは正常に完了しています。
    
    
    brew install maven

またM2_HOME環境変数にmvn --versionで出力されたMaven homeの項目を追加しておきます。下記コマンドでシェルにログインした際に自動的にM2_HOME環境変数が入るようにしておくとよいでしょう(下記はzshの場合)
    
    
    echo "export M2_HOME=`mvn --version 2>&1|grep "Maven home:"|awk '{print $3}'` " >> ~/.zshrc

## テストプロジェクトの作成

次にIntelliJを起動し「Create New Project」からNew Projectウィザードを表示します。ウィザード左側からMaven Moduleを選択、Project Nameに任意の名前(今回はSeleniumTestとしました)を入力し「Next」を押します。

![createnewproject]({{ site.baseurl }}/assets/createnewproject-300x262.png)

次の画面でGroupIdにパッケージ名(よくあるcom.hogehoge的なもの)を入力し「Finish」を押すとプロジェクトが作成されます。今回GroupIdにはcom.hirokikana.seleniumtestと入力しました。

![createnewproject2]({{ site.baseurl }}/assets/createnewproject2-300x262.png)これでプロジェクトの作成が完了しました。

## Seleniumで実行するテストの追加

まずプロジェクトのMaven設定ファイルであるpom.xmlを開きSeleniumをJavaから利用するためのドライバ(http://docs.seleniumhq.org/download/maven.jsp)とJUnit(http://maven.apache.org/surefire/maven-surefire-plugin/examples/junit.html)とPhantomJSを利用するためのGhostDriver(https://github.com/detro/ghostdriver)を依存関係に追加します。
    
    
    <dependencies>
     <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.8.1</version>
      <scope>test</scope>
     </dependency>
     <dependency>
      <groupId>org.seleniumhq.selenium</groupId>
      <artifactId>selenium-java</artifactId>
      <version>2.42.1</version>
     </dependency>
     <dependency>
      <groupId>com.github.detro.ghostdriver</groupId>
      <artifactId>phantomjsdriver</artifactId>
      <version>1.1.0</version>
     </dependency>
    </dependencies>
    

また画面上部にMaven Projectでimport が必要である旨のダイアログが出るので、自動的にimportを行うように「Enable Auto-import」をクリックして消します。

[![enableautoimport]({{ site.baseurl }}/assets/enableautoimport.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/enableautoimport.png) 次にプロジェクト左のsrc→test→javaを右クリックしNew→PackageをクリックしJavaの新しいパッケージを作成します。今回はcom.hirokikana.seleniumtest.googletestという名前にしました。次に作成されたパッケージを右クリックし、New→Java Classをクリックし新しいクラスを作成します。今回はTopPageTestという名称にしました。

[![newjavaclass]({{ site.baseurl }}/assets/newjavaclass-300x67.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/newjavaclass.png)

新しく作成されたクラスを下記のように記述します。
    
```java    
package com.hirokikana.seleniumtest.googletest;
import org.junit.Test;
import junit.framework.Assert;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.phantomjs.PhantomJSDriver;
import org.openqa.selenium.phantomjs.PhantomJSDriverService;
import org.openqa.selenium.remote.DesiredCapabilities;

public class TopPageTest {
    @Test
    public void topTitleTest() {
        DesiredCapabilities caps = new DesiredCapabilities();
        caps.setCapability(
            PhantomJSDriverService.PHANTOMJS_EXECUTABLE_PATH_PROPERTY,
            "/Users/hiroki/opt/bin/phantomjs"
        );
        WebDriver driver = new PhantomJSDriver(caps);
        driver.navigate().to("http://www.google.co.jp");

        Assert.assertTrue("title should start Google",
        driver.getTitle().startsWith("Google"));

        driver.close();
    }
}
```

上記のコードはGoogleのトップページを表示し、タイトルがGoogleで始まっているかどうかというのをテストしています。他にも遷移させたり、フォームに文字を入力したりすることができますが、今回は説明しません。

[![runtoppagetest]({{ site.baseurl }}/assets/runtoppagetest-300x273.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/runtoppagetest.png)

これでクラス名を右クリックし「Run 'topTitleTest()'」をクリックします。すると画面下に実行ウィンドウが表示され、実行されている様子がわかります。左側のAll Tests Passedと表示されればすべてのテストは正常に終了したということになります。

[![alltestpassed]({{ site.baseurl }}/assets/alltestpassed-300x113.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/alltestpassed.png)

## Jenkinsでの実行

次にこのプロジェクトをJenkinsで実行します。今回はこのマシンにJenkinsのwarファイルをダウンロードし、下記コマンドで立ち上げてる前提としてJenkinsにプロジェクトを追加します。
    
    
    $ sudo java -jar jenkins.war

Jenkinsから「新規ジョブの作成」をクリックし「Maven2/3プロジェクトのビルド」のジョブを新規作成します。ジョブ名は任意のものでかまいません(今回はtestにしました)

[![jobsetup]({{ site.baseurl }}/assets/jobsetup-300x141.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/jobsetup.png)

プロジェクトの設定画面にある「ビルド」のルートPOMにIntelliJで作成したプロジェクトのpom.xmlをフルパスで指定し、OKを押します。

[![pomsetting]({{ site.baseurl }}/assets/pomsetting-300x89.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/pomsetting.png)

これでプロジェクトの作成は完了です。「ビルド実行」を押すと自動的にビルドが開始され「成功」と表示されると思います。

[![buildsuccess]({{ site.baseurl }}/assets/buildsuccess-300x160.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/buildsuccess.png)

これだけでビルドのたびにJenkinsに結果が保存されテストの履歴を常に確認することができるようになりました。

[![testhistory]({{ site.baseurl }}/assets/testhistory-300x138.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/testhistory.png)

このままだと手動で実行しないとビルドされませんが、設定から「定期的に実行」に実行したい時間をcronで指定するときと同じ形式で指定することで定期的に実行することができます。また、SubversionやGitなどを利用している場合はそれらでソースに変更があった場合にビルドするといったことも可能です。

[![buildtrigger]({{ site.baseurl }}/assets/buildtrigger-300x85.png)](http://blog.hirokikana.com/wp-content/uploads/2014/06/buildtrigger.png)

## まとめ

今回はJavaを使ったSelenium + PhantomJSのテストをJenkinsで動かすところまでを紹介しました。SeleniumとPhantomJSという組み合わせにより、GUIを利用せずに省リソースかつ高速にテストを行うことができるのは大きな強みであると思います。これを常時動作しているサーバーにデプロイしたり、Webアプリケーションのコードの更新とhookさせたりすることでテストとしていきてくると思います。また、今日は紹介しませんでしたが、テストコードもより複雑な動きをさせたり、スクリーンショットをビルドのたびに保存することもできるので次回以降に紹介しようと思います。

## 参考サイト

  * [How to create a simple Selenium WebDriver test using Java and IntelliJ](https://www.youtube.com/watch?v=nqaY0UgRcFQ)
  * [GhostDriverでWebアプリケーションのテストを高速化する - CODESCRIBBLE](http://hutyao.hatenablog.com/entry/ghostdriver)



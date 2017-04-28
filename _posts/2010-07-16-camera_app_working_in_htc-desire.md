---
layout: post
title: HTC Desireで動作するカメラアプリをつくる
date: 2010-07-16 22:40:34.000000000 +09:00
type: post
published: true
status: publish
categories:
- dev
tags:
- android
- desire
meta:
  _edit_last: '1'
  _wp_old_slug: htc-desire%e3%81%a7%e5%8b%95%e4%bd%9c%e3%81%99%e3%82%8b%e3%82%ab%e3%83%a1%e3%83%a9%e3%82%a2%e3%83%97%e3%83%aa%e3%82%92%e3%81%a4%e3%81%8f%e3%82%8b
author:
  login: admin
  email: hiroki@hirokikana.com
  display_name: hiroki.kana
  first_name: Hiroki
  last_name: Takayasu
---
最近の携帯はカメラが当たり前のようについていて、カメラを使うことでアプリの幅も大きく広がります。  
そのカメラを使ったアプリを作成する基礎の基礎となるアプリを作ってみたいと思います。  
今回はとりあえず、カメラでプレビューだけを行うアプリを作成します。  
単純なアプリですが、Xperiaで動いてもDesireで動かないことがあり、意外とはまりました。

まずは、プロジェクトを作成します。
    
    
    
    iMac% android create project -p hello_camera -n HelloCamera -a MainActivity -k com.hirokikana.android.hellocamera -t 9
    Created project directory: hello_camera
    Created directory /Users/hiroki/android/hello_camera/src/com/hirokikana/android/hellocamera
    Added file hello_camera/src/com/hirokikana/android/hellocamera/MainActivity.java
    Created directory /Users/hiroki/android/hello_camera/res
    Created directory /Users/hiroki/android/hello_camera/bin
    Created directory /Users/hiroki/android/hello_camera/libs
    Created directory /Users/hiroki/android/hello_camera/res/values
    Added file hello_camera/res/values/strings.xml
    Created directory /Users/hiroki/android/hello_camera/res/layout
    Added file hello_camera/res/layout/main.xml
    Added file hello_camera/AndroidManifest.xml
    Added file hello_camera/build.xml
    iMac% 
    

次にメインのActivityをMainActivity.javaに実装します。
    
    
    
    package com.hirokikana.android.hellocamera;
    
    import android.app.Activity;
    import android.os.Bundle;
    import android.view.Window;
    import android.view.WindowManager;
    import android.view.WindowManager.LayoutParams;
    
    public class MainActivity extends Activity
    {
        @Override
        public void onCreate(Bundle savedInstanceState)
        {
            super.onCreate(savedInstanceState);
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);　...(1)
            requestWindowFeature(Window.FEATURE_NO_TITLE);　...(2)
            setContentView(new CameraView(this));　...(3)
        }
    }
    

(1)では画面をフルスクリーンにし、(2)でタイトルバーを非表示にしています。  
(3)でViewとしてCameraViewクラスのインスタンスを指定しています。

CameraViewクラスはCameraView.javaに実装します。
    
    
    
    package com.hirokikana.android.hellocamera;
    
    import android.view.SurfaceHolder;
    import android.view.SurfaceView;
    import android.content.Context;
    import android.hardware.Camera.Parameters;
    import android.hardware.Camera;
    import android.hardware.Camera.Size;
    import java.util.List;
    import android.graphics.PixelFormat;
    
    public class CameraView extends SurfaceView implements SurfaceHolder.Callback
    {
        private SurfaceHolder holder; //holder??
        private Camera camera; //カメラ
        private Size previewSize;
    
        // コンストラクタ
        public CameraView(Context context) {
            super(context);
    
            //SurfaceHolderの生成
            holder = getHolder();
            holder.addCallback(this);
    
            //PushBuffer
            holder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
        }
    
        // Surface生成イベント
        public void surfaceCreated(SurfaceHolder holder) {
            // Initialize Camera
            try {
                camera = Camera.open();　　...(1)
                camera.setPreviewDisplay(holder);
            } catch (Exception e) {
                // TODO
            }
            Camera.Parameters param = camera.getParameters();
            List supportedSizes = param.getSupportedPreviewSizes();　...(2)
            previewSize = supportedSizes.get(0); 　...(3)
        }
    
        // Surface変更イベント
        public void surfaceChanged(SurfaceHolder holder, int format, int w, int h) {
            camera.stopPreview();
            Camera.Parameters param = camera.getParameters();
            param.setPreviewSize(previewSize.width, previewSize.height);　...(4)
            param.setPreviewFormat(PixelFormat.JPEG);
            camera.setParameters(param);
            camera.startPreview();
        }
    
        // Surface解放イベント
        public void surfaceDestroyed(SurfaceHolder holder) {
            camera.setPreviewCallback(null);
            camera.stopPreview();
            camera.release();
            camera = null;
        }
    }
    

CameraViewクラスはSurfaceViewの子クラスでSurfaceHolderのCallbackを実装します。  
(1)ではSurfaceの生成時にCameraをopen(生成)します。

(2)でgetSupportedPreviewSizes()でサポートしているPreviewのサイズを取得します。  
getSupportedPreviewSizesで取得した配列で一番目のものは、サポートしているサイズの一番大きいサイズを取得しています。(3)

このあたり、実は一番今回のポイントでした。  
Android 1.6だと(4)のようにPreviewSizeを指定する際に特に何を指定(surfaceChangedの引数)しても問題なく動作していました。  
Android 2.0以降だと、ここをきちんとgetSupportedPreviewSizes()で取って来たサイズでないと起動しませんでした。  
今回書いたコードだと、Android 2.0以上でないと動作しませんが、対策はあるようです。  
この辺は下記の参考サイトに書いてありました。

  * [Androidアプリサービス開発者ブログ:Androidアプリ開発者向け：「HTC Desire」における注意点](http://android.asai24.com/archives/51501707.html)
  * [NexusOneでAPIDemos/CameraPreviewが落ちる件 | Techfirm Android Lab](http://labs.techfirm.co.jp/android/cho/1647)



こういう機種やカーネルのバージョンによる違いはカメラ以外にもたくさんありそうな気がします。

さらにカメラを使用するにはPermissionの設定が必要です。  
AndroidManifest.xmlに下記を追記します。  
追記する場所はの直前です。
    
    
    
        <uses-permission android:name="android.permission.CAMERA" />
        <uses-feature android:name="android.hardware.camera" />
        <uses-feature android:name="android.hardware.camera.autofocus" />
        <uses-feature android:name="android.hardware.camera.flash" />
    

ここまで終わったら実機もしくはエミュレータを接続したうえで、プロジェクトのディレクトリでant installをするとビルドとインストールが行われるので、インストールされたアプリを起動します。  
アプリを起動すると、カメラのプレビューが動作します。

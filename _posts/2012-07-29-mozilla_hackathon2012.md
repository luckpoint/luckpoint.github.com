---
layout: post
title: "Mozilla Hackathon 2012"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## 内容

Mozila Hackathon 2012で調べた事と作ったアドオンについて。

https://etherpad.mozilla.org/mozillahackathon2012-extdev

## 作ったもの 

![MyAddons.png](/assets/Mozilla_Hackathon2012/MyAddons.png)

### 1. 選択した文字列で検索できるようにコンテキストメニューを追加

* https://github.com/luckpoint/luckpoint.github.com/blob/master/assets/Mozilla_Hackathon2012/search_contextmenu_addon.xpi

![SearchContextMenu.png](/assets/Mozilla_Hackathon2012/SearchContextMenu.png)

### 2. 音声検索できるメニューを追加

* https://github.com/luckpoint/luckpoint.github.com/blob/master/assets/Mozilla_Hackathon2012/fennec-17.0a1.en-US.android-arm.apk
* https://github.com/luckpoint/luckpoint.github.com/blob/master/assets/Mozilla_Hackathon2012/recognize_speech_addon.xpi

2-1. メニューからSearch(by voice)を選択

![SearchbyVoice.png](/assets/Mozilla_Hackathon2012/SearchbyVoice.png)

2-2. 音声認識のダイアログが表示されるので、検索したい言葉をしゃべる。

![RecognizeSpeech.png](/assets/Mozilla_Hackathon2012/RecognizeSpeech.png)

## 調べた事

### 1. mobile.jsにいろいろな設定情報がある。

mobile/android/app/mobile.js

    // True if you always want dump() to work
    //
    // On Android, you also need to do the following for the output
    // to show up in logcat:
    //
    // $ adb shell stop
    // $ adb shell setprop log.redirect-stdio true
    // $ adb shell start
    pref("browser.dom.window.dump.enabled", true);

とか。

### 2. Androidでlogcatにログを表示

log関数

    Cu.import("resource://gre/modules/Services.jsm");
    function log(s) {
      Services.console.logStringMessage("MyAddon: " + s);
    }

ターミナル上で、

    adb logcat | grep GeckoConsole

でconsoleのログを表示できる。GeckoAppのタグにも有用なログが流れるため、もう少し範囲を広くして、

    adb logcat | grep Gecko

にしておくのもあり。


### 3. エラーコンソール

    chrome://global/content/console.xul

を開く


### 4. リスタートの方法

    let appStartup = Cc["@mozilla.org/toolkit/app-startup;1"].getService(Ci.nsIAppStartup);
    appStartup.quit(Ci.nsIAppStartup.eRestart | Ci.nsIAppStartup.eAttemptQuit);


### 5. アドオン側とアプリ側とのメッセージのやりとり

mobile/andrid/base/GeckoApp.java

    org.mozilla.gecko.GeckoApp
    public void handleMessage(String event, JSONObject message) {

アドオンからメッセージをJava側にメッセージを投げると、
ここでハンドリングされるよう。

    function getBridge() {
      return Cc["@mozilla.org/android/bridge;1"].getService(Ci.nsIAndroidBridge);
    }

    function sendMessageToJava(aMessage) {
      return getBridge().handleGeckoMessage(JSON.stringify(aMessage));
    }

    sendMessageToJava({
      gecko: {
        type: "Share:Image",
        url: src,
        mime: type,
      }
    });


### 6. Firefox Android版のビルド
* hg pull; hg updateで最新を取得。

* autoconf 2.13のインストール
  ** client.mk:263: *** Could not find autoconf 2.13. のエラー
  ** https://gist.github.com/1010716 をベースにhomeblewでいれた。

* webmのためにYASMのインストール https://developer.mozilla.org/en/YASM

* Compile Error

>> nsNPAPIPluginInstance.cpp:138:14: error: cannot convert 'std::nullptr_t' to 'mozilla::gl::SharedTextureHandle {aka unsigned int}' in return

とりあえず、nunullを使用せずに0に置き換える。

* Link Error
https://bugzilla.mozilla.org/show_bug.cgi?id=776489 と同様のエラーが発生 
>> jsutil.cpp:57: error: undefined reference to 'deflateInit_'
bugzillaにあった関係しそうな 
ac_add_options --with-system-zlib 
をmozconfigに入れてもdistcleanしてもエラーが解消できず。 
--> zlib系はコメントアウトでとりあえず対応。

### 7. 音声認識の呼び出し

Recognize:Speechというメッセージで音声認識を呼び出す

bootstrap.js:

    function getBridge() {
      return Cc["@mozilla.org/android/bridge;1"].getService(Ci.nsIAndroidBridge);
    }
    function sendMessageToJava(aMessage) {
      return getBridge().handleGeckoMessage(JSON.stringify(aMessage));
    }
    sendMessageToJava({                                                                                
       gecko: {                                                                                        
         type: "Recognize:Speech"                                                                      
       }
    });

GeckoApp#initialize

    GeckoAppShell.registerGeckoEventListener("Recognize:Speech", GeckoApp.mAppContext);

GeckoApp#handleMessage:

     // Added For Handling of RecognizerIntent.
     else if (event.equals("Recognize:Speech")) {
         final Intent intent = new Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
         final int requestCode = GeckoAppShell.sActivityHelper.makeRequestCodeForRecognizeSpeech();
         startActivityForResult(intent, requestCode);
    }

ActivityHandlerHelper:

     private final ActivityResultHandler mRecognizeSppechResultHandler = new ActivityResultHandler() {
         @Override
         public void onActivityResult(int resultCode, Intent data) {
             if (resultCode == Activity.RESULT_OK) {
                 final ArrayList<String> results = data.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS);
                 Log.e(LOGTAG, "mRecognizeSppechResultHandler2: " + results.size());
                 if (results.size() > 0) {
                     GeckoApp.mAppContext.loadRequest(results.get(0), AwesomeBar.Target.CURRENT_TAB, "Google", false);
                 }
             }
         }
     };


## 今回の修正

    diff -r 08428deb1e89 dom/plugins/base/nsNPAPIPluginInstance.cpp
    --- a/dom/plugins/base/nsNPAPIPluginInstance.cpp    Thu Jul 26 12:36:15 2012 -0700
    +++ b/dom/plugins/base/nsNPAPIPluginInstance.cpp    Sun Jul 29 13:19:43 2012 +0900
    @@ -96,7 +96,7 @@
       ~SharedPluginTexture()
       {
         // This will be destroyed in the compositor (as it normally is)
    -    mCurrentHandle = nsnull;
    +    mCurrentHandle = 0;//nsnull;
       }
     
       TextureInfo Lock()
    @@ -130,12 +130,12 @@
           return mCurrentHandle;
     
         if (!EnsureGLContext())
    -      return nsnull;
    +      return 0;//nsnull;
     
         mNeedNewImage = false;
     
         if (mTextureInfo.mWidth == 0 || mTextureInfo.mHeight == 0)
    -      return nsnull;
    +      return 0;//nsnull;
     
         mCurrentHandle = sPluginContext->CreateSharedHandle(TextureImage::ThreadShared, (void*)mTextureInfo.mTexture, GLContext::TextureID);
     
    @@ -1022,7 +1022,7 @@
       } else if (mContentSurface) {
         EnsureGLContext();
         return sPluginContext->CreateSharedHandle(TextureImage::ThreadShared, mContentSurface, GLContext::SurfaceTexture);
    -  } else return nsnull;
    +  } else return 0;//nsnull;
     }
     
     void* nsNPAPIPluginInstance::AcquireVideoWindow()
    diff -r 08428deb1e89 js/src/jsutil.cpp
    --- a/js/src/jsutil.cpp Thu Jul 26 12:36:15 2012 -0700
    +++ b/js/src/jsutil.cpp Sun Jul 29 13:19:43 2012 +0900
    @@ -25,18 +25,18 @@
     
     #include "zlib.h"
     
    -using namespace js;
    +//using namespace js;
     
     static void *
     zlib_alloc(void *cx, uInt items, uInt size)
     {
    -    return OffTheBooks::malloc_(items * size);
    +    return js::OffTheBooks::malloc_(items * size);
     }
     
     static void
     zlib_free(void *cx, void *addr)
     {
    -    Foreground::free_(addr);
    +    js::Foreground::free_(addr);
     }
     
     
    @@ -54,16 +54,16 @@
         zs.avail_in = inplen;
         zs.next_out = out;
         zs.avail_out = inplen;
    -    int ret = deflateInit(&zs, Z_BEST_SPEED);
    -    if (ret != Z_OK) {
    -        JS_ASSERT(ret == Z_MEM_ERROR);
    -        return false;
    -    }
    -    ret = deflate(&zs, Z_FINISH);
    -    DebugOnly<int> ret2 = deflateEnd(&zs);
    -    JS_ASSERT(ret2 == Z_OK);
    -    if (ret != Z_STREAM_END)
    -        return false;
    +    //int ret = deflateInit(&zs, Z_BEST_SPEED);
    +    //if (ret != Z_OK) {
    +    //    JS_ASSERT(ret == Z_MEM_ERROR);
    +    //    return false;
    +    //}
    +    //ret = deflate(&zs, Z_FINISH);
    +    //DebugOnly<int> ret2 = deflateEnd(&zs);
    +    //JS_ASSERT(ret2 == Z_OK);
    +    //if (ret != Z_STREAM_END)
    +    //    return false;
         *outlen = inplen - zs.avail_out;
         return true;
     }
    @@ -81,14 +81,14 @@
         zs.next_out = out;
         JS_ASSERT(outlen);
         zs.avail_out = outlen;
    -    int ret = inflateInit(&zs);
    -    if (ret != Z_OK) {
    -      JS_ASSERT(ret == Z_MEM_ERROR);
    -      return false;
    -    }
    -    ret = inflate(&zs, Z_FINISH);
    -    JS_ASSERT(ret == Z_STREAM_END);
    -    ret = inflateEnd(&zs);
    +    //int ret = inflateInit(&zs);
    +    //if (ret != Z_OK) {
    +    //  JS_ASSERT(ret == Z_MEM_ERROR);
    +    //  return false;
    +    //}
    +    //ret = inflate(&zs, Z_FINISH);
    +    //JS_ASSERT(ret == Z_STREAM_END);
    +    //ret = inflateEnd(&zs);
         JS_ASSERT(ret == Z_OK);
         return true;
     }
    diff -r 08428deb1e89 mobile/android/base/ActivityHandlerHelper.java
    --- a/mobile/android/base/ActivityHandlerHelper.java    Thu Jul 26 12:36:15 2012 -0700
    +++ b/mobile/android/base/ActivityHandlerHelper.java    Sun Jul 29 13:19:43 2012 +0900
    @@ -21,6 +21,7 @@
     import android.os.Environment;
     import android.provider.MediaStore;
     import android.util.Log;
    +import android.speech.*;
     
     class ActivityHandlerHelper {
         private static final String LOGTAG = "GeckoActivityHandlerHelper";
    @@ -33,6 +34,20 @@
         private final CameraImageResultHandler mCameraImageResultHandler;
         private final CameraVideoResultHandler mCameraVideoResultHandler;
     
    +    private final ActivityResultHandler mRecognizeSppechResultHandler = new ActivityResultHandler() {
    +        @Override
    +        public void onActivityResult(int resultCode, Intent data) {
    +            Log.e(LOGTAG, "mRecognizeSppechResultHandler: " + resultCode);
    +            if (resultCode == Activity.RESULT_OK) {
    +                final ArrayList<String> results = data.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS);
    +                Log.e(LOGTAG, "mRecognizeSppechResultHandler2: " + results.size());
    +                if (results.size() > 0) {
    +                    GeckoApp.mAppContext.loadRequest(results.get(0), AwesomeBar.Target.CURRENT_TAB, "Google", false);
    +                }
    +            }
    +        }
    +    };
    +
         ActivityHandlerHelper() {
             mFilePickerResult = new SynchronousQueue<String>();
             mActivityResultHandlerMap = new ActivityResultHandlerMap();
    @@ -46,6 +61,10 @@
             return mActivityResultHandlerMap.put(mAwesomebarResultHandler);
         }
     
    +    int makeRequestCodeForRecognizeSpeech() {
    +        return mActivityResultHandlerMap.put(mRecognizeSppechResultHandler);
    +    }
    +
         private int addIntentActivitiesToList(Context context, Intent intent, ArrayList<PromptService.PromptListItem> items, ArrayList<Intent> aIntents) {
             PackageManager pm = context.getPackageManager();
             List<ResolveInfo> lri = pm.queryIntentActivityOptions(GeckoApp.mAppContext.getComponentName(), null, intent, 0);
    diff -r 08428deb1e89 mobile/android/base/GeckoApp.java
    --- a/mobile/android/base/GeckoApp.java Thu Jul 26 12:36:15 2012 -0700
    +++ b/mobile/android/base/GeckoApp.java Sun Jul 29 13:19:43 2012 +0900
    @@ -50,6 +50,7 @@
     import android.provider.*;
     import android.content.pm.*;
     import android.content.pm.PackageManager.*;
    +import android.speech.*;
     import dalvik.system.*;
     
     abstract public class GeckoApp
    @@ -1096,7 +1097,14 @@
                     GeckoAppShell.shareImage(src, type);
                 } else if (event.equals("Sanitize:ClearHistory")) {
                     handleClearHistory();
    +            } 
    +            // Added For Handling of RecognizerIntent.
    +            else if (event.equals("Recognize:Speech")) {
    +                final Intent intent = new Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
    +                final int requestCode = GeckoAppShell.sActivityHelper.makeRequestCodeForRecognizeSpeech();
    +                startActivityForResult(intent, requestCode);
                 }
    +
             } catch (Exception e) {
                 Log.e(LOGTAG, "Exception handling message \"" + event + "\":", e);
             }
    @@ -1727,6 +1735,7 @@
             GeckoAppShell.registerGeckoEventListener("Share:Text", GeckoApp.mAppContext);
             GeckoAppShell.registerGeckoEventListener("Share:Image", GeckoApp.mAppContext);
             GeckoAppShell.registerGeckoEventListener("Sanitize:ClearHistory", GeckoApp.mAppContext);
    +        GeckoAppShell.registerGeckoEventListener("Recognize:Speech", GeckoApp.mAppContext);
     
             if (SmsManager.getInstance() != null) {
               SmsManager.getInstance().start();





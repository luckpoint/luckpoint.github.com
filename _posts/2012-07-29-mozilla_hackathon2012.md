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

## 作ったもの 

![MyAddons.png](/assets/Mozilla_Hackathon2012/MyAddons.png)

### 1. 選択した文字列で検索できるようにコンテキストメニューを追加

![SearchContextMenu.png](/assets/Mozilla_Hackathon2012/SearchContextMenu.png)

### 2. 音声検索できるメニューを追加

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






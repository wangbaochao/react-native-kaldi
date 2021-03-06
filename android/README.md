# React Native Kaldi bridge demo

cd into "android" directory and perform gradle build:
$ gradle build
Make sure that there are no errors 

In first terminal:
- cd into project root directory (ReactNativeKaldi)
- run "npm install"
- May be but try without it first: Run "npx react-native start" 
In the second terminal:
npx react-native run-android --port 9000

====================================================================

If you have any build issues, first delete "build" directory from "android" and build again. Most of the time it fixes all issues.

For "Invariant Violation: "AppName" has not been registered" look at the file app.json and package.json and check same name field both files, maybe them difference.

You might want to build aars and models projects separetly:
cd ReactNativeKaldi/android/aars
gradle build

cd ReactNativeKaldi/android/models
gradle build

You might want to do some clean up for updating sources 
react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --assets-dest android/app/src/main/res/


Here's a sample of Kaldi Event in form of JSON:

[Tue Jun 30 2020 17:22:59.114]  BUNDLE  ./index.js

[Tue Jun 30 2020 17:23:03.238]  LOG      Running "HelloReactWorld" with {"rootTag":1}
[Tue Jun 30 2020 17:23:09.149]  GROUP  ..  running recognize mic
[Tue Jun 30 2020 17:23:46.615]  LOG    ..  event.eventProperty   :   text:
[Tue Jun 30 2020 17:23:46.644]  LOG    ..  event.eventProperty   :   result: {"result" : [  ], "text" : "" }
[Tue Jun 30 2020 17:23:48.997]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own"}
[Tue Jun 30 2020 17:23:49.930]  LOG    ..  event.eventProperty   :   partial: {"partial" : "example"}
[Tue Jun 30 2020 17:23:49.833]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own juice"}
[Tue Jun 30 2020 17:23:50.960]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own juices"}
[Tue Jun 30 2020 17:23:50.321]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own juices forms"}
[Tue Jun 30 2020 17:23:50.637]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own juices forms"}
[Tue Jun 30 2020 17:23:51.260]  LOG    ..  event.eventProperty   :   partial: {"partial" : "his own juices forms"}
[Tue Jun 30 2020 17:23:51.438]  LOG    ..  event.eventProperty   :   text: his own juices forms
[Tue Jun 30 2020 17:23:51.487]  LOG    ..  event.eventProperty   :   result: {"result" : [ {"word": "his", "start" : 37.8, "end" : 38.01, "conf" : 1},
{"word": "own", "start" : 38.01, "end" : 38.22, "conf" : 1},
{"word": "juices", "start" : 38.85, "end" : 39.3, "conf" : 1},
{"word": "forms", "start" : 39.3, "end" : 39.9, "conf" : 1}
 ], "text" : "his own juices forms" }
[Tue Jun 30 2020 17:23:51.836]  LOG    ..  event.eventProperty   :   partial: {"partial" : ""}




## Troubleshooting

If you don't see audio permission dialog, make sure that your AndroidManifest.xml has this line:
<uses-permission android:name="android.permission.RECORD_AUDIO" />

Some good Java code examples:
https://github.com/JoaoCnh/react-native-android-voice/blob/master/src/main/java/com/wmjmc/reactspeech/VoiceModule.java

https://www.programcreek.com/java-api-examples/?code=jordanbyron/react-native-quick-actions/react-native-quick-actions-master/android/src/main/java/com/reactNativeQuickActions/AppShortcutsModule.java

https://reactnative.dev/docs/native-modules-android.html


Design aprroaches and decision

# Bridge

The source code for React Native bridge is located in facebook/react-native repository:
https://github.com/facebook/react-native/tree/master/ReactAndroid/src


## Native Modules to React Native UI Bridge (Java calls JavaScript)

There are different ways to pass information from Native Modules to React Native UI. One of the easiest way is to provide callbacks to the JavaScript function calls of Native Module methods.
Callbacks implement Inversion of Control principle which is good for certain scenarios such as one time native calls.  Native modules can also fulfill a promise (or async/await syntax) that represents asynchronous operations such as method calls similar to callbacks.

But if we deal with cases which more contolled by backend side, it is better use an event driven mechanism.
Native modules can signal events to JavaScript without being invoked directly. By using RCTDeviceEventEmitter class which can be obtained from the ReactContext class we can emit events from Java and pass
as many parameters as we want using WritableMap. Here's a sendEvent method used to trigger events to React Native UI:
```
private static void sendEvent(String eventName,
                            @Nullable WritableMap params) {
    reactContext
        .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
        .emit(eventName, params);
}
```

NativeEventEmitter class is a part of 'react-native' package and is helpful to subscribe to events produced by Native Module. Here's an example of this kind of subscription:

```
const eventEmitter = new NativeEventEmitter(NativeModules.ToastExample);
    this.eventListener = eventEmitter.addListener('EventReminder', (event) => {
       console.log(event.eventProperty);
    });
```
In this example we use addListener method to receive events.

## React Native UI to Native Module Bridge (JavaScript calls Java)

A native module is a Java class that usually extends the ReactContextBaseJavaModule class and implements the functionality required by the JavaScript. We create 
The specifics of React Native Kaldi PoC is that once we initialize Kaldi Speech Recongrizer we deligate control to Native Module that connects Kaldi to microphone directly and sends Kaldi events to JavaScript outside of main UI thread.
For this goal we can simply expose Native Module Java methods annotated using @ReactMethod in our KaldiModule.java which will be invoked by JavaScript (for example on button click events). Here we can also use callbacks or promises to pass result back to JavaScript but as discussed in "Java calls JavaScript" section sending information back with emitted events will use established communication model. 
The example of using @ReactMethod annotated Java function:

    @ReactMethod
    public void init(int permission, Promise promise) {
        try {
            
            Activity currentActivity = getCurrentActivity();
            
            // If something bad happens (no associated activity is below case), we can use promise to trigger error right away.
            if (currentActivity == null) {
                promise.reject(ACTIVITY_DOES_NOT_EXIST);
                return;
            }

            // Here we use events to signal JavaScript that we are ready to start
            setUiState(STATE_START);
            
            ....
            // After all validations and checks are done we execute setup task.  
            new SetupTask(currentActivity).execute();
        } catch (Exception e) {
            promise.reject(new JSApplicationIllegalArgumentException(e.getMessage()));
        }
    }

Note, that at the end of function execution there is a call of execute method of SetupTask that finalize initialization process - this is a separeted thread instance using AsyncTask android class which AsyncTask must be subclassed to be used.
Key methods of SetupTask are (at least doInBackground will override by the subclass):
doInBackground which is invoked on the background thread immediately. This step is used to perform background computation that can take a long time.
onPostExecute(Result) that is invoked on the UI thread after the background computation finishes

AsyncTasks should ideally be used for short operations (a few seconds at the most.) If we need to keep threads running for long periods of time, it is highly recommended to use the various APIs provided by the java.util.concurrent package such as Executor, ThreadPoolExecutor and FutureTask.

The other way to establish React Native UI to Native Module involves using ActivityEventListener so you will be listening onActivityResult. To do this, you must extend BaseActivityEventListener or implement ActivityEventListener. The former is preferred as it is more resilient to API changes. Then, you need to register the listener in the module's constructor.


## Kaldi Speech Recognizer

We use Vosk as an open source speech recogintion toolkit that uses (Kaldi behind the scene)[https://github.com/alphacep/vosk-api]

Vosk installation instructions, examples and turorial and documentation (can be found here)[https://alphacephei.com/vosk/]

SpeechRecognizer is a main class to access recognizer function of KaldiRecognizer. After configuration this class starts a listener thread which records the data and recognizes it using VOSK engine. Recognition events are passed to a client using RecognitionListener.

For (SpeechRecognizer code refer)[https://github.com/alphacep/vosk-api/blob/master/android/src/main/java/org/kaldi/SpeechRecognizer.java]

KaldiRecognizer uses JNI (voskJNI) to access Kaldi C++ library (libkaldi_jni.so build). It provides main methods to operate Kaldi such as new instance, accept wave form, new model and final/partial results.
Another important java class called Assets that provides utility methods to keep asset files to external storage to allow further JNI code access assets from a filesystem.

https://github.com/alphacep/vosk-api/blob/master/android/src/main/java/org/kaldi/Assets.java

In the project, a vosk-android-demo was use as [Kaldi wrapper for Android](https://github.com/alphacep/vosk-android-demo)
It also provides kaldi-andorid aar [kaldi-android-5.2.aar](https://github.com/alphacep/vosk-android-demo/blob/master/aars/kaldi-android-5.2.aar)

Kaldi default settings in Vosk:

- sample rate: 16,000

- buffer size: sample rate * 0.4F

AudioRecord(Andorid) settings: 

 - source: VOICE_CALL 
  
 - channel: CHANNEL_IN_MONO
  
 - audio format: ENCODING_PCM_16BIT
  
 - buffer size: default buffer size * 2
  

## Haskell Demo: Android

### building
- in `demo-android-lib`: `nix build .#aarch64-android:lib:demo`
- in `demo-android-app`:
  - `mkdir -p app/src/main/jniLibs && unzip -o ../demo-android-lib/result/pkg-aarch64-android-libdemo.zip -d app/src/main/jniLibs/arm64-v8a`
  - `./gradlew assemble`

### how does it work
- The `com.example.haskell_demo.DemoApp` Kotlin class and the `Demo.Main` Haskell module's `start` function
  are boilerplate that tie the native library libdemo.so compiled from Haskell into the Kotlin Android app.
- 2 threads, a Haskell spawned event loop (`loop` in module `Main`) and the native Android UI thread,
  are communicating via `MVar`s `uiUpdatePort` (Haskell writes, Android reads) and `uiEventPort`
  (Android writes, Haskell reads).
- Android event handlers require deriving a class and implementing/overriding event handler methods. Event handler delegate to
  `com.example.haskell_demo.Event.onEvent`, see e.g. `com.example.haskell_demo.MainActivity.onCreate`.
  - In the special case of an event handler class of all `void` type methods, e.g. `OnXxxxListener`s
    like `android.view.View.OnClickListener`, the generic `com.example.haskell_demo.VoidInvocationHandler` can be used through a Java dyanmic proxy
    instance. See  `Demo.Android.UI.mkButton`.
- Haskel-driven updates to the Android UI have to run in Android's UI thread:
  - Put dynamic UI content into `uiUpdatePort` and call `notifyUIUpdate`. `com.example.haskell_demo.MainActivity.run` will invoke Haskell
    UI update code in the UI thread.
- The UI consist of a static part drawn at activity creation (layouts, 1 text view initially empty, 1 button)
  and a dynamic part continuously updated (text view contents).
  - `initUI` draws the static initial part and `updateUI` does incremental updates
- Android may destroy and recreate the main activity e.g. when the phone rotates.
  - The Haskell event loops receives activity creation events and stores the current activity in `mainActivityOpt`.
  - The `activityCache` MVar caches the last main activity instance initialized with the static UI part. If the
    activity instance to be updated is different it has been recreated and needs to be reinitialized, otherwise
    only an incremental update needs to be applied.

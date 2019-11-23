# CrashKiOS - Crash reporting for Kotlin/iOS

Thin library that provides symbolicated crash reports for Kotlin code on 
iOS. Currently implemented for Crashlytics and Bugsnag.

## The Problem

Kotlin's design has obviously been influenced by Java. In Java, exceptions
are a normal thing, and further in Kotlin, checked exceptions aren't a thing.
Crash reporters (Crashlytics, Bugsnag, etc) can take the unhandled exceptions
and provide the full stack trace.

On iOS, exceptions exist, but they're very much a special case. When a "crash" happens, 
the app stops, and crash reporting tools take the state of the application threads.
When calling into Kotlin code, if a crash happens in the Kotlin code, the exception 
bubbles back up to the iOS/Kotlin interface, at which point, if not @Throws, the app
is forcibly crashed. You can see the crash info from the local device and from the app store, 
assuming the user reported it, but crash reporting services like Crashlytics and Bugsnag 
only get the stack trace from the iOS/Kotlin interface. Not where the crash actually happened.

## The Solution

 With some stack trace visibility improvements included in Kotlin 1.3.60, we can report 
 "handled errors" to Crashlytics and Bugsnag, and have them record symbolicated crash reports.
 These can be sent explicitly, but more likely, reported as part of the uncaught exception handler.
 
 ## Usage
 
 The crash library itself is minimal, but because of current transitive code rules and how interop 
 works with Objc and Swift, we'll be copying some code manually.
 
 ### Gradle
 
 ```groovy
kotlin {
    sourceSets {
        iosMain {
            dependencies {
                api "co.touchlab:crashkios:0.2"
            }
        } 
    }
}
```
 
 ### Kotlin
 
 In your app, you'll need to include some extra passthrough code. In your iOS Kotlin source, add the 
 following function
 
 ```kotlin
import co.touchlab.crashkios.CrashHandler
import co.touchlab.crashkios.setupCrashHandler

fun crashInit(handler: CrashHandler){
    setupCrashHandler(handler)
}
```

In our sample, we put this in a file called `CrashIntegration.kt`. If it's in a different file, you'll
need to know that for the Swift config.

This will be called from your iOS code to set the default crash handler. There are other options,
but this is generally what you want to do. The name `crashInit` is unimportant, but creating this
method forces the Kotlin compiler to include the necessary crash handling code. In the future, with 
proper transitive dependency settings, this may not be necessary.

### Swift

In Swift, you need to add a `CrashHandler` instance specific to the crash reporting service you'll 
be using. We currently support Crashlytics and Bugsnag. Setup currently involves copy/pasting some
Swift/Objc code rather than using interop bindings in Kotlin. This is more manual, but simpler. We 
may add interop code in the future.

#### Crashlytics

Create the `CrashHandler` class

```swift
class CrashlyticsCrashHandler: CrashkiosCrashHandler {
    override func crashParts(
        addresses: [KotlinLong],
        exceptionType: String,
        message: String) {
        let clsStackTrace = addresses.map {
            CLSStackFrame(address: UInt(truncating: $0))
        }

        Crashlytics.sharedInstance().recordCustomExceptionName(
            exceptionType,
            reason: message,
            frameArray: clsStackTrace
        )
    }
}
```

Note, `CrashkiosCrashHandler` and possibly `KotlinLong` may have different generated names from
the Kotlin compiler output.

Inside `AppDelegate.swift`, or wherever you do app setup, add the following:

```swift
    CrashIntegrationKt.crashInit(handler: CrashlyticsCrashHandler())
```

#### Bugsnag

Bugsnag's setup is a little more complicated. We create an extension of `NSException` which is provided
to the Bugsnag crash reporting library.

In Swift, add the following:

```swift
class CrashNSException: NSException {
    init(callStack:[NSNumber], exceptionType: String, message: String) {
        super.init(name: NSExceptionName(rawValue: exceptionType), reason: message, userInfo: nil)
        self._callStackReturnAddresses = callStack
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    private lazy var _callStackReturnAddresses: [NSNumber] = []
    override var callStackReturnAddresses: [NSNumber] {
        get { return _callStackReturnAddresses }
        set { _callStackReturnAddresses = newValue }
    }
}

class BugsnagCrashHandler: CrashkiosCrashHandler {
    override func crashParts(addresses: [KotlinLong], exceptionType: String, message: String) {
        Bugsnag.notify(CrashNSException(callStack: addresses, exceptionType: exceptionType, message: message))
    }
}
```

Note, `CrashkiosCrashHandler` and possibly `KotlinLong` may have different generated names from
the Kotlin compiler output.

Inside `AppDelegate.swift`, or wherever you do app setup, add the following:

```swift
    CrashIntegrationKt.crashInit(handler: BugsnagCrashHandler())
```

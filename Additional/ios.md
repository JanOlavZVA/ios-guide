

# Application Execution States

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_states.png">

** States: **
* Not running - The app has not been launched or was running but was terminated by the system.
* Inactive - The app is running in the foreground but is currently not receiving events. (It may be executing other code though.) An app usually stays in this state only briefly as it transitions to a different state.
* Active - The app is running in the foreground and is receiving events. This is the normal mode for foreground apps.
* Background - The app is in the background and executing code. Most apps enter this state briefly on their way to being suspended. However, an app that requests extra execution time may remain in this state for a period of time. In addition, an app being launched directly into the background enters this state instead of the inactive state. For information about how to execute code while in the background, see Background Execution.
* Suspended - The app is in the background but is not executing code. The system moves apps to this state automatically and does not notify them before doing so. While suspended, an app remains in memory but does not execute any code. When a low-memory condition occurs, the system may purge suspended apps without notice to make more space for the foreground app.

# Application Launch

## main()
The execution of every C program starts with a function called main(), and since Objective-C is a strict superset of C, the same must be true for an Objective-C program. If you create a new iOS project from one of the default templates, Xcode places this function in a separate file called main.m in the Supporting Files group. Usually, you never have to look at that file but let’s do. This is the entire code of main():

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

The function’s arguments argc and argv contain info about the command-line arguments passed to the executable on launch. We can safely ignore them for this discussion. Let’s have a look at what the function does, which seems to be very litte:

1. It creates an autorelease pool because in every Cocoa app one must exist at all times (otherwise, an autorelease call would fail).
2. It calls a function named UIApplicationMain(). We will take a deeper look at it below.
3. It drains the autorelease pool it just created.
4. It returns the return value of UIApplicationMain() to its caller (which is the shell that launched the executable).

When an (Objective-)C program reaches the end of main(), it ends. So this looks like a very short program indeed. Nevertheless, this is how all iOS apps work, so the secret must be the UIApplicationMain() function. Should it ever return, our program would end immediately.

## UIApplicationMain()

This function instantiates the application object from the principal class and and instantiates the delegate (if any) from the given class and sets the delegate for the application. It also sets up the main event loop, including the application’s run loop, and begins processing events. If the application’s Info.plist file specifies a main nib file to be loaded, by including the NSMainNibFile key and a valid nib file name for the value, this function loads that nib file.

Despite the declared return type, this function never returns.

Let’s take this apart step by step:

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/lifecycle2.png">

1. First, the function creates the main application object (step 3 in the flowchart). If you specify nil as the third argument to UIApplicationMain() (the default), it will create an instance of UIApplication in this step. This is usually what you want. However, if you need to subclass UIApplication (for example, to override its event handling in sendEvent:), you have to pass a string with the name of your subclass to UIApplicationMain().

2. The function then looks at its fourth argument. If it is non-nil, it interprets it as the name of the class for the application delegate, instantiates an object of this class and assigns it as the application object’s delegate. The default for the fourth argument is nil, though, which signifies that the app delegate will be created in the main NIB file.

3. Next, UIApplicationMain() loads and parses your app’s Info.plist (step 4). If it contains a key named “Main nib file base name” (NSMainNibFile), the function will also load the NIB file specified there (step 5).

4. By default, the main NIB file is called MainWindow.nib. It contains at least an object representing the application delegate, connected to the File’s Owner’s delegate outlet (step 6), and a UIWindow object that will be used as the app’s main window, connected to an outlet of the app delegate. If you used a view-controller-based app template, the NIB file will also contain your app’s root view controller and possibly one or more view child controllers.

> Note: It is worth mentioning that this is the only step where the UIKit-based app templates (Window-based, View-based, Navigation-based, Tab-based, etc.) differ significantly from each other. If you started out with a view-based app and later want to introduce a navigation controller, there is no need to start a new project: simply replace the root view controller in the main NIB file and adjust one or two lines of code in the app delegate. I noticed that many newbies to the iOS platform struggle with this problem and assume a huge difference between the different project templates. There isn’t.

5. Now, UIApplicationMain() creates the application’s run loop that is used by the UIApplication instance to process events such as touches or network events (step 7). The run loop is basically an infinite loop that causes UIApplicationMain() to never return.

6. Before the application object processes the first event, it finally sends the well-known application:didFinishLaunchingWithOptions: message to its delegate, giving us the chance to do our own setup (step 8). The least we have to do here is put our main window on the screen by sending it a makeKeyAndVisible message.

### Application Delegate

Most application state transitions are accompanied by a corresponding call to the methods of your app delegate object. These methods are your chance to respond to state changes in an appropriate way. These methods are listed below, along with a summary of how you might use them.

* application:willFinishLaunchingWithOptions: — This method is your app’s first chance to execute code at launch time.
* application:didFinishLaunchingWithOptions: — This method allows you to perform any final initialization before your app is displayed to the user.
* applicationDidBecomeActive: — Lets your app know that it is about to become the foreground app. Use this method for any last minute preparation.
* applicationWillResignActive: — Lets you know that your app is transitioning away from being the foreground app. Use this method to put your app into a quiescent state.
* applicationDidEnterBackground: — Lets you know that your app is now running in the background and may be suspended at any time.
* applicationWillEnterForeground: — Lets you know that your app is moving out of the background and back into the foreground, but that it is not yet active.
* applicationWillTerminate: — Lets you know that your app is being terminated. This method is not called if your app is suspended.

### Main Run Loop

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/main_run_loop.png">

An app’s main run loop processes all user-related events. The UIApplication object sets up the main run loop at launch time and uses it to process events and handle updates to view-based interfaces. As the name suggests, the main run loop executes on the app’s main thread. This behavior ensures that user-related events are processed serially in the order in which they were received.

Figure shows the architecture of the main run loop and how user events result in actions taken by your app. As the user interacts with a device, events related to those interactions are generated by the system and delivered to the app via a special port set up by UIKit. Events are queued internally by the app and dispatched one-by-one to the main run loop for execution. The UIApplication object is the first object to receive the event and make the decision about what needs to be done. A touch event is usually dispatched to the main window object, which in turn dispatches it to the view in which the touch occurred. Other events might take slightly different paths through various app objects.

Many of these event types are delivered using the main run loop of your app, but some are not. Some events are sent to a delegate object or are passed to a block that you provide. For information about how to handle most types of events—including touch, remote control, motion, accelerometer, and gyroscopic events — see [Event Handling Guide for iOS](apple.com).

## Entry Points

Besides application:didFinishLaunchingWithOptions:, there are several more entry points for custom code during the launch sequence (none of which are usually needed):

* Directly in main() before UIApplicationMain() is called.
* The init method of a custom UIApplication subclass.
* The initWithCoder: or awakeFromNib methods of our application delegate if it is created from a NIB file (the default).
* The +initialize methods of our application delegate class or a custom UIApplication subclass. Any class receives an +initialize message before it is sent its first message from within the program.

> Note that this sequence only happens at the actual launch of an app. If the app is already running and simply brought back from the background, none of this occurs.

## Application Termination

Apps must be prepared for termination to happen at any time and should not wait to save user data or perform other critical tasks. System-initiated termination is a normal part of an app’s life cycle. The system usually terminates apps so that it can reclaim memory and make room for other apps being launched by the user, but the system may also terminate apps that are misbehaving or not responding to events in a timely manner.

Suspended apps receive no notification when they are terminated; the system kills the process and reclaims the corresponding memory. If an app is currently running in the background and not suspended, the system calls the applicationWillTerminate: of its app delegate prior to termination. The system does not call this method when the device reboots.

In addition to the system terminating your app, the user can terminate your app explicitly using the multitasking UI. User-initiated termination has the same effect as terminating a suspended app. The app’s process is killed and no notification is sent to the app.


# Handling Application State Transitions

When your app is launched, it moves from the not running state to the active or background state, transitioning briefly through the inactive state. As part of the launch cycle, the system creates a process and main thread for your app and calls your app’s main function on that main thread. The default main function that comes with your Xcode project promptly hands control over to the UIKit framework, which does most of the work in initializing your app and preparing it to run.

> Note: To determine whether your app is launching into the foreground or background, check the applicationState property of the shared UIApplication object in your application:willFinishLaunchingWithOptions: or application:didFinishLaunchingWithOptions: delegate method.

## Interrupted Temporarily
Alert-based interruptions result in a temporary loss of control by your app. Your app continues to run in the foreground, but it does not receive touch events from the system. (It does continue to receive notifications and other types of events, such as accelerometer events, though.).

So in ```applicationWillResignActive:``` you should to save your app state. Pause, terminate or suspend all tasks (some critical network requests can be untouched).
When your app is moved back to the active state, its ```applicationDidBecomeActive:``` method should reverse any of the steps taken in the ```applicationWillResignActive:``` method.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_temporary_interrupt.png">


## Transition To Foreground
**Be Prepared to Process Queued Notifications**
An app in the suspended state must be ready to handle any queued notifications when it returns to a foreground or background execution state. A suspended app does not execute any code and therefore cannot process notifications related to orientation changes, time changes, preferences changes, and many others that would affect the app’s appearance or state. To make sure these changes are not lost, the system queues many relevant notifications and delivers them to the app as soon as it starts executing code again (either in the foreground or background). To prevent your app from becoming overloaded with notifications when it resumes, the system coalesces events and delivers a single notification (of each relevant type) that reflects the net change since your app was suspended.

Queued notifications are delivered on your app’s main run loop and are typically delivered before any touch events or other user input. Most apps should be able to handle these events quickly enough that they would not cause any noticeable lag when resumed.

An app returning to the foreground also receives view-update notifications for any views that were marked dirty since the last update. An app running in the background can still call the setNeedsDisplay or setNeedsDisplayInRect: methods to request an update for its views. However, because the views are not visible, the system coalesces the requests and updates the views only after the app returns to the foreground.

**Notifications delivered to waking apps:**
Event | Notification
------ | ------
An accessory is connected or disconnected. | EAAccessoryDidConnectNotification EAAccessoryDidDisconnectNotification
The device orientation changes. | UIDeviceOrientationDidChangeNotification. In addition to this notification, view controllers update their interface orientations automatically.
There is a significant time change. | UIApplicationSignificantTimeChangeNotification
The battery level or battery state changes. | UIDeviceBatteryLevelDidChangeNotificationUIDeviceBatteryStateDidChangeNotification
The proximity state changes. | UIDeviceProximityStateDidChangeNotification
The status of protected files changes. | UIApplicationProtectedDataWillBecomeUnavailableUIApplicationProtectedDataDidBecomeAvailable
An external display is connected or disconnected. | UIScreenDidConnectNotificationUIScreenDidDisconnectNotification
The screen mode of a display changes. | UIScreenModeDidChangeNotification
Preferences that your app exposes through the Settings app changed. | NSUserDefaultsDidChangeNotification
The current language or locale settings changed. | NSCurrentLocaleDidChangeNotification
The status of the user’s iCloud account changed. | NSUbiquityIdentityDidChangeNotification

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_transition_to_foreground.png">

## Transition To Background

When moving from foreground to background execution, use the applicationDidEnterBackground: method of your app delegate to do the following:

1. Prepare to have your app’s picture taken. 
Shortly after an app delegate’s ```applicationDidEnterBackground:``` method returns, the system takes a snapshot of the app’s windows. Similarly, when an app is woken up to perform background tasks, the system may take a new snapshot to reflect any relevant changes. For example, when an app is woken to process downloaded items, the system takes a new snapshot so that can reflect any changes caused by the incorporation of the items. The system uses these snapshot images in the multitasking UI to show the state of your app.

If you make changes to your views upon entering the background, you can call the ```snapshotViewAfterScreenUpdates:``` method of your main view to force those changes to be rendered. Calling the ```setNeedsDisplay``` method on a view is ineffective for snapshots because the snapshot is taken before the next drawing cycle, thus preventing any changes from being rendered. Calling the ```snapshotViewAfterScreenUpdates:``` method with a value of YES forces an immediate update to the underlying buffers that the snapshot machinery uses.

2. Save any relevant app state information. Prior to entering the background, your app should already have saved all critical user data. Use the transition to the background to save any last minute changes to your app’s state.

3. Free up memory as needed. Release any cached data that you do not need and do any simple cleanup that might reduce your app’s memory footprint. Apps with large memory footprints are the first to be terminated by the system, so release image resources, data caches, and any other objects that you no longer need. For more information, see Reduce Your Memory Footprint.

Your app delegate’s ```applicationDidEnterBackground:``` method has approximately 5 seconds to finish any tasks and return. In practice, this method should return as quickly as possible. If the method does not return before time runs out, your app is killed and purged from memory. If you still need more time to perform tasks, call the ```beginBackgroundTaskWithExpirationHandler:``` method to request background execution time and then start any long-running tasks in a secondary thread. Regardless of whether you start any background tasks, the ```applicationDidEnterBackground:``` method must still exit within 5 seconds.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/appclose.png">

## State Preservation and Restoration

State preservation and restoration is not an automatic feature and apps must opt-in to use it. Apps indicate their support for the feature by implementing the following methods in their app delegate:

```application:shouldSaveApplicationState:```
```application:shouldRestoreApplicationState:```

UIKit preserves only those objects that have an assigned restoration identifier. A restoration identifier is a string that identifies the view or view controller to UIKit and your app. The value of this string is significant only to your code but the presence of this string tells UIKit that it needs to preserve the tagged object. During the preservation process, UIKit walks your app’s view controller hierarchy and preserves all objects that have a restoration identifier. If a view controller does not have a restoration identifier, that view controller and all of its views and child view controllers are not preserved.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_state_restoration.png">

For each view controller you choose to preserve, you also need to decide how you want to restore it later. UIKit offers two ways to recreate objects. You can let your app delegate recreate it or you can assign a restoration class to the view controller and let that class recreate it. A restoration class implements the UIViewControllerRestoration protocol and is responsible for finding or creating a designated object at restore time. Here are some tips for when to use each one:

* If the view controller is always loaded from your app’s main storyboard file at launch time, do not assign a restoration class. Instead, let your app delegate find the object or take advantage of UIKit’s support for implicitly finding restored objects.
* For view controllers that are not loaded from your main storyboard file at launch time, assign a restoration class. The simplest option is to make each view controller its own restoration class.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_preservation_flow.png">
<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/app_restoration_flow.png">

When the restoration identifier of a view controller is nil, that view controller and any child view controllers it manages are not preserved automatically. For example, in Figure 5-5, because a navigation controller did not have a restoration identifier, it and all of its child view controllers and views are omitted from the preserved data.
Even if you decide not to preserve view controllers, that does not mean all of those view controllers disappear from the view hierarchy altogether. At launch time, your app might still create the view controllers as part of its default setup. For example, if any view controllers are loaded automatically from your app’s storyboard file, they would still appear, albeit in their default configuration.

**Restoring Your View Controllers at Launch Time**

During the restoration process, UIKit asks your app to create (or locate) the view controller objects that comprise your preserved user interface. UIKit adheres to the following process when trying to locate view controllers:

1. If the view controller had a restoration class, UIKit asks that class to provide the view controller. UIKit calls the viewControllerWithRestorationIdentifierPath:coder: method of the associated restoration class to retrieve the view controller. If that method returns nil, it is assumed that the app does not want to recreate the view controller and UIKit stops looking for it.
2. If the view controller did not have a restoration class, UIKit asks the app delegate to provide the view controller. UIKit calls the application:viewControllerWithRestorationIdentifierPath:coder: method of your app delegate to look for view controllers without a restoration class. If that method returns nil, UIKit tries to find the view controller implicitly.
3. If a view controller with the correct restoration path already exists, UIKit uses that object. If your app creates view controllers at launch time (either programmatically or by loading them from a resource file) and assigns restoration identifiers to them, UIKit finds them implicitly through their restoration paths.
4. If the view controller was originally loaded from a storyboard file, UIKit uses the saved storyboard information to locate and create it. UIKit saves information about a view controller’s storyboard inside the restoration archive. At restore time, it uses that information to locate the same storyboard file and instantiate the corresponding view controller if the view controller was not found by any other means.

It is worth noting that if you specify a restoration class for a view controller, UIKit does not try to find your view controller implicitly. If the viewControllerWithRestorationIdentifierPath:coder: method of your restoration class returns nil, UIKit stops trying to locate your view controller. This gives you control over whether you really want to create the view controller. If you do not specify a restoration class, UIKit does everything it can to find the view controller for you, creating it as necessary from your app’s storyboard files.

## Background Execution

### Executing Finite-Length Tasks

Apps moving to the background are expected to put themselves into a quiescent state as quickly as possible so that they can be suspended by the system. If your app is in the middle of a task and needs a little extra time to complete that task, it can call the ```beginBackgroundTaskWithName:expirationHandler:``` or ```beginBackgroundTaskWithExpirationHandler:``` method of the UIApplication object to request some additional execution time. Calling either of these methods delays the suspension of your app temporarily, giving it a little extra time to finish its work. Upon completion of that work, your app must call the ```endBackgroundTask:``` method to let the system know that it is finished and can be suspended.

```swift
var backgroundUpdateTask: UIBackgroundTaskIdentifier!

func applicationDidEnterBackground(_ application: UIApplication) {
	doBackgroundTask()
}

func doBackgroundTask() {
	beginBackgroundDownload()

	let queue = DispatchQueue.global(qos: .background)
	queue.async {
		sleep(2) // simulate operations
		self.endBackgroundDownload()
	}
}

func beginBackgroundDownload() {
	backgroundUpdateTask = UIApplication.shared.beginBackgroundTask(withName: "DownloadImages", expirationHandler: {
		self.endBackgroundDownload()
	})
}

func endBackgroundDownload() {
	UIApplication.shared.endBackgroundTask(backgroundUpdateTask)
	backgroundUpdateTask = UIBackgroundTaskInvalid
}
```

### Downloading Content in the Background

When downloading files, apps should use an NSURLSession object to start the downloads so that the system can take control of the download process in case the app is suspended or terminated. When you configure an NSURLSession object for background transfers, the system manages those transfers in a separate process and reports status back to your app in the usual way. If your app is terminated while transfers are ongoing, the system continues the transfers in the background and launches your app (as appropriate) when the transfers finish or when one or more tasks need your app’s attention.

To support background transfers, you must configure your NSURLSession object appropriately. To configure the session, you must first create a NSURLSessionConfiguration object and set several properties to appropriate values. You then pass that configuration object to the appropriate initialization method of NSURLSession when creating your session.

The process for creating a configuration object that supports background downloads is as follows:

1. Create the configuration object using the backgroundSessionConfigurationWithIdentifier: method of NSURLSessionConfiguration.
2. Set the value of the configuration object’s sessionSendsLaunchEvents property to YES.
3. if your app starts transfers while it is in the foreground, it is recommend that you also set the discretionary property of the configuration object to YES.
4. Configure any other properties of the configuration object as appropriate.
5. Use the configuration object to create your NSURLSession object.
6. Handle app delegate’s ```application:handleEventsForBackgroundURLSession:completionHandler:```

**Example:**
```swift
    // Network Manager
    
    let downloadUrl = "http://www.download.me/plz.pdf"
    let sessionConfig = URLSessionConfiguration.background(withIdentifier: "com.apple.unique.background.id")
    
    var urlSession: URLSession {
        sessionConfig.isDiscretionary = true // manage slow speed network
        return URLSession(configuration: sessionConfig, delegate: self, delegateQueue: .main)
    }
    
    func downloadDocument() {
        if let url = URL(string: downloadUrl) {
            let task = urlSession.downloadTask(with: url)
            task.resume()
        }
    }
    
    // URLSessionDelegate
    
    func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didFinishDownloadingTo location: URL) {
        do {
            let dir = FileManager.default.urls(for: FileManager.SearchPathDirectory.documentDirectory, in: FileManager.SearchPathDomainMask.userDomainMask).last
            let path = dir?.appendingPathComponent("Documnet.pdf")
            try FileManager.default.copyItem(at: location, to: path!)
        } catch let error {
            print(error)
        }
    }
    
    func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession) {
        let sessionIdentifier = urlSession.configuration.identifier
        if let sessionId = sessionIdentifier, let app = UIApplication.shared.delegate as? AppDelegate, let handler = app.completionHandlers.removeValue(forKey: sessionID) {
            handler()
        }
    }
```
```swift    
    // Application Delegate

    var completionHandlers = [String : () -> Void]()
    
    func application(_ application: UIApplication, handleEventsForBackgroundURLSession identifier: String, completionHandler: @escaping () -> Void) {
        completionHandlers[identifier] = completionHandler
    }
    //
```

### Long Running Tasks

* Apps that play audible content to the user while in the background, such as a music player app
* Apps that record audio content while in the background
* Apps that keep users informed of their location at all times, such as a navigation app
* Apps that support Voice over Internet Protocol (VoIP)
* Apps that need to download and process new content regularly
* Apps that receive regular updates from external accessories

To declare the background modes your app supports from the Capabilities tab of your project settings. Enabling the Background Modes option adds the UIBackgroundModes key to your app’s Info.plist file.

* Audio and AirPlay - The app plays audible content to the user or records audio while in the background. (This content includes streaming audio or video content using AirPlay.)
* Location updates - The app keeps users informed of their location, even while it is running in the background.
* Voice over IP - The app provides the ability for the user to make phone calls using an Internet connection.
* Newsstand downloads - The app is a Newsstand app that downloads and processes magazine or newspaper content in the background.
* External accessory communication - The app works with a hardware accessory that needs to deliver updates on a regular schedule through the External Accessory framework.
* Uses Bluetooth LE accessories - The app works with a Bluetooth accessory that needs to deliver updates on a regular schedule through the Core Bluetooth framework.
* Acts as a Bluetooth LE accessory - The app supports Bluetooth communication in peripheral mode through the Core Bluetooth framework.
* Background fetch - The app regularly downloads and processes small amounts of content from the network.
* Remote notifications - The app wants to start downloading content when a push notification arrives. Use this notification to minimize the delay in showing content related to the push notification.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/backround_fetch.png">

### Background Fetch

```swift
// AppDelegate
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey:Any]?) -> Bool {
	UIApplication.shared.setMinimumBackgroundFetchInterval(2)
}

func application(_ application: UIApplication, performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    NetworkServiceLayer.fetchNewBatch(completion: completionHandler)
}

// NetworkServiceLayer
static func fetchNewBatch(completion: (UIBackgroundFetchResult) -> Void) {
    
    let downloadUrl = "http://www.download.me/plz.pdf"
    let url = URL(string: downloadUrl)!
    let urlSession = URLSession.shared
    
    let downloadTask = urlSession.downloadTask(with: url) { (url: URL?, response: URLResponse?, error: Error?) in
        
        // Handle response
        // Call completion with the results of your response
        completion(UIBackgroundFetchResult.newData)
    }
    
    downloadTask.resume()
}
```

* If fetch interval is UIApplicationBackgroundFetchIntervalMinimum then then system will decide when to call performFetchWithCompletionHandler method.
* On launch app starts in background mode, calling application:performFetchWithCompletionHandler:
* After you have done your works you must call completionHandler with status parameter (UIBackgroundFetchResult)
* You only have 30 seconds, after this period app will be terminated by the system
* Besides application:performFetchWithCompletionHandler:, application:didFinishLaunchingWithOptions: is also called, so you should handle any UI code or unrelated to background fetch for the economy of CPU time. To do so you need to check application.applicationState, which will be UIApplicationState.background

> Note: if in the next 30 seconds, while app is working on background, user launches it via UI, application:didFinishLaunchingWithOptions: wont be called second time. So if you need to handle skipped UI code you can do it in applicationWillEnterForeground:

### Background Fetch with push notification

**Notification Payloads**

A notification is sent to the user as a JSON dictionary that itself contains a dictionary with the key aps. In this second dictionary you specify the payload keys and values.

The most common are:

* The notification message shown to the user. This can be a simple string, or a dictionary with keys like title, body, etc.
* The sound the device will play when it receives the notification. It can be a custom sound of the app, or a system sound.
* The number the app will have in the corner of its icon as a badge. Setting this to 0 removes the badge.
* **content-available**: Use with value 1 to send a silent notification to the user. It will not play any sound, or set any badge number, but it will wake up the app so it can communicate with the server.

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/backround_fetch_push.png">

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
  // Override point for customization after application launch.

  application.applicationIconBadgeNumber = 0; // Clear badge when app launches

  // Check if launched from notification
  if let notification = launchOptions?[UIApplicationLaunchOptionsRemoteNotificationKey] as? [String: AnyObject] {
    	// launched with notification
    	notificationReceived(notification: notification)
    } else {
      	window?.rootViewController?.present(ViewController(), animated: true, completion: nil) // Setup UI in this case 
      	registerPushNotifications()
    }
    return true
}

func registerPushNotifications() {
	let types = [UIUserNotificationType.alert, UIUserNotificationType.badge, UIUserNotificationType.sound]
    if #available(iOS 10.0, *) {
        let options = types.authorizationOptions()
        UNUserNotificationCenter.current().requestAuthorization(options) { (granted, error) in
            if granted {
                self.application.registerForRemoteNotifications()
            }
        }
    } else {
        let settings = UIUserNotificationSettings(types: types, categories: nil)
        application.registerUserNotificationSettings(settings)
        application.registerForRemoteNotifications()
    }
}

func application(_ application: UIApplication, performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    NetworkServiceLayer.fetchNewBatch(completion: completionHandler)
}

```

* On receive of the push notification application:performFetchWithCompletionHandler: gets called
* You only have 30 seconds, after this period app will be terminated by the system
* If application starts in background application:didFinishLaunchingWithOptions: is also called, so you can handle this with launchOptions key UIApplicationLaunchOptions.remoteNotification

## Inter-App Communication

A URL scheme lets you communicate with other apps through a protocol that you define. To communicate with an app that implements such a scheme, you must create an appropriately formatted URL and ask the system to open it. To implement support for a custom scheme, you must declare support for the scheme and handle incoming URLs that use the scheme.

**Handling URL Requests**

An app that has its own custom URL scheme must be able to handle URLs passed to it. All URLs are passed to your app delegate, either at launch time or while your app is running or in the background. To handle incoming URLs, your delegate should implement the following methods:

* Use the application:willFinishLaunchingWithOptions: and application:didFinishLaunchingWithOptions: methods to retrieve information about the URL and decide whether you want to open it. If either method returns NO, your app’s URL handling code is not called.
* Use the application:openURL:sourceApplication:annotation: method to open the file.

If your app is not running when a URL request arrives, it is launched and moved to the foreground so that it can open the URL. The implementation of your application:willFinishLaunchingWithOptions: or application:didFinishLaunchingWithOptions: method should retrieve the URL from its options dictionary and determine whether the app can open it. If it can, return YES and let your application:openURL:sourceApplication:annotation: (or application:handleOpenURL:) method handle the actual opening of the URL. (If you implement both methods, both must return YES before the URL can be opened.) 

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/applaunch_url.png">
<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/ios/applaunch_url_background.png">

**Displaying a Custom Launch Image When a URL is Opened**
Apps that support custom URL schemes can provide a custom launch image for each scheme. When the system launches your app to handle a URL and no relevant snapshot is available, it displays the launch image you specify. To specify a launch image, provide a PNG image whose name uses the following naming conventions:

<basename>-<url_scheme><other_modifiers>.png

In this naming convention, basename represents the base image name specified by the UILaunchImageFile key in your app’s Info.plist file. If you do not specify a custom base name, use the string Default. The <url_scheme> portion of the name is your URL scheme name. To specify a generic launch image for the myapp URL scheme, you would include an image file with the name Default-myapp@2x.png in the app’s bundle. (The @2x modifier signifies that the image is intended for Retina displays. If your app also supports standard resolution displays, you would also provide a Default-myapp.png image.)

## Handle UIApplication launch options

### Opening from URL

Apps can launch other apps by passing URLs:
```objc
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"app://..."]];
```

> 1. UIApplicationLaunchOptionsURLKey: Indicates that the app was launched in order to open a URL. The value of this key is an NSURL object containing the URL to open.

> 2. UIApplicationLaunchOptionsSourceApplicationKey: Identifies the app that requested the launch of your app. The value of this key is an NSString object that represents the bundle ID of the app that made the request

> 3. UIApplicationLaunchOptionsAnnotationKey: Indicates that custom data was provided by the app that requested the opening of the URL. The value of this key is a property-list object containing the custom data.

### Remote Notification

To register for remote notifications, registerForRemoteNotificationTypes: is called in application:didFinishLaunchingWithOptions:.

…which, if successful, calls -application:didRegisterForRemoteNotificationsWithDeviceToken:. Once the device has been successfully registered, it can receive push notifications at any time.

If an app receives a notification while in the foreground, its delegate will call application:didReceiveRemoteNotification:. However, if the app is launched, perhaps by swiping the alert in notification center, application:didFinishLaunchingWithOptions: is called with the UIApplicationLaunchOptionsRemoteNotificationKey launch option:

> UIApplicationLaunchOptionsRemoteNotificationKey: Indicates that a remote notification is available for the app to process. The value of this key is an NSDictionary containing the payload of the remote notification. > - alert: Either a string for the alert message or a dictionary with two keys: body and show-view. > - badge: A number indicating the quantity of data items to download from the provider. This number is to be displayed on the app icon. The absence of a badge property indicates that any number currently badging the icon should be removed. > - sound: The name of a sound file in the app bundle to play as an alert sound. If “default” is specified, the default sound should be played.

A common approach is to have application:didFinishLaunchingWithOptions: manually call application:didReceiveRemoteNotification::

```objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ...

    if (launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey]) {
        [self application:application didReceiveRemoteNotification:launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey]];
    }
}
```

### Local Notification

Apps can schedule UILocalNotifications to trigger at some future time or interval. If the app is active in the foreground at that time, the app calls -application:didReceiveLocalNotification: on its delegate. However, if the app is not active, the notification will be posted to Notification Center.

Unlike remote notifications, UIApplication delegate provides a unified code path for handling local notifications. If an app is launched through a local notification, it calls -application:didFinishLaunchingWithOptions: followed by -application:didReceiveLocalNotification: (that is, there is no need to call it from -application:didFinishLaunchingWithOptions: like remote notifications).

A local notification populates the launch options on UIApplicationLaunchOptionsLocalNotificationKey, which contains a payload with the same structure as a remote notification:

UIApplicationLaunchOptionsLocalNotificationKey: Indicates that a local notification is available for the app to process. The value of this key is an NSDictionary containing the payload of the local notification.

In the case where it is desirable to show an alert for a local notification delivered when the app is active in the foreground, and otherwise wouldn’t provide a visual indication, here’s how one might use the information from UILocalNotification to do it manually:

```objc
- (void)application:(UIApplication *)application
didReceiveLocalNotification:(UILocalNotification *)notification
{
    if (application.applicationState == UIApplicationStateActive) {
        UIAlertView *alertView =
            [[UIAlertView alloc] initWithTitle:notification.alertAction
                                       message:notification.alertBody
                                      delegate:nil
                             cancelButtonTitle:NSLocalizedString(@"OK", nil)
                             otherButtonTitles:nil];

        if (!self.localNotificationSound) {
            NSURL *soundURL = [[NSBundle mainBundle] URLForResource:@"Sosumi"
                                                      withExtension:@"wav"];
            AudioServicesCreateSystemSoundID((__bridge CFURLRef)soundURL, &_localNotificationSound);
        }
        AudioServicesPlaySystemSound(self.localNotificationSound);

        [alertView show];
    }
}

- (void)applicationWillTerminate:(UIApplication *)application {
    if (self.localNotificationSound) {
        AudioServicesDisposeSystemSoundID(self.localNotificationSound);
    }
}
```

### Location Event

> UIApplicationLaunchOptionsLocationKey: Indicates that the app was launched in response to an incoming location event. The value of this key is an NSNumber object containing a Boolean value. You should use the presence of this key as a signal to create a CLLocationManager object and start location services again. Location data is delivered only to the location manager delegate and not using this key.

Example to handle the significant location changes:

```objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ...

    if (![CLLocationManager locationServicesEnabled]) {
        [[[UIAlertView alloc] initWithTitle:NSLocalizedString(@"Location Services Disabled", nil)
                                    message:NSLocalizedString(@"You currently have all location services for this device disabled. If you proceed, you will be asked to confirm whether location services should be reenabled.", nil)
                                   delegate:nil
                          cancelButtonTitle:NSLocalizedString(@"OK", nil)
                          otherButtonTitles:nil] show];
    } else {
        self.locationManager = [[CLLocationManager alloc] init];
        self.locationManager.delegate = self;
        [self.locationManager startMonitoringSignificantLocationChanges];
    }

    if (launchOptions[UIApplicationLaunchOptionsLocationKey]) {
        [self.locationManager startUpdatingLocation];
    }
}
```

### Newsstand

Newsstand can launch when newly-downloaded assets are available.

This is how you register:
```objc
[application registerForRemoteNotificationTypes: UIRemoteNotificationTypeNewsstandContentAvailability];
```

And this is the key to look out for in launchOptions:

> UIApplicationLaunchOptionsNewsstandDownloadsKey: Indicates that newly downloaded Newsstand assets are available for your app. The value of this key is an array of string identifiers that identify the NKAssetDownload objects corresponding to the assets. Although you can use the identifiers for cross-checking purposes, you should obtain the definitive array of NKAssetDownload objects (representing asset downloads in progress or in error) through the downloadingAssets property of the NKLibrary object representing the Newsstand app’s library.

### Bluetooth

If an app launches, instantiates a CBCentralManager or CBPeripheralManager with a particular identifier, and connects to other Bluetooth peripherals, the app can be re-launched by certain actions from the Bluetooth system. Depending on whether it was a central or peripheral manager that was notified, one of the following keys will be passed into launchOptions:

> 1. UIApplicationLaunchOptionsBluetoothCentralsKey: Indicates that the app previously had one or more CBCentralManager objects and was relaunched by the Bluetooth system to continue actions associated with those objects. The value of this key is an NSArray object containing one or more NSString objects. Each string in the array represents the restoration identifier for a central manager object.

> 2. UIApplicationLaunchOptionsBluetoothPeripheralsKey: Indicates that the app previously had one or more CBPeripheralManager objects and was relaunched by the Bluetooth system to continue actions associated with those objects. The value of this key is an NSArray object containing one or more NSString objects. Each string in the array represents the restoration identifier for a peripheral manager object.

Example of the handling:

```objc
self.centralManager = [[CBCentralManager alloc] initWithDelegate:self queue:nil options:@{CBCentralManagerOptionRestoreIdentifierKey:(launchOptions[UIApplicationLaunchOptionsBluetoothCentralsKey] ?: [[NSUUID UUID] UUIDString])}];

if (self.centralManager.state == CBCentralManagerStatePoweredOn) {
    static NSString * const UID = @"7C13BAA0-A5D4-4624-9397-15BF67161B1C"; // generated with `$ uuidgen`
    NSArray *services = @[[CBUUID UUIDWithString:UID]];
    NSDictionary *scanOptions = @{CBCentralManagerScanOptionAllowDuplicatesKey:@YES};
    [self.centralManager scanForPeripheralsWithServices:services options:scanOptions];
}
```

# Resources
[Oleb Blog](https://oleb.net/blog/2011/06/app-launch-sequence-ios/)
[NSHipster](http://nshipster.com/launch-options/)
[Habrahabr 1](https://habrahabr.ru/post/208434/)
[SitePoint](https://www.sitepoint.com/developing-push-notifications-for-ios-10/)
[Habrahabr 2](https://habrahabr.ru/company/e-Legion/blog/303970/)








# App Groups and App Share
# App Extensions

https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html

## Shared User Defaults
https://medium.com/@JoyceMatos/today-extensions-in-ios-10-and-swift-3-f7dc316d9df9
https://medium.com/ios-os-x-development/shared-user-defaults-in-ios-3f15cd2c9409

## Today Extension
https://medium.com/ios-os-x-development/learnings-from-building-a-today-view-extension-in-ios-8-710d5f481594

## Siri
https://medium.com/ios-os-x-development/extending-your-ios-app-with-sirikit-fd1a7ef12ba6

## Widgets 
https://hackernoon.com/app-extensions-and-today-extensions-widget-in-ios-10-e2d9fd9957a8

## Notification Extensions
https://habrahabr.ru/company/e-Legion/blog/303970/

# Resources
[](http://www.atomicbird.com/blog/sharing-with-app-extensions)
[](https://medium.com/@maximbilan/ios-shared-coredata-storage-for-app-groups-447b4ba43eec)
[](https://medium.com/@JoyceMatos/today-extensions-in-ios-10-and-swift-3-f7dc316d9df9)
[](https://medium.com/@wireapp/the-challenge-of-implementing-ios-share-extension-for-end-to-end-encrypted-messenger-dd33b52b1e97)
[](https://medium.com/@stasost/ios-how-to-open-deep-links-notifications-and-shortcuts-253fb38e1696)
[](https://medium.com/@tackmobile/app-groups-and-imessage-extensions-for-ios-10-28d169494b2)


# iOS Data Persistence
https://medium.com/@deepcut/understanding-ios-data-persistence-nsuserdefaults-and-the-file-system-cf062dc53761
https://medium.com/@robdeans/exploring-ioss-sandbox-b72e4697ab2f
## Bundle
https://developer.apple.com/library/content/documentation/CoreFoundation/Conceptual/CFBundles/AccessingaBundlesContents/AccessingaBundlesContents.html
## Filesystem
https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html
https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/Introduction/Introduction.html
https://www.imore.com/apfs
https://www.cultofmac.com/435718/apfs-new-apple-file-system/
## UserDefaults
https://developer.apple.com/documentation/foundation/userdefaults
https://habrahabr.ru/post/324400/
http://macbug.ru/cocoa/userdefns
http://www.imaladec.com/story/nsuserdefaults
## Keychain
https://medium.com/ios-os-x-development/securing-user-data-with-keychain-for-ios-e720e0f9a8e2
https://developer.apple.com/library/content/documentation/Security/Conceptual/keychainServConcepts/iPhoneTasks/iPhoneTasks.html
https://developer.apple.com/library/content/documentation/Security/Conceptual/keychainServConcepts/01introduction/introduction.html
http://macbug.ru/cocoa/keychainios
https://www.imore.com/how-use-icloud-keychain-iphone-and-ipad
## Cache
https://medium.com/ios-os-x-development/caching-anything-in-ios-102176e46eba
http://nshipster.com/nscache/
https://habrahabr.ru/post/254969/
## CoreData
https://cocoacasts.com/an-introductory-core-data-tutorial
https://habrahabr.ru/company/livetyping/blog/309624/
https://habrahabr.ru/post/218457/
https://medium.cobeisfresh.com/ios-how-to-store-your-users-last-network-request-for-seamless-app-launch-with-swift-8ecdf119db4a
https://medium.com/flawless-app-stories/cracking-the-tests-for-core-data-15ef893a3fee
https://www.objc.io/issues/4-core-data/full-core-data-application/
## iCloud
https://habrahabr.ru/company/ZeptoLab/blog/137947/
https://habrahabr.ru/company/everydaytools/blog/326050/
https://habrahabr.ru/post/224961/
https://medium.com/@rodrigo_freitas/cloudkit-cdda2f725042
https://blog.frozenfirestudios.com/cloudkit-operations-319523a2a6d3




# Multithreading
https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i
https://developer.apple.com/documentation/foundation/nstimer#1667624?language=objc
http://macbug.ru/nstimer/ns0
https://medium.com/@dkw5877/more-gcd-semaphores-and-dispatch-groups-5b767c700a03
https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html
http://macbug.ru/cocoa/mthreadrunloops







# Deeplinks notifications and shortcuts

**Local notification triggering**

```swift
func scheduleNotification(identifier: String, title: String, subtitle: String, body: String, timeInterval: TimeInterval, repeats: Bool = false) {
    if #available(iOS 10, *) {
        let content = UNMutableNotificationContent()
        content.title = title
        content.subtitle = subtitle
        content.body = body

        let trigger = UNTimeIntervalNotificationTrigger(timeInterval: timeInterval, repeats: repeats)
        let request = UNNotificationRequest(identifier: identifier, content: content, trigger: trigger)
        UNUserNotificationCenter.current().add(request, withCompletionHandler: nil)
    } else {
        let notification = UILocalNotification()
        notification.alertBody = "\(title)\n\(subtitle)\n\(body)"
        notification.fireDate = Date(timeIntervalSinceNow: 1)

        application.scheduleLocalNotification(notification)
    }
}
```

**Showing in foreground**

```swift
@available(iOS 10, *)
public func userNotificationCenter(_ center: UNUserNotificationCenter, willPresent notification: UNNotification, withCompletionHandler completionHandler: (UNNotificationPresentationOptions) -> Void) {
    completionHandler([.alert, .sound, .badge])
}
```

https://www.sitepoint.com/developing-push-notifications-for-ios-10/
https://medium.com/@piyush.dez/downloading-in-background-using-swift-3-0-f770531db619

https://medium.com/@stasost/ios-how-to-open-deep-links-notifications-and-shortcuts-253fb38e1696

https://medium.com/@abhimuralidharan/universal-links-in-ios-79c4ee038272




















































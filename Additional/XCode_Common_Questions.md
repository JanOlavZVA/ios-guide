# Impact of Xcode build options “Enable bitcode” Yes/No

>What does the ENABLE_BITCODE actually do, will it be a non-optional requirement in the future?

When you build your project, Xcode invokes clang for Objective-C targets and swift/swiftc for Swift targets. Both of these compilers **compile** the app to an **intermediate representation (IR)**, one of these IRs is bitcode. From this IR, a program called LLVM takes over and creates the **binaries needed for x86 32 and 64 bit modes (for the simulator) and arm6/arm7/arm7s/arm64 (for the device).** Normally, all of these different binaries are lumped together in a **single file** called a fat binary.

The ENABLE_BITCODE option **cuts out** this final step. It creates a version of the app with an IR bitcode binary. This has a number of nice features, but one giant drawback: it can't run anywhere. In order to get an app with a bitcode binary to run, the bitcode needs to be recompiled (maybe assembled or transcoded… I'm not sure of the correct verb) into an x86 or ARM binary.

When a bitcode app is submitted to the App Store, Apple will do this final step and create the finished binaries.

Right now, bitcode apps are optional, but history has shown Apple turns optional things into requirements (like 64 bit support). This usually takes a few years, so third party developers (like Parse) have time to update.

>Can I use the above method without any negative impact and without compromising a future appstore submission?
Yes, you can turn off ENABLE_BITCODE and everything will work just like before. Until Apple makes bitcode apps a requirement for the App Store, you will be fine.

>Are there any performance impacts if I enable / disable it?
There will never be negative performance impacts for enabling it, but internal distribution of an app for testing may get more complicated.

**APPLE DOCS**

>Bitcode is an intermediate representation of a compiled program. Apps you upload to iTunes Connect that contain bitcode will be compiled and linked on the store. Including bitcode will allow Apple to re-optimize your app binary in the future without the need to submit a new version of your app to the store.
>
>Xcode hides symbols generated during build time by default, so they are not readable by Apple. Only if you choose to include symbols when uploading your app to iTunes Connect would the symbols be sent to Apple. You must include symbols to receive crash reports from Apple.
>
>Note: For iOS apps, bitcode is the default, but optional. For watchOS and tvOS apps, bitcode is required. If you provide bitcode, all apps and frameworks in the app bundle (all targets in the project) need to include bitcode. After you distribute your app using iTunes Connect, you can download the dSYMs file for the build, described in Viewing and Importing Crashes in the Devices Window
>
>Apple's initial roll-out of the bitcode and app thinning service was put on hold, because issues in upgrading from one type of hardware to a different type of hardware didn't restore the right versions of binaries. This issue was subsequently fixed with iOS 9.0.2 and the feature re-enabled.
>
>Bitcode has always been a part of the LLVM compile and optimisation phases, but by moving the back-end logic to the Apple servers, it moves the optimise and assemble phases from developer compile time to App Store deployment. This unlocks the potential of future re-optimisation or re-translation to support newer and faster processors in future. Bitcode deployments are required for watchOS and tvOS deploments, and can be conditionally enabled for existing iOS deployments with the "Enable Bitcode" option in the project settings. This will add a flag embed-bitcode-marker for debug builds, and embed-bitcode for archive/device builds. These can be passed to the Swift compiler with -embed-bitcode or by using clang with -fembed-bitcode.
>
>Bitcode also has some disadvantages. Developers can debug crash reports from applications by storing copies of the debug symbols corresponding to the binary that was shipped to Apple. When a crash happens in a given stack, the developer can restore the original stack trace by symbolicating the crash report, using these debug symbols. However, the symbols are a by-product of translating the intermediate form to the binary; but if that step is done on the server, this information is lost. Apple provides a crash reporting service that can play the part of the debugger, provided that the developer has uploaded the debug symbols at the time of application publication. The fact that the developer never sees the exact binary means that they may not be able to test for speciic issues as new hardware evolves. There are also some concerns about ceding power to Apple to perform compilation – including the ability to inject additional routines or code snippets – but since Apple is in full control of the publication process these are currently possible whether or not the developer uses bitcode or compiled binaries.
>
>Finally, the bitcode on the server can be translated to support new architectures and instruction sets as they evolve. Provided that they maintain the calling convention and size of the alignment and words, a bitcode application might be translated into different architecture types and optimised specifically for a new processor. If standard libraries for math and vector routines are used, these can be optimised into processor specific vector instructions to gain the best performance for a given application. The optimisers might even generate multiple different encodings and judge based on size or execution speed.


# Slicing

Slicing is the process of creating and delivering variants of the app bundle for different target devices. A variant contains only the executable architecture and resources that are needed for the target device. You continue to develop and upload full versions of your app to iTunes Connect. The store will create and deliver different variants based on the devices your app supports. Image resources are sliced according to their resolution and device family. GPU resources are sliced according to device capabilities. For tvOS apps, assets in catalogs shared between iOS and tvOS targets are sliced and large app icons are removed. When the user installs an app, a variant for the user’s device is downloaded and installed.

Xcode simulates slicing during development so you can create and test variants locally. Xcode slices your app when you build and run your app on a device. When you create an archive, Xcode includes the full version of your app but allows you to export variants from the archive.

**Note: Sliced apps are supported on devices running 9.0 and later; otherwise, the store delivers universal apps to customers.**

**Note: To test the variants that the store builds before you distribute your app to users, invite internal testers (your team’s iTunes Connect users) only and download the variants using TestFlight. If you invite external testers (users specifying only their email addresses), the external testers must wait for Beta App Review to approve the app before they can download variants.**

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/XCode_Series/slicing.png">

Slicing is performed during your normal development and distribution workflows, which proceed generally as follows:

* In Xcode, specify target devices and provide multiple resolutions of images in the asset catalog.
You must use the asset catalog in order for resources to be sliced.

* Build and run the app in Simulator or on a device.
Xcode builds a variant of the app for the selected device type, improving the debug build time and allowing you to test variants locally.

* Create an archive of the app and export a variant locally for target devices.
Test all the variants you export on target devices to discover configuration issues early.

* Upload the app to iTunes Connect.
The store creates individual app variants from the archive. The number of variants depends on the architectures and resources specified in the Xcode project.

* In iTunes Connect, distribute a prerelease version of your app to designated testers.
Testers install the prerelease version on supported devices using TestFlight. TestFlight downloads a variant of the app specific to the user’s device.

* In iTunes Connect, release the app.
Users install the app on supported devices, and the store app downloads a variant of the app specific to the user’s device.



































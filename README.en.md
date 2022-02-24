# Flutter Hand Tracking Plugin

This `Flutter Packge` is to call the `Andorid` device camera to accurately track and recognize the motion path/trajectory and gesture actions of the ten fingers, and output 22 hand key points to support more gesture customization. Based on this package, you can write business The logic converts gesture information into instruction information in real time: one, two, three, four, five, rock, spiderman... At the same time, based on `Flutter`, different special effects can be written for different gestures. It can be used for short video live effects, intelligent hardware and other fields, for human-computer interaction Bring a more natural and rich experience.

![demo1](img/demo1.gif)![demo2](img/demo2.gif)

> The source code is hosted on Github: [https://github.com/zhouzaihang/flutter_hand_tracking_plugin](https://github.com/zhouzaihang/flutter_hand_tracking_plugin)

> [Bilibili Demo](https://www.bilibili.com/video/av92842489/)

## use

> The `android/libs/hand_tracking_aar.aar` used in the project is hosted in `git-lfs`, after you download it, you need to confirm whether the `.aar` file exists (and the file exceeds 100MB). If you do not have `git-lfs installed ` You may need to download it manually and replace it in your project path.

This project is a starting point for a Flutter
[plug-in package](https://flutter.dev/developing-packages/),
a specialized package that includes platform-specific implementation code for
Android.

For help getting started with Flutter, view our
[online documentation](https://flutter.dev/docs), which offers tutorials,
samples, guidance on mobile development, and a full API reference.

## Technology involved

1. Write a `Flutter Plugin Package`
1. Use `Docker` to configure the `MediaPipe` development environment
1. Using `MediaPipe` in `Gradle`
1. The `Flutter` program runs the `MediaPipe` graph
1. Embed native views in `Flutter` pages
1. Use of `protobuf`

## What is `Flutter Package`

`Flutter Package` has the following two types:

`Dart Package`: Packages written entirely in `Dart`, such as the `path` package. Some of these may contain `Flutter` specific functionality, such packages are completely dependent on the `Flutter` framework.

`Plugin Package`: A class of runtime-dependent packages that contain an `API` written in `Dart` code, combined with `Android` (using `Java` or `Kotlin`) and `iOS` (using `Java` or `Kotlin`) `ObjC` or `Swift`) platform specific implementation. For example the `battery` package.

## Why do you need `Flutter Plugin Package`

As a cross-platform `UI` framework, `Flutter` itself cannot directly call native functions. If you need to use the functions of the native system, you need to implement platform-specific implementations, and then implement them in the `Dart` layer of `Flutter` compatible.
Here you need to use the calling camera and `GPU` to implement the business. So use the `Flutter Plugin Package`.

## How `Flutter Plugin Package` works

Here is the directory of the `Flutter Plugin Package` project:

![Flutter Plugin Directory Structure](img/directory_structure.png)

- where `pubspec.yaml` is used to add dependencies or resources (images, fonts, etc.) that `Plugin` may use

- Under the `example` directory is a complete `Flutter APP`, a `Plugin` for test writing

- In addition, there are three directories `android`, `ios` and `lib` in both a `Flutter app` project and a `Flutter Plugin` project. The `lib` directory is used to store the `Dart` code, and the other The two directories are used to store platform-specific implementation code. `Flutter` will run the code corresponding to the platform according to the actual operating platform, and then use [Platform Channels](https://flutter.dev/docs/development/platform -integration/platform-channels?tab=android-channel-kotlin-tab) returns the result of running the code to the `Dart` layer.

The following is a `Flutter` architecture diagram officially given by `Flutter`:

![Flutter System Overview](img/flutter_system_overview.png)

As can be seen from the architecture diagram, the reason why `Flutter` is a cross-platform framework is that there is `Embedder` as the operating system adaptation layer, the `Engine` layer implements functions such as rendering engines, and the `Framework` layer is a `Dart` layer. ` Implemented `UI SDK`. For a `Flutter Plugin Package`, it is to use a native platform-specific implementation in the `Embedder` layer, and encapsulate it as a `UI API` in the `Dart` layer, so as to realize cross Platform. The `Embedder` layer cannot be directly connected to the `Framework`, but must go through the `Platform Channels` of the `Engine` layer.

The process of using `Platform Channels` to pass between the client (`UI`) and the host (specific platform) is shown in the following diagram:

![PlatformChannels](img/PlatformChannels.png)


## New `Flutter Plugin Package`

1. Open `Android Studio`, click `New Flutter Project`
1. Select the `Flutter Plugin` option
1. Enter the project name, description and other information

## write `Android` platform `view`

First create two `kotlin` files in `android/src/main/kotlin/xyz/zhzh/flutter_hand_tracking_plugin` directory: `FlutterHandTrackingPlugin.kt` and `HandTrackingViewFactory.kt` files.

### Write the `Factory` class

Write a `HandTrackingViewFactory` class in `HandTrackingViewFactory.kt` to implement the abstract class `PlatformViewFactory`. The `Android` platform components written later need to use this `Factory` class to generate. When generating a view, you need to pass in a parameter` id` to identify the view (`id` will be created by `Flutter` and passed to `Factory`):

``` Kotlin
package xyz.zhzh.flutter_hand_tracking_plugin

import android.content.Context
import io.flutter.plugin.common.PluginRegistry
import io.flutter.plugin.common.StandardMessageCodec
import io.flutter.plugin.platform.PlatformView
import io.flutter.plugin.platform.PlatformViewFactory

class HandTrackingViewFactory(private val registrar: PluginRegistry.Registrar) :
        PlatformViewFactory(StandardMessageCodec.INSTANCE) {
    override fun create(context: Context?, viewId: Int, args: Any?): PlatformView {
        return FlutterHandTrackingPlugin(registrar, viewId)
    }
}
````

### Write the `AndroidView` class

Write `FlutterHandTrackingPlugin` in `FlutterHandTrackingPlugin.kt` to implement `PlatformView` interface, this interface needs to implement two methods `getView` and `dispose`.

`getView` is used to return a view that will be embedded into the `Flutter` interface

`dispose` does something when trying to close

First add a `SurfaceView`:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    companion object {
        private const val TAG = "HandTrackingPlugin"
        private const val NAMESPACE = "plugins.zhzh.xyz/flutter_hand_tracking_plugin"

        @JvmStatic
        fun registerWith(registrar: Registrar) {
            registrar.platformViewRegistry().registerViewFactory(
                    "$NAMESPACE/view",
                    HandTrackingViewFactory(registrar))
        }

        init { // Load all native libraries needed by the app.
            System.loadLibrary("mediapipe_jni")
            System.loadLibrary("opencv_java3")
        }
    }
    private var previewDisplayView: SurfaceView = SurfaceView(r.context())
}
````

Then return the added `SurfaceView` via `getView`:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    // ...
    override fun getView(): SurfaceView? {
        return previewDisplayView
    }

    override fun dispose() {
        // TODO: ViewDispose()
    }
}
````

## Call the native implementation of `View` in `Dart`

> Open the `lib/flutter_hand_tracking_plugin.dart` of the `plugin package` project for editing (the specific file name is based on the package name created when the new project is created).

To call a native `Android` component in `Flutter`, you need to create an `AndroidView` and tell it the registration name of the component. When you create an `AndroidView`, an `id` is assigned to the component, and this `id` can be passed through the parameter` The onPlatformViewCreated` incoming method gets:

```` dart
AndroidView(
    viewType: '$NAMESPACE/blueview',
    onPlatformViewCreated: (id) => _id = id),
)
````

Since only the components of the `Android` platform are implemented, they cannot be used on other systems, so you also need to obtain the `defaultTargetPlatform` to determine the running platform:

```` dart
import 'dart:async';

import 'package:flutter/cupertino.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:flutter_hand_tracking_plugin/gen/landmark.pb.dart';

const NAMESPACE = "plugins.zhzh.xyz/flutter_hand_tracking_plugin";

typedef void HandTrackingViewCreatedCallback(
    HandTrackingViewController controller);

class HandTrackingView extends StatelessWidget {
  const HandTrackingView({@required this.onViewCreated})
      : assert(onViewCreated != null);

  final HandTrackingViewCreatedCallback onViewCreated;

  @override
  Widget build(BuildContext context) {
    switch (defaultTargetPlatform) {
      case TargetPlatform.android:
        return AndroidView(
          viewType: "$NAMESPACE/view",
          onPlatformViewCreated: (int id) => onViewCreated == null
              ? null
              : onViewCreated(HandTrackingViewController._(id)),
        );
      case TargetPlatform.fuchsia:
      case TargetPlatform.iOS:
      default:
        throw UnsupportedError(
            "Trying to use the default webview implementation for"
            " $defaultTargetPlatform but there isn't a default one");
    }
  }
}
````

The above uses `typedef` to define a `HandTrackingViewCreatedCallback`, the incoming parameter type is `HandTrackingViewController`, this `controller` is used to manage the `id` corresponding to `AndroidView`:

```` dart
class HandTrackingViewController {
  final MethodChannel _methodChannel;

  HandTrackingViewController._(int id)
      : _methodChannel = MethodChannel("$NAMESPACE/$id"),
        _eventChannel = EventChannel("$NAMESPACE/$id/landmarks");

  Future<String> get platformVersion async =>
      await _methodChannel.invokeMethod("getPlatformVersion");
}
````

The `MethodChannel` is used to call the method of the `Flutter Plugin Package`, this time you don't need to use the `MethodChannel`, so don't pay attention.

## Build `MediaPipe AAR` with `Docker` and add it to the project

`MediaPipe` is a cross-platform framework released by `Google` that uses `ML pipelines` technology to build multiple models connected together. (`Machine learning pipelines`: Simply put, it is a set of `API` to solve each model/algorithm/` data transfer between workflow`). `MediaPipe` supports video, audio, etc. any `time series data`([WiKi--Time Series](https://en.wikipedia.org/wiki/Time_series)).

Here we use `MediaPipe` to pass the camera data into the `TFlite` model of gesture detection for processing. Then build the whole program as `Android archive library`.

`MediaPipe Android archive library` is a way to use `MediaPipe` with `Gradle`. `MediaPipe` does not ship a regular AAR that can be used in all projects, so developers need to build it themselves. This is officially given [MediaPipe Installation tutorial](https://github.com/google/mediapipe/blob/master/mediapipe/docs/install.md). The author here is the `Ubuntu` system, and chose the `Docker` installation method (`git clone` If the network is unstable during `docker pull`, you can set `proxy` or change the source).

After the installation is complete, use `docker exec -it mediapipe /bin/bash` to enter the `bash` operation.

### Create a `mediapipe_aar()`

First create a `BUILD` file in `mediapipe/examples/android/src/java/com/google/mediapipe/apps/aar_example` and add the following contents to the text file.

``` build
load("//mediapipe/java/com/google/mediapipe:mediapipe_aar.bzl", "mediapipe_aar")

mediapipe_aar(
    name = "hand_tracking_aar",
    calculators = ["//mediapipe/graphs/hand_tracking:mobile_calculators"],
)
````

### Generate `aar`

According to the text file created above, run the `bazel build` command to generate an `AAR`, where `--action_env=HTTP_PROXY=$HTTP_PROXY`, this parameter is used to specify the proxy setting (because during the build process will be Download many dependencies from `Github`.)

``` bash
bazel build -c opt --action_env=HTTP_PROXY=$HTTP_PROXY --action_env=HTTPS_PROXY=$HTTPS_PROXY --fat_apk_cpu=arm64-v8a,armeabi-v7a mediapipe/examples/android/src/java/com/google/mediapipe/apps/ aar_example:hand_tracking_aar
````

In the `/mediapipe/examples/android/src/java/com/google/mediapipe/apps/aar_example` path inside the container, you can find the `hand_tracking_aar.aar` just built, use `docker cp` from the container Copy it to the project's `android/libs`.

### Generate `binary graph`

The execution of the above `aar` also depends on the `MediaPipe binary graph`, which can be constructed and generated using the following command to generate a `binary graph`:

``` bash
bazel build -c opt mediapipe/examples/android/src/java/com/google/mediapipe/apps/handtrackinggpu:binary_graph
````

Copy the `binary graph` from `build` from `bazel-bin/mediapipe/examples/android/src/java/com/google/mediapipe/apps/handtrackinggpu`, when it goes to `android/src/main/assets` Down

### Add `assets` and `OpenCV library`

In the `/mediapipe/models` directory of the container, you will find `hand_lanmark.tflite`, `palm_detection.tflite` and `palm_detection_labelmap.txt` Copy these to `android/src/main/assets`.

In addition, `MediaPipe` depends on `OpenCV`, so you need to download the `JNI libraries` library precompiled by `OpenCV` and place it in the `android/src/main/jniLibs` path. You can download it from [here](https: //github.com/opencv/opencv/releases/download/3.4.3/opencv-3.4.3-android-sdk.zip) Download the official `OpenCV Android SDK` and run `cp` into the corresponding path

``` bash
cp -R ~/Downloads/OpenCV-android-sdk/sdk/native/libs/arm* /path/to/your/plugin/android/src/main/jniLibs/
````

![add aar in plugin](img/mediapipe.png)

The `MediaPipe` framework uses `OpenCV`. To load the `MediaPipe` framework, you must first load `OpenCV` in the `Flutter Plugin`. Use the following code in the `companion object` of the `FlutterHandTrackingPlugin` to load these two dependencies:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    companion object {
        init { // Load all native libraries needed by the app.
            System.loadLibrary("mediapipe_jni")
            System.loadLibrary("opencv_java3")
        }
    }
}
````

### Modify `build.grage`

Open `android/build.gradle`, add `MediaPipe dependencies` and `MediaPipe AAR` to `app/build.gradle`:

````
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    implementation fileTree(dir: 'libs', include: ['*.aar'])
    // MediaPipe deps
    implementation 'com.google.flogger:flogger:0.3.1'
    implementation 'com.google.flogger:flogger-system-backend:0.3.1'
    implementation 'com.google.code.findbugs:jsr305:3.0.2'
    implementation 'com.google.guava:guava:27.0.1-android'
    implementation 'com.google.guava:guava:27.0.1-android'
    implementation 'com.google.protobuf:protobuf-lite:3.0.1'
    // CameraX core library
    def camerax_version = "1.0.0-alpha06"
    implementation "androidx.camera:camera-core:$camerax_version"
    implementation "androidx.camera:camera-camera2:$camerax_version"
    implementation "androidx.core:core-ktx:1.2.0"
}
````

## Call the camera via `CameraX`

### Get camera permission

To use the camera in our application, we need to request the user to provide access to the camera. To request the camera permission, add the following to `android/src/main/AndroidManifest.xml`:

````xml
<!-- For using the camera -->
<uses-permission android:name="android.permission.CAMERA" />

<uses-feature android:name="android.hardware.camera" />
<uses-feature android:name="android.hardware.camera.autofocus" />
<!-- For MediaPipe -->
<uses-feature
    android:glEsVersion="0x00020000"
    android:required="true" />
````

Also in `build.gradle` change the minimum `SDK` version to above `21` and the target `SDK` version to above `27`

``` Gradle Script
defaultConfig {
    // TODO: Specify your own unique Application ID (https://developer.android.com/studio/build/application-id.html).
    applicationId "xyz.zhzh.flutter_hand_tracking_plugin_example"
    minSdkVersion 21
    targetSdkVersion 27
    versionCode flutterVersionCode.toInteger()
    versionName flutterVersionName
    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
}
````

To make sure the user is prompted to request the camera permission and enable us to access the camera using the `CameraX` library. To request the camera permission, you can use the component `PermissionHelper` provided by the `MediaPipe` component to use it. First add the request permission inside the component `init` the code:

``` kotlin
PermissionHelper.checkAndRequestCameraPermissions(activity)
````

This prompts the user with a dialog on the screen to request permission to use the camera in this application.

Then add the following code to handle the user response:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    private val activity: Activity = r.activity()

    init {
        // ...
        r.addRequestPermissionsResultListener(CameraRequestPermissionsListener())
        PermissionHelper.checkAndRequestCameraPermissions(activity)
        if (PermissionHelper.cameraPermissionsGranted(activity)) onResume()
    }

    private inner class CameraRequestPermissionsListener :
            PluginRegistry.RequestPermissionsResultListener {
        override fun onRequestPermissionsResult(requestCode: Int,
                                                permissions: Array<out String>?,
                                                grantResults: IntArray?): Boolean {
            return if (requestCode != 0) false
            else {
                for (result in grantResults!!) {
                    if (result == PERMISSION_GRANTED) onResume()
                    else Toast.makeText(activity, "Please grant camera permission", Toast.LENGTH_LONG).show()
                }
                true
            }
        }
    }

    private fun onResume() {
        // ...
        if (PermissionHelper.cameraPermissionsGranted(activity)) startCamera()
    }
    private fun startCamera() {}
}
````

Leave the `startCamera()` method empty for now. When the user responds to the prompt, the `onResume()` method will be called. The code will confirm that permission to use the camera has been granted, and will then start the camera.

### call camera

Now add `SurfaceTexture` and `SurfaceView` to the plugin:

``` kotlin
// {@link SurfaceTexture} where the camera-preview frames can be accessed.
private var previewFrameTexture: SurfaceTexture? = null
// {@link SurfaceView} that displays the camera-preview frames processed by a MediaPipe graph.
private var previewDisplayView: SurfaceView = SurfaceView(r.context())
````

In the component `init` method, add the `setupPreviewDisplayView()` method to the front of the request for the camera permission:

``` kotlin
init {
    r.addRequestPermissionsResultListener(CameraRequestPermissionsListener())

    setupPreviewDisplayView()
    PermissionHelper.checkAndRequestCameraPermissions(activity)
    if (PermissionHelper.cameraPermissionsGranted(activity)) onResume()
}
````

Then write the `setupPreviewDisplayView` method:

``` kotlin
private fun setupPreviewDisplayView() {
    previewDisplayView.visibility = View.GONE
    // TODO
}
````

To get `previewDisplayView` for camera data, you can use `CameraX`, `MediaPipe` provides a class named `CameraXPreviewHelper` to use `CameraX`. When the camera is turned on, you can update the listener function `onCameraStarted(@Nullable SurfaceTexture) `

Now define a `CameraXPreviewHelper`:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    // ...
    private var cameraHelper: CameraXPreviewHelper? = null
    // ...
}
````

Then implement the previous `startCamera()`:

``` kotlin
private fun startCamera() {
    cameraHelper = CameraXPreviewHelper()
    cameraHelper!!.setOnCameraStartedListener { surfaceTexture: SurfaceTexture? ->
        previewFrameTexture = surfaceTexture
        // Make the display view visible to start showing the preview. This triggers the
        // SurfaceHolder.Callback added to (the holder of) previewDisplayView.
        previewDisplayView.visibility = View.VISIBLE
    }
    cameraHelper!!.startCamera(activity, CAMERA_FACING, /*surfaceTexture=*/null)
}
````

This will `new` a `CameraXPreviewHelper` object and add an anonymous listener to that object. When a `cameraHelper` listener detects that the camera is up, `surfaceTexture` can grab the camera frame and pass it to `previewFrameTexture`, And make `previewDisplayView` visible.

When calling the camera, you need to determine the camera to use. `CameraXPreviewHelper` inherits the two options of `CameraHelper`: `FRONT` and `BACK`. And pass the `cameraHelper!!.startCamera` method as the parameter `CAMERA_FACING`. Here Set the front camera to a static variable:

``` kotlin
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    companion object {
        private val CAMERA_FACING = CameraHelper.CameraFacing.FRONT
        // ...
    }
    //...
}
````
## Convert camera image frame data using `ExternalTextureConverter`

The above uses `SurfaceTexture` to capture the camera image frame from the stream and store it in the `OpenGL ES texture` object. To use the `MediaPipe graph`, you need to store the frame captured by the camera in a normal `Open GL texture` object. `MediaPipe ` The class `ExternalTextureConverter` is provided for converting image frames stored in `SurfaceTexture` objects to regular `OpenGL texture` objects.

To use `ExternalTextureConverter`, you also need an `EGLContext` created and managed by an `EglManager` object. Add the following declaration to the plugin:

``` kotlin
// Creates and manages an {@link EGLContext}.
private var eglManager: EglManager = EglManager(null)

// Converts the GL_TEXTURE_EXTERNAL_OES texture from Android camera into a regular texture to be
// consumed by {@link FrameProcessor} and the underlying MediaPipe graph.
private var converter: ExternalTextureConverter? = null
````

Modify the previously written `onResume()` method to add code to initialize the `converter` object:

``` kotlin
private fun onResume() {
    converter = ExternalTextureConverter(eglManager.context)
    converter!!.setFlipY(FLIP_FRAMES_VERTICALLY)
    if (PermissionHelper.cameraPermissionsGranted(activity)) {
        startCamera()
    }
}
````

To transfer `previewFrameTexture` to `converter` for conversion, add the following code block to `setupPreviewDisplayView()`:

``` kotlin
private fun setupPreviewDisplayView() {
    previewDisplayView.visibility = View.GONE
    previewDisplayView.holder.addCallback(
            object : SurfaceHolder.Callback {
                override fun surfaceCreated(holder: SurfaceHolder) {
                    processor.videoSurfaceOutput.setSurface(holder.surface)
                }

                override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) { // (Re-)Compute the ideal size of the camera-preview display (the area that the
                    // camera-preview frames get rendered onto, potentially with scaling and rotation)
                    // based on the size of the SurfaceView that contains the display.
                    val viewSize = Size(width, height)
                    val displaySize = cameraHelper!!.computeDisplaySizeFromViewSize(viewSize)
                    val isCameraRotated = cameraHelper!!.isCameraRotated
                    // Connect the converter to the camera-preview frames as its input (via
                    // previewFrameTexture), and configure the output width and height as the computed
                    // display size.
                    converter!!.setSurfaceTextureAndAttachToGLContext(
                            previewFrameTexture,
                            if (isCameraRotated) displaySize.height else displaySize.width,
                            if (isCameraRotated) displaySize.width else displaySize.height)
                }

                override fun surfaceDestroyed(holder: SurfaceHolder) {
                    // TODO
                }
            })
}
````

In the above code, first customize and add `SurfaceHolder.Callback` to `previewDisplayView` and implement `surfaceChanged(SurfaceHolder holder, int format, int width, int height)`:

1. Calculate the appropriate display size of the camera frame on the device screen
2. Pass `previewFrameTexture` and `displaySize` to `converter`

Now the image frame captured by the camera can be passed into the `MediaPipe graph`.

## call `MediaPipe graph`

First, you need to load all the resources needed by `MediaPipe graph` (`tflite` model copied from the container, `binary graph`, etc.), you can use the `MediaPipe` component `AndroidAssetUtil` class:

``` kotlin
// Initialize asset manager so that MediaPipe native libraries can access the app assets, eg,
// binary graphs.
AndroidAssetUtil.initializeNativeAssetManager(activity)
````

Then add the following code to set the `processor`:

````
init {
    setupProcess()
}

private fun setupProcess() {
    processor.videoSurfaceOutput.setFlipY(FLIP_FRAMES_VERTICALLY)
    // TODO
}
````

Then declare static variables according to the name of the `graph` used, these static variables will be used later to use the `graph`:

````
class FlutterHandTrackingPlugin(r: Registrar, id: Int) : PlatformView, MethodCallHandler {
    companion object {
        private const val BINARY_GRAPH_NAME = "handtrackinggpu.binarypb"
        private const val INPUT_VIDEO_STREAM_NAME = "input_video"
        private const val OUTPUT_VIDEO_STREAM_NAME = "output_video"
        private const val OUTPUT_HAND_PRESENCE_STREAM_NAME = "hand_presence"
        private const val OUTPUT_LANDMARKS_STREAM_NAME = "hand_landmarks"
    }
}
````

Now set up a `FrameProcessor` object, send the camera image frame converted by `converter` to the `MediaPipe graph` and run the graph to get the output image frame, then update the `previewDisplayView` to display the output. Add the following code to declare `FrameProcessor`:

``` kotlin
// Sends camera-preview frames into a MediaPipe graph for processing, and displays the processed
// frames onto a {@link Surface}.
private var processor: FrameProcessor = FrameProcessor(
        activity,
        eglManager.nativeContext,
        BINARY_GRAPH_NAME,
        INPUT_VIDEO_STREAM_NAME,
        OUTPUT_VIDEO_STREAM_NAME)
````

Then edit `onResume()` to set `convert` through `converter!!.setConsumer(processor)` to output the converted image frame to `processor`:

``` kotlin
private fun onResume() {
    converter = ExternalTextureConverter(eglManager.context)
    converter!!.setFlipY(FLIP_FRAMES_VERTICALLY)
    converter!!.setConsumer(processor)
    if (PermissionHelper.cameraPermissionsGranted(activity)) {
        startCamera()
    }
}
````

The next step is to output the image frame processed by the `processor` to the `previewDisplayView`. Re-edit the `setupPreviewDisplayView` and modify the previously defined `SurfaceHolder.Callback`:

``` kotlin
private fun setupPreviewDisplayView() {
    previewDisplayView.visibility = View.GONE
    previewDisplayView.holder.addCallback(
            object : SurfaceHolder.Callback {
                override fun surfaceCreated(holder: SurfaceHolder) {
                    processor.videoSurfaceOutput.setSurface(holder.surface)
                }

                override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {
                    // (Re-)Compute the ideal size of the camera-preview display (the area that the
                    // camera-preview frames get rendered onto, potentially with scaling and rotation)
                    // based on the size of the SurfaceView that contains the display.
                    val viewSize = Size(width, height)
                    val displaySize = cameraHelper!!.computeDisplaySizeFromViewSize(viewSize)
                    val isCameraRotated = cameraHelper!!.isCameraRotated
                    // Connect the converter to the camera-preview frames as its input (via
                    // previewFrameTexture), and configure the output width and height as the computed
                    // display size.
                    converter!!.setSurfaceTextureAndAttachToGLContext(
                            previewFrameTexture,
                            if (isCameraRotated) displaySize.height else displaySize.width,
                            if (isCameraRotated) displaySize.width else displaySize.height)
                }

                override fun surfaceDestroyed(holder: SurfaceHolder) {
                    processor.videoSurfaceOutput.setSurface(null)
                }
            })
}
````

When a `SurfaceHolder` is created, the camera image frames are converted via `convert` and output to a `processor`, which in turn outputs a `Surface` via `VideoSurfaceOutput`.

## Realize the communication between native components and `Flutter` through `EventChannel`

The previous processing is only processing image frames. In addition to processing images, `processor` can also obtain the coordinates of key points of the hand. Through `EventChannel`, these data can be transmitted to the `Dart` layer of `Flutter`, and then according to these The key point can be to write various business logic to convert gesture information into instruction information in real time: one, two, three, four, five, rock, spiderman... or write different special effects for different gestures. This brings a more natural and rich experience to human-computer interaction.

### Open `EventChannel` on `Android` platform

First define an `EventChannel` and an `EventChannel.EventSink`:

``` kotlin
private val eventChannel: EventChannel = EventChannel(r.messenger(), "$NAMESPACE/$id/landmarks")
private var eventSink: EventChannel.EventSink? = null
````

`EventChannel.EventSink` is used to send messages later. Then `eventChannel` is initialized in the `init` method:

``` kotlin
inti {
    this.eventChannel.setStreamHandler(landMarksStreamHandler())
}

private fun landMarksStreamHandler(): EventChannel.StreamHandler {
    return object : EventChannel.StreamHandler {

        override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
            eventSink = events
            // Log.e(TAG, "Listen Event Channel")
        }

        override fun onCancel(arguments: Any?) {
            eventSink = null
        }
    }
}
````

After setting up the message channel, edit the previous `setupProcess()` method. Add code before setting the output of `processor` to get the position of the hand key point and send it to the previously opened `eventChannel` through `EventChannel.EventSink` :

``` kotlin
private val uiThreadHandler: Handler = Handler(Looper.getMainLooper())

private fun setupProcess() {
    processor.videoSurfaceOutput.setFlipY(FLIP_FRAMES_VERTICALLY)
    processor.addPacketCallback(
            OUTPUT_HAND_PRESENCE_STREAM_NAME
    ) { packet: Packet ->
        val handPresence = PacketGetter.getBool(packet)
        if (!handPresence) Log.d(TAG, "[TS:" + packet.timestamp + "] Hand presence is false, no hands detected.")
    }
    processor.addPacketCallback(
            OUTPUT_LANDMARKS_STREAM_NAME
    ) { packet: Packet ->
        val landmarksRaw = PacketGetter.getProtoBytes(packet)
        if (eventSink == null) try {
            val landmarks = LandmarkProto.NormalizedLandmarkList.parseFrom(landmarksRaw)
            if (landmarks == null) {
                Log.d(TAG, "[TS:" + packet.timestamp + "] No hand landmarks.")
                return@addPacketCallback
            }
            // Note: If hand_presence is false, these landmarks are useless.
            Log.d(TAG, "[TS: ${packet.timestamp}] #Landmarks for hand: ${landmarks.landmarkCount}\n ${getLandmarksString(landmarks)}")
        } catch (e: InvalidProtocolBufferException) {
            Log.e(TAG, "Couldn't Exception received - $e")
            return@addPacketCallback
        }
        else uiThreadHandler.post { eventSink?.success(landmarksRaw) }
    }
}
````

Here `LandmarkProto.NormalizedLandmarkList.parseFrom()` is used to parse marker data in `byte array` format. Because all marker data is encapsulated with `protobuf`. `Protocol buffers` is a kind of `Google` can A tool for cross-platform serialization of structured data used in various languages. For details, you can view the [official website](https://developers.google.com/protocol-buffers).

In addition, `uiThreadHandler` is finally used to send data, because the `callback` of `processor` will be executed in the thread, but the `Flutter` framework needs to send messages to `eventChannel` in the `UI` thread, so use `uiThreadHandler` Come to `post`.

The complete details of `FlutterHandTrackingPlugin.kt` can be found at [github](https://github.com/zhouzaihang/flutter_hand_tracking_plugin)

### Get `eventChannel` data at `Dart` layer

Open `lib/flutter_hand_tracking_plugin.dart` again, edit the `HandTrackingViewController` class. Add an `EventChannel` according to the `id`, and then use `receiveBroadcastStream` to receive messages from this channel:

```` dart
class HandTrackingViewController {
  final MethodChannel _methodChannel;
  final EventChannel _eventChannel;

  HandTrackingViewController._(int id)
      : _methodChannel = MethodChannel("$NAMESPACE/$id"),
        _eventChannel = EventChannel("$NAMESPACE/$id/landmarks");

  Future<String> get platformVersion async =>
      await _methodChannel.invokeMethod("getPlatformVersion");

  Stream<NormalizedLandmarkList> get landMarksStream async* {
    yield* _eventChannel
        .receiveBroadcastStream()
        .map((buffer) => NormalizedLandmarkList.fromBuffer(buffer));
  }
}
````

As mentioned before, the transmitted data format is a `byte array` with a certain structure serialized by `protobuf`. So you need to use `NormalizedLandmarkList.fromBuffer()` to parse. `NormalizedLandmarkList.fromBuffer()` This interface , is provided by `protobuf` according to the `.dart` file generated by [protos/landmark.proto](https://github.com/zhouzaihang/flutter_hand_tracking_plugin/blob/master/protos/landmark.proto).

First open `pubspec.yaml` and add the dependencies of `protoc_plugin`:

```` yaml
dependencies:
  flutter:
    sdk: flutter
  protobuf: ^1.0.1
````

Then install and activate the plugin:

``` bash
pub install
pub global activate protoc_plugin
````

Then configure `Protocol` according to [protobuf installation tutorial](https://github.com/google/protobuf)

Then run the `protoc` command to generate the `.dart` file:

``` bash
protoc --dart_out=../lib/gen ./landmark.proto
````

You can also directly use the generated [flutter_hand_tracking_plugin/lib/gen/](https://github.com/zhouzaihang/flutter_hand_tracking_plugin/tree/master/lib/gen):

![ProtoBufGen](img/protobuf_gen.png)

After the generation is completed, the received data can be stored through `NormalizedLandmarkList`, and the `NormalizedLandmarkList` object has `fromBuffer()`, `fromJson()` various methods to deserialize the data.

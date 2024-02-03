---
description: >-
  Learn how to use dotnet-dsrouter and dotnet-trace to profile the CPU usage of
  your app
---

# Profiling Fabulous apps

.NET 6.0 introduced a new set of tools allowing to profile any .NET app very easily, without requiring paid or Windows-only tools. Since any Fabulous apps built with .NET MAUI requires at least .NET 7.0, we can make use of those exact same tools to profile our apps.

This page will focus on how to set up and profile an app on your machine and an actual Android and iOS devices. For more information, please read [Profiling .NET MAUI Apps](https://github.com/dotnet/maui/wiki/Profiling-.NET-MAUI-Apps).

### Installing dotnet-dsrouter and dotnet-trace

First thing, make sure you are using the latest version of .NET that you are targeting, ideally .NET 8.0 or higher.

Then we will need to install on our machine the 2 dotnet tools:

* **dotnet-trace**\
  Responsible for starting the profiler and generating the profiling report at the end of the execution.&#x20;
* **dotnet-dsrouter**\
  Proxy to redirect the output of a device so dotnet-trace can listen to it

```sh
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dsrouter
```

### Enabling the profiler for our app

In order to get the most representative data so we can do the appropriate optimizations, we want to profile in the most real-world conditions by deploying our app in release mode on an actual device.

To do that, first we need to plug the device to our computer and make sure we can deploy debug apps to it correctly. Also it is recommended to unplug any other devices and stop any emulator / simulator to guarantee we are working with the right device.

#### Android only : Configuring the profiler

Now that we are sure we can deploy the app on our device correctly, we need to configure our device to send the profiling data at the right place and also wait for us before starting the app we want to profile.

This is done by using the adb command line from the Android SDK.

```shell-session
adb shell setprop debug.mono.profile '127.0.0.1:9000,suspend'
```

This command line tells the profiler to output profiling data on 127.0.0.1:9000 (on the device) and as well as prevent the app from starting before we connect to it with a trace tool.

Like you could have guessed the profiler is output on the port 9000 on the device, except our trace tool is running on our computer. We need tell adb (from the Android SDK) to redirect the port 9000 on the device to the port 9001 on our computer.

```shell-session
adb reverse tcp:9000 tcp:9001
```

Once done, we can use the dotnet build command to deploy the release app to the device. The twist here is that we are going to add an additional flag to also enable the profiler, so we can later retrieve profiling data from the app.

#### Running the app in release mode with the profiler enabled

Everything is ready. We can compile the app for release mode and deploy it to the device.

```sh
dotnet build -f net8.0-android -c Release -t:Run -p:AndroidEnableProfiler=true
-p:AndroidSigningKeyStore=(...) -p:AndroidSigningStorePass=(...)
-p:AndroidSigningKeyAlias=(...) -p:AndroidSigningKeyAlias=(...)
```

A few things are happening in this single command line. Let's see each parameter individually:

* **dotnet build** : the command line to build the app
* **-f net8.0-android** : Build only for Android (we want to specifically target our device's OS)
* **-c Release** : Build in release mode (best for profiling)
* **-t:Run** : Tell the compiler to deploy and execute the app after compiling
* **-p:AndroidEnableProfiler** : Embed a profiler into our app that will send its data to a specific port on the device
* **-p:AndroidSigning(...)** : Use the specified keystore to sign the app so it can be deployed to the device

Once the command executes successfully, the app will automatically be deployed and launched to a white screen on our device. It is currently waiting for us to start the trace tool.

### Starting dotnet-dsrouter

We need to proxy the output of the device to our trace tool by using dotnet-dsrouter.

Simply execute the following command:

```sh
dotnet-dsrouter android -v debug
```

Keep this tool running as long as you need to profile your app. When starting, it will give you a process id (PID) that is required for starting dotnet-trace next.

### Starting dotnet-trace

Everything is now ready for us to actually profile the application.

Using the PID from previous step, we can attach our trace tool to the running app on the device.

```shell-session
dotnet-trace collect -p PID_FROM_PREVIOUS_STEP --format speedscope
-o ~/Desktop/profiling/04-string-literals
```

* **dotnet-trace collect** : Start the collection of profiling data
* **-p PID** : The process id we got from dotnet-dsrouter
* **--format speedscope** : Output the profiling data in the SpeedScope format (readable online on [https://speedscope.app](https://speedscope.app))
* **-o PATH** : SpeedScope trace file generated at the end of the profiling session (recommend give it a proper name to differentiate each profiling sessions later on)

When the command executes successfully, the application on the device will start and the UI will show up. Now, every code executed is being measured including the start time of the app and every interaction you do on the app.&#x20;

When you are done, stop dotnet-trace by pressing the Enter key in the terminal where the command line is running. dotnet-trace will generate the SpeedScope file that you can then upload to [https://speedscope.app](https://speedscope.app).

_Example of a Fabulous app profile file opened in SpeedScope_

<figure><img src="../../../.gitbook/assets/image (5).png" alt="Example of a Fabulous app trace on speedscope.app"><figcaption></figcaption></figure>

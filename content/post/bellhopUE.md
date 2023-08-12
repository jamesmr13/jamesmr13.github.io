---
title: "Bellhop as a Plugin in Unreal Engine 5.0"
date: 2023-08-11T15:06:35-04:00
draft: true
---

As part of my research at the University of Maryland I found myself in need of a way to simulate the arrival of passive SONAR signals. Unfortunately, there isn't too much in the way of open source passive SONAR simulators. I came across a Python Library called [arlpy](https://arlpy.readthedocs.io/en/latest/) and was introduced to the [Acoustic Toolbox](https://oalib-acoustics.org/models-and-software/acoustics-toolbox/) which included a program called Bellhop developed by Michael Porter.

Bellhop is a beam tracing model originally based upon the Gaussian beam method developed in the field of seismology by [Cerveny et. al.](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1365-246x.1982.tb06394.x). Porter first introduced the method's application for ocean acoustics in this [paper](https://pubs.aip.org/asa/jasa/article-abstract/82/4/1349/786154/Gaussian-beam-tracing-for-computing-ocean-acoustic?redirectedFrom=fulltext). Upon reading the [Bellhop user's guide](https://oalib-acoustics.org/website_resources/AcousticsToolbox/Bellhop-2010-1.pdf) I came across the section describing its ability to calculate broadband arrivals and realized I had found the software I was looking for. For a more in depth discussion of the mathematics involved in wave calculations, how ocean acoustics work and what the Bellhop progam does under the hood here are a few more resources:

* [Computational Ocean Acoustics 2nd ed. 2011](https://www.amazon.com/Computational-Acoustics-Modern-Signal-Processing/dp/1441986774)
* [Bellhop3D User Guide](https://usermanual.wiki/Document/Bellhop3D20User20Guide202016725.1524880335/html)
* [Ray Theory and Gaussian Beam Method for Geophysicists](http://www.cpgg.ufba.br/publicacoes/popov.pdf)

# How to use Bellhop in Unreal Engine?

Now that I had found this program I needed a way to utilize it. I had been experimenting with Unreal Engine for 2 semesters as a side project and my research advisor, Prof. Ramani Duraiswami, expressed interest in finding new ways to use Unreal Engine in his lab. There were a couple options I was aware of at this point. 

* Use the arlpy library with a SocketIO connection
* Write C++ scripts to execute the bellhop.exe file
* Write C++ scripts to execute bellhop functions inside MATLAB

These all seemed like they would work but I felt that requiring a .exe file for a plugin to function wasn't the best option. Additionally, my project was using another plugin for Machine Learning that already involved a SocketIO connection to Python so I wasn't sure how great performance would be with 2 plugins doing something like that. 

That is when I found the magnificent work of Jules Jaffe at the University of Callifornia Marine Physical Lab at Scripps Oceanography who had created a C++/CUDA port of the Bellhop and Bellhop3D underwater acoustic simulators under the name [bellhopcuda](https://github.com/A-New-BellHope/bellhopcuda). Firstly, I could obtain a .dll and .lib file to load into my custom plugin. Secondly, it made the File I/O normally required for Bellhop unnecessary. The port had numerous (and I mean numerous) custom structs to store the input and output data that I could directly access. This was a great relief as it meant I didn't have to parse output files myself and I could lower the demand on the program by not having it write to a file. Initially, I did still write a C++ script to generate the .env and .bty files since that was way more intuitive than all the structs I briefly mentioned before.

# Creating a Custom Plugin from a Third-Party C++ Library

This is a topic that brought me much frustration. Unreal Engine's documentation leaves much to be desired in a lot of ways. Many tutorials I found on how to accomplish this led to unexpected errors to solve. I finally had a lot of success upon finding this [blog post(http://www.valentinkraft.de/including-the-point-cloud-library-into-unreal-tutorial/)]. It was a little outdated (UE 4.16 vice UE 5.0) but combining that blog with the default files generated when you create a Third-Party plugin in Unreal Engine I was able to make it work. I wanted to write this blog partially because the example online of how to do this that weren't just GitHub repos are very few and far between.

## Setting Up Your File Structure

Whether you follow a similar file structure as the one in the above blog post isn't necessarily important but I found it to just make sense how it was set up and I definitely recommend sticking pretty close to it. I will outline it again here.

First things first you need to create a plugin. Importantly, you need to have either A. initalized your project as C++ vice Blueprint or B. created a GameModeBase class. If you chose A then when you go to create a plugin you will see all the available options (Blank, Third-Party, etc.). If you are working in a Blueprint project you only see the option for "Content-Only" that is no good and is what happened to me. The fix was simply creating a GameModeBase C++ class. You don't have to edit this class just generate it and then restart the Editor and you'll see all the plugin options (this is what fixed it for me in UE 5.0 if you are using a different engine version and/or see another issue I would advise searching the [UE wiki](https://unrealcommunity.wiki/)). I chose to create a ThirdPartyPlugin and edit it from there.

I created another module to handle the task of adding the include paths, runtime dependencies, and definitions calling it BHCLibrary adding it as a dependency module in the *Bellhop.Build.cs* file.

There were several files I needed my plugin to find namely,

* bellhopcxxlib.lib
* bellhopcxxlib.dll
* bhc.hpp
* math.hpp
* platform.hpp
* structs.hpp
* OpenGLMathematics Library (enormous library with far too many files in the include path to name)

I created the following file structures

* *Bellhop/Source/ThirdParty/BHCLibrary/lib* for the .lib file
* *Bellhop/Source/ThirdParty/BHCLibrary/include/bhc* for the header (.hpp) files specific to Bellhop
* *Bellhop/Source/ThirdParty/BHCLibrary/include/glm* for the OpenGL Mathematics Library

You could place the .dll file in the *lib* folder as well but I felt it more appropriate to place it in the following location

* *Bellhop/Binaries/ThirdParty/BHCLibrary/Win64/bellhopcxxlib.dll*

If you are reading this blog and hope to use a third-party library that also supports Mac and Linux this sort of file structure will be helpful for sorting your different files since .dll is Windows-specific.

## Using the Library

Now that your file structure is established it's time to load the library so you can use it. As mentioned in the previous section the file *BHCLibrary.Build.cs* will be telling the UnrealBuildTool what libraries and headers it needs. Like Valentin Kraft mentions in his blog you could do this in the *Bellhop.Build.cs* file but I agree with him that it is cleaner to do it in two separate files. Below is the *BHCLibrary.Build.cs* file

```cs
// Copyright Epic Games, Inc. All Rights Reserved.

using System;
using System.IO;

using UnrealBuildTool;

public class BHCLibrary : ModuleRules
{
    public BHCLibrary(ReadOnlyTargetRules Target) : base(Target)
    {
        // this tells UE that this module only imports assets from third parties
        Type = ModuleType.External; 

        // set define needed for using bellhopcuda when loaded from DLL (see bellhopcuda documentation)
        PublicDefinitions.Add("BHC_DLL_IMPORT=1");
        // add location of the .lib file 
        PublicAdditionalLibraries.Add(Path.Combine(ModuleDirectory, "lib", "bellhopcxxlib.lib"));
        // delay load the DLL
        PublicDelayLoadDLLs.Add("bellhopcxxlib.dll");
        // stage DLL
        RuntimeDependencies.Add("$(PluginDir)/Binaries/ThirdParty/BHCLibrary/Win64/bellhopcxxlib.dll");
        // add the include paths
        PublicIncludePaths.Add(Path.Combine(ModuelDirectory, "include"));
    }
}
```

Next, be sure to add the BHCLibrary module as a dependency in *Bellhop.Build.cs* as well as the Projects module (this is for using IPluginManager later on)

```cs
PublicDependecyModuleNames.AddRange(
    new string[]
    {
        "Core",
        "BHCLibrary",
        "Projects",
    }
)
```

Now that our *Build.cs* files are setup let's move onto our *Bellhop.h* and *Bellhop.cpp* files. The only thing you should need to add to your header file will be variables you plan on using so in my case my class looks like this

```cpp
public:
    virtual void StartupModule() override;
    virtual void ShutdownModule() override;
private:
    void* BHCLibraryHandle;
    FString BaseDir;
```

Then we override our *StartupModule* and *ShutdownModule* functions in *Bellhop.cpp*

```cpp
void FBellhopModule::StartupModule()
{
    // get base directory of the plugin
    BaseDir = IPluginManager::Get().FindPlugin("Bellhop")->GetBaseDir();
    // add relative location of the BHC DLL file
    FString BHCLibraryPath = 
        FPaths::Combine(*BaseDir, TEXT("Binaries/ThirdParty/BHCLibrary/Win64/bellhopcxxlib.dll"));
    // load the library if the DLL file is there
    BHCLibraryHandle = 
        !BHCLibraryPath.IsEmpty() ? FPlatformProcess::GetDllHandle(*BHCLibraryPath) : nullptr;
    // though obviously not necessary I definitely recommend logging the library loaded successfully 
    // and warning if it failed
    if (BHCLibraryHandle)
    {
        UE_LOG(LogTemp, Log, TEXT("BHC Library loaded successfully!"));
    }
    else
    {
        FMessageDialog::Open(EAppMsgType::Ok, LOCTEXT("ThirdPartyLibraryError", "Failed to load BHC Library"));
    }
}

void FBellhopModule::ShutdownModule()
{
    FPlatformProcess::FreeDllHandle(BHCLibraryHandle);
    BHCLibraryHandle = nullptr;
}
```

Now we can create classes within the bellhop plugin that will be able to use the functionality of the bellhop library.

## EXAMPLE: Blueprint Callable Functions

As an example I will show you how I created a blueprint callable function that uses bellhop to calculate the arrivals of 2 actors, a Receiver and a Source.

First, we have to create a blueprint library. Inside the Unreal Editor Tools -> New C++ Class -> Blueprint Function Library. Choose Public, name the class something like *BellhopBPLibrary* and to the right of the name select your plugin from the dropdown menu. This will generate a *.h* and *.cpp* file inside your plugin. I will walk through the example listed above. The specifics of the *CalculateArrivals* function will be of most use to people interested in bellhopcuda.

Now inside your header file declare all the functions you want to use. I use a combination of helper functions that stay hidden from Blueprints as well as the ones that can be called from Blueprints. For my example:

```cpp
class BELLHOP_API UBellhopBPLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()
    // declare function that can be called in Blueprints
    // note lack of ; in 1st line
    UFUNCTION(BlueprintCallable, Category = "Bellhop")
    static void CalculateArrivals(AActor* Receiver, AActor* Source);
    // declare a helper function that is called within CalculateArrivals
    // since I do not call UFUNCTION this function will not be available in blueprints
    static void GenerateEnvFile(AActor* Receiver, AActor* Source);
}
```

Importantly I (and Unreal Engine's documentation) recommend doing the following when importing header files from a third-party. They state that it prevents errors but I did not try it without the THIRD_PARTY_INCLUDES lines so I couldn't tell you what possible errors you could potentially see.

```cpp
THIRD_PARTY_INCLUDES_START
#include <bhc/bhc.hpp>
#include <bhc/structs.hpp>
#include <bhc/math.hpp>
#include <bhc/platform.hpp>
THIRD_PARTY_INCLUDES_END
```


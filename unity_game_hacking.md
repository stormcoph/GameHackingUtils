# Base for hacking unity games (that doesn't use IL2CPP)

## Creating The Project

1. create new project -> Class Library (.NET Framework) -> .NET Framework 4.7.2

2. create a Loader.cs class, should look something like this:
```csharp
using UnityEngine;

namespace Namespace
{
    public class Loader
    {
        private static GameObject _gameObject;
        public static void Load()
        {
            _gameObject = new GameObject();

            _gameObject.AddComponent<Main>();

            UnityEngine.Object.DontDestroyOnLoad(_gameObject);
        }
        public static void Unload()
        {
            if (_gameObject != null)
            {
                UnityEngine.Object.Destroy(_gameObject);
            }
        }
    }
}
```

3. create Main.cs class, this is where the cheat will actually be, e.g:
```csharp
using System;
using UnityEngine;

namespace Namespace
{
    public class Main : MonoBehaviour
    {
        void OnGUI()
        {
            GUI.color = Color.red;
            Rect labelRect = new Rect(10, 10, 300, 40);

            GUI.Label(labelRect, "cheat loaded");

            GUI.color = Color.white;
        }
    }
}
```

## adding the DLL And fixing the stupid annoying error

1. in the first `<PropertyGroup>` at the bottom line, add `"<NoStdLib>true</NoStdLib>"`

2. add a new Property Group with the path to all of the game DLL's <-> DO NOT PUT IT INSIDE OF ANOTHER `<PropertyGroup>` IT NEEDS TO BE SEPERATE!
```xml
  <PropertyGroup>
    <GameManagedPath>C:\Program Files (x86)\Steam\steamapps\common\ROUNDS\ROUNDS_Data\Managed</GameManagedPath>
  </PropertyGroup>
```

3. add the game DLL's by right clicking "References" in Visual Studio -> Add References -> Browse -> go to all of the game dlls and add all of them -> in the list, select all of the unity DLL's and the Assembly-CSharp add them

4. IF THAT STILL DOESNT WORK, add it manually with the other dll's, e.g:
```xml
<ItemGroup>
  <!-- THIS IS THE NEW CRITICAL REFERENCE -->
  <Reference Include="mscorlib">
    <HintPath>C:\Program Files (x86)\Steam\steamapps\common\ROUNDS\ROUNDS_Data\Managed\mscorlib.dll</HintPath>
  </Reference>
  <!-- It is better to use the specific modules for Unity than the old monolithic UnityEngine.dll -->
  <Reference Include="UnityEngine.CoreModule">
    <HintPath>C:\Program Files (x86)\Steam\steamapps\common\ROUNDS\ROUNDS_Data\Managed\UnityEngine.CoreModule.dll</HintPath>
  </Reference>
  <Reference Include="UnityEngine">
    <HintPath>C:\Program Files (x86)\Steam\steamapps\common\ROUNDS\ROUNDS_Data\Managed\UnityEngine.dll</HintPath>
  </Reference>
  <Reference Include="Assembly-CSharp">
    <HintPath>C:\Program Files (x86)\Steam\steamapps\common\ROUNDS\ROUNDS_Data\Managed\Assembly-CSharp.dll</HintPath>
  </Reference>
  <!-- ... You should add other Unity DLLs here as needed (e.g., UnityEngine.IMGUIModule) -->
</ItemGroup>
```

## Mono Injector (github.com/wh0am15533/SharpMonoInjector/tree/master)

1. launch smi_gui.exe

2. Inject
- Process -> Refresh until game shows up
- Assembly to inject -> choose your DLL from your cheat
- Namespace -> should be auto detected, otherwise idk
- Class name -> add the name of your loader class, if following example its just "Loader"
- Method name -> add the name of your init / load method in your Loader class, if following example its "Load"

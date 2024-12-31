---
description: How to install dependencies to Geode mods
---

# Dependencies

Geode provides utilities for mods to depend on other mods, to make sharing code easy.

Note that Geode **only manages dependencies that are mods**. Normal C++ dependencies, like a JSON parsing library, ZIP library, or some networking utilities, **should just be installed like they are in any other C++ project**. Use CPM, Git submodules, copy the code to your project, whatever you prefer. If the library is dynamic, include the `.dll` with your mod through the `files` key in `resources`. 

However, sometimes you want to depend on code that can't just be used as a normal library; for example, **custom keybinds**. Geode does not provide any custom keybinds utilities out-of-the-box, so you need to use a library. However, it would not make much sense if every mod bundled their own incompatible systems for dealing with custom keybinds. Instead, there should be one mod that just adds the functionality, and then other mods can depend on that mod and call its functions to add keybinds.

## How dependencies work

Dependency mods are like any other mods, except that they **include their headers and linkable files in their .geode package**. Usually the headers are located in an `include` directory inside the package, but they may also be anywhere. The linkable file is `mod.id.lib` on Windows and the normal mod binary on other platforms, located at the root of the package.

Otherwise, dependency mods are just like normal mods; they may place hooks, patches, add features to the game, and be published on the mods index. However, **dependency mods should keep the features they add to a minimum**, and be focused on the specific features they're meant to add. For example, a custom keybinds dependency should only add the necessities for working with custom keybinds; it shouldn't also add a bunch of other features, like adding more icons or customizing menus.

If a dependency is required, it is **linked to**; this means that for the mod that depends on it to run, the dependency must be present. However, as sometimes you may want to only use a dependency if it is loaded, **dependencies may also be marked optional**. In this case, the dependency is not linked to, which means that you can't use any of the dependency's functions, but have to use Geode's functions for working with optional dependencies. See [Optional dependencies](#optional-dependencies) for more info.

## Adding dependencies

Dependencies can be added to your mod by simply adding it to the `dependencies` key in [mod.json](/mods/configuring.md):

```json
{
    "geode": "v1.0.0",
    "id": "my.example-mod",
    "name": "My example mod",
    "developer": "Me",
    "version": "v1.0.0",
    "dependencies": [
        {
            "id": "someone-elses.mod",
            "version": ">=v1.2.5",
            "importance": "required"
        }
    ]
}
```

Dependencies have two required properties: the ID of the dependency and the version depended on. Additionally, the dependency may have an [importance](#importance), which specifies if the dependency is required or not.

The `version` field of a dependency may be written as `>=version`, `=version`, or `<=version`. The comparisons work as expected, with the addition that if the major versions are different, the comparison is always false. This means that if you depend on version `>=1.2.5` of a mod, version `v1.8.0` will be considered acceptable but `v2.0.0` will not. For this reason, [if you make a mod that is depended upon, you should follow strict semver](https://semver.org).

Once you have added a dependency to your `mod.json`, if you have [Geode CLI v1.4.0 or higher](/geode/installcli), it's headers are automatically added to your project. If you have the mod installed, the headers from the installed version will be used. If you don't have the mod installed, Geode will install it from the mods index. The added dependency files for your project can be found in `build/geode-deps/<dep.id>`. Geode automatically add `build/geode-deps` to your project's include path, and links whatever binaries are found in dependencies, meaning you most likely don't have to configure anything.

## Example

The mod `hjfod.gmd-api` contains utilities for working with [.GMD files](https://fileinfo.com/extension/gmd). You can add it to your mod by adding the following to your `mod.json`:

```json
{
    "dependencies": [
        {
            "id": "hjfod.gmd-api",
            "version": ">=v1.0.0"
        }
    ]
}
```

Now, if you reconfigure your CMake (or if you're using CMake Tools in VS Code, it should be reconfigured automatically), Geode should automatically find `hjfod.gmd-api` in your installed mods. If the mods index, and install it for you.

```cpp
#include <hjfod.gmd-api/include/GMD.hpp>
// do things with gmd-api's functions
```

If you now go to compile and test your mod, everything should work out-of-the-box.

## Importance

> Possible values:
> - `required` (default) - Dependency must be installed for this mod to work
> - `recommended` - Dependency is not required, but is recommended and will be downloaded by default.
> - `suggested` - Dependency is not required, and will not be downloaded by default.

In case the dependency is **not required**, it is not linked to, which in turn means that you can't use any of its exported functions. In cases like these, the dependency should provide ways through Geode to dynamically call its functions, such as through events.

## Events

The key system Geode provides for optional mod interop are events. For example, a mod that adds support for drag-and-dropping files on the GD window could define a drag-and-drop event that other mods can then listen to.

Usually however, events are defined in code in a way that requires linking by inheriting from the `Event` class. To avoid this, mods that want to support being used optionally should also provide events that are specializations of the `DispatchEvent` class:

```cpp
#include <Geode/loader/Dispatch.hpp>
using ExampleEvent = geode::DispatchEvent<type>;

template <class>
struct ToFilterImpl;

template <class... Args>
struct ToFilterImpl<geode::DispatchEvent<Args...>> {
	using type = geode::DispatchFilter <Args...>;
};

template <class T>
using ToFilter = typename ToFilterImpl<T>::type;

$execute {
    new EventListener(+[](type name) {
        code...
        return ListenerResult::Stop;
    }, ToFilter<ExampleEvent>("mod.id/EventName));
}
```

All `DispatchEvent`s have an associated ID, which is specific for each `DispatchEvent` specialization. This can be used to differentiate between events; for example, a drag drop API might use this to let dependencies determine which file types they listen to.

Mods that wish to use the dependency can now post events:

```cpp
#include<Geode/loader/Dispatch.hpp>
DispatchEvent<type>("mod.id/EventName", value).post();
```

An example of using dispatch events in practice [can be found in Texture Loader](https://github.com/geode-sdk/textureldr/commit/578d855a63a55525ab5ef29ce492a5eaf232fe4f). Look at line 28 of include/TextureLoader.hpp, and lines 35-40 of src/API.cpp on the linked commit version (this might change in future versions). 

## Attributes

One way for mods to communicate with optional dependencies is through **node attributes**, which may contain any data. These are like the `setUserData` and `setUserObject` functions native to `CCNode`s, except that attributes have a string key associated with them. For example, a mod that adds scrollbars to layers might use the following check to see if a scrollbar should be added to a layer:

```cpp
if (layer->getAttribute<bool>("hjfod.cool-scrollbars/enable")) {
    // add scrollbar
}
```

Other mods can set this attribute on their layers with the `CCNode::setAttribute` function.

Mods can also add an event listener to listen for when attributes are added/changed:

```cpp
$execute {
    new EventListener<AttributeSetFilter>(
        +[](AttributeSetEvent* event) {
            addScrollbar(event->node);
        },
        AttributeSetFilter("hjfod.cool-scrollbars/enable")
    );
}
```

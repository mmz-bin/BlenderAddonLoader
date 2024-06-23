This library uses English that has been machine-translated from Japanese.

__[日本語のreadmeはこちらから](README.ja.md)__

# BlenderAddonLoader
This script allows for the dynamic registration and unregistration of Blender addons. It automates the tedious tasks of registering, unregistering, disabling, prioritizing classes, and registering shortcut keys. Verified to work with Blender 4.1.

__Note: The three files inside the core directory (addon_register.py, shortcuts_register.py, proc_loader.py) should be placed in the same directory.__

## Features
- Registers and unregisters all addon classes within a specified directory, including subdirectories.
- Define a list named `ignore` in the `__init__.py` of each directory to specify module names that should be ignored.
    - The module path is relative to the directory where the `__init__.py` file defining the list is located.
        - Example (in the `__init__.py` file of the `operators` folder): `ignore = ['your_operator']`
- Use the [`disable`](#proc_loaderpy) decorator to ignore specific classes.
    - Example: `@disable`
- Use the [`priority`](#proc_loaderpy) decorator to control the loading order of specific classes.
    - Example: `@priority(42)`
- Use the [`ShortcutsRegister`](#shortcuts_registerpy) class to register shortcut keys.

In this readme, the sample code is written with the following directory structure:
```
.
└── your_addon_folder/
    ├── __init__.py
    ├── core/
    │   ├── addon_register.py
    │   ├── shortcuts_register.py
    │   └── proc_loader.py
    ├── operators/
    │   ├── __init__.py
    │   └── your_operator.py
    └── panels/
        └── your_panel.py
```

## addon_register.py
- __AddonRegister__ class
  - This is the central class for addon registration.
    - The first argument of the constructor should be the addon folder (usually the `__file__` variable in the `__init__.py` file), and the second argument should be a list of folder names containing the respective functionalities.
    - The third and fourth arguments are optional; they automatically register and unregister by passing the module name and Blender's standard translation table.
        - The third argument is the module name (usually the `__name__` variable in the `__init__.py` file), and the fourth argument is the translation table dictionary.
    - Create an instance in the `__init__.py` file and wrap the `register()` and `unregister()` functions.
        - The `__init__.py` file in the root directory is a sample.
    - Example
    ```
        addon = AddonRegister(__file__, ['operators', 'panels']) # Instance creation
        def register() -> None: addon.register()                 # Wrap register() function
        def unregister() -> None: addon.unregister()             # Wrap unregister() function
    ```
    - Example of registering a translation dictionary
    ```
    # Definition of translation table
    translation_dict = {
        "en_US": { ('*', 'hoge') : 'English' },
        "ja_JP" : { ('*', 'hoge') : '日本語' }
    }

    addon = AddonRegister(__file__, ['operators', 'panels'], __name__, translation_dict) # Instance creation
    def register() -> None: addon.register()                                             # Wrap register() function
    def unregister() -> None: addon.unregister()                                         # Wrap unregister() function
    ```

## shortcuts_register.py
- __Key__ class
    - This is a data class for storing shortcut key information.
        - `bl_idname`: The bl_idname of the operator the shortcut key targets
        - `key`: The shortcut key

        __The following attributes are optional.__

        - `key_modifier`: Additional shortcut key (default is `'NONE'`)
        - `trigger`: The action of the trigger key (default is `'PRESS'`)

            _Settings for special keys (keys pressed simultaneously)_

        - `any`: Any special key (default is `False`)
        - `shift`: Shift key (default is `False`)
        - `ctrl`: Control key (default is `False`)
        - `alt`: Alt key (default is `False`)
        - `oskey`: OS key (default is `False`)
    - Example: `Key(HOGE_OT_YourOperator.bl_idname, 'A')`

- __ShortcutsRegister__ class
    - This class registers shortcut keys.
    - It adopts the singleton pattern.

    **`add(keys, name, space_type, region_type, modal, tool) -> List[tuple[KeyMap, KeyMapItem]]` method**

    - Registers one or multiple shortcut keys and returns a list of registered keymaps and items.
    - Arguments
        - `keys`
            - One or multiple Key objects
            - If multiple, pass them as a list

        _The following arguments are optional and correspond to the arguments of the context.window_manager.keyconfigs.addon.keymaps.new() function._

        - `name`: Identifier for the shortcut (default is `'Window'`)
        - `space_type`: Specifies the area where the shortcut key operates (default is `'EMPTY'`)
        - `region_type`: Specifies the range where the shortcut key operates (default is `'WINDOW'`)
        - `modal`: Specifies whether it is in modal mode (default is `False`)
        - `tool`: Specifies whether it is in tool mode (default is `False`)

    - Example: `ShortcutsRegister().add(Key(HOGE_OT_YourOperator.bl_idname, 'A'))`

    **`delete(km, kmi)` method**

    - Takes a keymap and keymap item to delete the shortcut key.

    - Example: `ShortcutsRegister().delete(km, kmi)`

    **`unregister()` method**

    - Deletes all shortcut keys.
    - It is automatically called within the unregister() method of the AddonRegister class, so it usually does not need to be explicitly called.

    - Example: `ShortcutsRegister().unregister()`

## proc_loader.py
- __DuplicateAttributeError__ class
    - This is an exception class raised when an attribute used by the decorator already exists in the target class.

- __disable__ decorator
    - A class with this decorator will be ignored and not loaded by ProcLoader.
    - The target class must not have the `addon_proc_disabled` attribute.
    - Example
        ```
        @disable
        class HOGE_OT_YourOperator(bpy.types.Operator): pass
        ```
- __priority__ decorator
    - This decorator allows you to control the loading order of classes by specifying a number.
    - The number must be 0 or higher, and the smaller the number, the earlier it is loaded.
    - If this decorator is not attached or a value of 0 or less is specified, it will be loaded last.
    - If this decorator is not attached or the same number is specified, the loading order is not guaranteed.
    - The target class must not have the `addon_proc_priority` attribute.
    - Example
        ```
        @priority(42)
        class HOGE_OT_YourOperator(bpy.types.Operator): pass
        ```

- __ProcLoader__ class
    - This class loads addon files and registers them with Blender.
    - It retrieves all modules below the specified folder, including subfolders.
    - Define a list named `ignore` in the `__init__.py` of each directory and list the module names to ignore those modules.
        - The module path is a relative path from the directory where the `__init__.py` file defining the list is located.
            - Example (`__init__.py` file in the `operators` folder): `ignore = ['your_operator']`
    - There are no restrictions on the number of modules or classes.
    - It is usually manipulated by the AddonRegister class, so there is no need to explicitly manipulate it.

    - **`__init__(path, target_classes)` method**
        - Arguments
            - `path`: Absolute path to the addon (usually the `__file__` variable in the addon's `__init__.py` file)
            - `target_classes` (optional): Specifies the target classes to load.
                - If not specified, the `Operator`, `Panel`, `Menu`, `Preferences`, and `PropertyGroup` classes under bpy.types will be targeted.
        - Example: `pl = ProcLoader(__file__)`

    - **`load(dirs) -> List[Sequence[Union[ModuleType, object]]]` method**
        - Returns a list of loaded modules and classes.
        - Argument: `dirs`: Specifies the directories to load.
        - Example: `modules, classes = pl.load(['operators', 'panels'])`

    - **`load_files(dirs) -> List[str]` method**
        - Obtains the module paths in the form of `[addon name].[folder name].[file name]`.
        - Excludes modules specified in the `ignore` list.
        - Argument: `dirs`: Specifies the directories to load.
        - Example: `modules_path = pl.load_files(['operators', 'panels'])`

    - **`load_modules(paths) -> List[ModuleType]` method**
        - Imports modules based on the given paths.
        - Argument: `paths`: Specifies the paths to the modules to load.
        - Example: `modules = pl.load_module(module_path)`

    - **`load_classes(modules) -> List[object]` method**
        - Loads the classes within the given modules.
        - Sorts the class objects based on the `disable` and `priority` decorators.
        - Argument: `modules`: Specifies the target modules.
        - Example: `classes = pl.load_classes(modules)`

### Sample

`__init__.py`
```
from .core.register_addon import AddonRegister

bl_info = {
    "name": "Addon_name",
    "author": "your_name",
    "version": (1, 0, 0),
    "blender": (4, 1, 0),
    "location": "View3D > Tools > Addon_name",
    "description": "",
    "category": "General",
}

addon = AddonRegister(__file__, [
    'operators',
    'panels'
])

def register() -> None: addon.register()

def unregister() ->

 None: addon.unregister()

if __name__ == '__main__':
    register()
```
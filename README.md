# Minimal Eiffel Inkview
Minimal "Hello, World"-like demo of using Eiffel with PocketBook Inkview library.

Compiled and tested with EiffelStudio 19.05 GPL Edition.

## Requirements
* PocketBook EiffelStudio spec from https://github.com/imustafin/estudio-cross-configs
* PocketBook SDK. This demo was tested with https://github.com/pocketbook/SDK_6.3.0/tree/5.19 SDK-B288

## Step-by-Step Guide

### 1. Installing Requirements
1. Install the PocketBook EiffelStudio spec
2. Install PocketBook SDK
3. Configure environment variables for the spec
4. Run EiffelStudio with environment variables set up

### 2. Configuring a New Project for the Spec
In EiffelStudio:

1. Click File > New Project > Basic application (no graphics library included) > Create
2. Disable Concurrency checkbox. Fill other settings as you want. Click OK.
3. The project will not compile because there is no precompile in the PocketBook spec.
  Remove the default precompile from the project:
  1. Click Project > Project Settings > Target: "your target name" > Groups
  2. Remove the default precompile
  3. Close the Project Settings menu
4. Now you can compile the project (Project > Compile)

### 3. Running on the PocketBook
Executables should have the `.app` filename extension
and be placed in the `applications`.

#### Finalized Compilation

By default, finalized compilation (Project > Finalize) produces the executable
at the `EIFGENs/"your target name"/F_code/"your target name"` location.

You can copy the executable to the `applications` folder and add the `.app` extension.
The application should now appear in the device's Applications menu.

#### Workbench Compilation
Workbench compilations (Project > Compile) produce not only the executable file
(`EIFGENs/"your target name"/W_code/"your target name"`) 
but also the `"your target name".melted` bytecode file near the executable. 

Both files need to be present on the e-reader. You can put them
in the `applications` folder. The executable should have the `.app` filename
extension. The bytecode file should not be renamed and should keep the `.melted`
extension.

After that the application should appear in the device's Applications menu.


#### How to Run Console Applications
Running a console application from the menu will not draw anything to screen.
To see program's output you can use various terminal applications such as
pbterm from https://userpage.physik.fu-berlin.de/~jtt/PB/.

In the terminal program navigate to the `applications` directory. Depending on
the firmware version the location may vary. In recent firmwares the command
would be:
```bash
cd /mnt/ext1/application
```

From the `applications` folder you can run the executable:
```bash
"your target name".app
```

And you shuold see the output.

### 4. Linking Inkview
1. Click Project > Project Settings > Advanced > Externals
2. Click "Add Include" and specify the "Location" of the `freetype2` includes.
  Most probably it is at `SDK_6.3.0-5.19/SDK-B288/usr/arm-obreey-linux-gnueabi/sysroot/usr/include/freetype2`
3. Click "Add Include" and specify the "Location" of the `inkview` includes.
  Most probably it is at `SDK_6.3.0-5.19/SDK-B288/usr/arm-obreey-linux-gnueabi/sysroot/usr/local/include`
4. Click "Add Library" and specify the location of `libinkview.so`.
  Most probably it is at `SDK_6.3.0-5.19/SDK-B288/usr/arm-obreey-linux-gnueabi/sysroot/usr/local/lib/libinkview.so`
5. Now you can use `"inkview.h"` in `external` features and it will compile:
    ```eiffel
    make
            -- Run application.
        do
            print ("Hello, world!%N")
            clear_screen -- Will segfault here, read below
            full_update
        end

    clear_screen
            -- Fill canvas with white color
        external
            "C use %"inkview.h%""
        alias
            "ClearScreen"
        end

    full_update
            -- Draw the canvas on the actual display
        external
            "C use %"inkview.h%""
        alias
            "FullUpdate"
        end
    ```
  
**This will segfault** in `clear_screen` because it seems that
`ClearScreen` and `FullUpdate` require some resources which should be opened.
Usually this is done by Inkview itself in the `InkViewMain` function. Read further
for an example of how this can be achieved.

### 5. Registering the Event Handler in InkViewMain
The usual way of writing applications with Inkview is by registering
an event handler through `InkViewMain(iv_handler h)`.

Event handler is a function pointer with this signature:
```c
typedef int  (*iv_handler)(int type, int par1, int par2);
```

Eiffel has `agent`s but not function pointers. Here I present the least complex
way to wrap an Eiffel feature and to send it to C.

1. Create a new header file in a directory. For example, `include/iv_handler_wrapper.h`.
2. Add this include to the project
  (Project > Project Settings > Advanced > Externals > Add Include)
3. Add the following to the header:
    ```c
    #ifndef IV_HANDLER_WRAPPER_H___
    #define IV_HANDLER_WRAPPER_H___

    #include <eif_portable.h>

    // Call target for iv_handler_eiffel_feature
    static void* iv_handler_object;

    // Eiffel feature of iv_handler_object
    // which implements Inkview's iv_handler
    static int (*iv_handler_eiffel_feature) (void *a_class, int type, int par1, int par2);

    // Delegates calls to iv_handler_object's iv_handler_eiffel_feature feature
    static inline int call_iv_handler(int type, int par1, int par2) {
        if (iv_handler_eiffel_feature && iv_handler_object) {
            return iv_handler_eiffel_feature(
                eif_access(iv_handler_object),
                type,
                par1,
                par2
            );
        } else {
            printf("No handler feature/object (%p, %p)\n",
                (void *) iv_handler_eiffel_feature,
                (void *) iv_handler_object
            );
            return 0;
        }
    }

    #endif
    ```
4. Add the Eiffel part:
    ```eiffel
    set_inkview_main (a_target, a_feature: POINTER)
            -- `a_target` - target to call `a_feature`
            -- `a_feature` - pointer to an `(INTEGER, INTEGER, INTEGER): INTEGER` function
            --    callable on `a_target`
        require
            not a_target.is_default_pointer
            not a_feature.is_default_pointer
        external
            "C inline use %"inkview.h%", %"base_pb.h%""
        alias
            "[
                iv_handler_object = eif_protect($a_target);
                iv_handler_eiffel_feature = $a_feature;

                InkViewMain(call_iv_handler);
            ]"
        end
    ```
5. Use it like this:
    ```eiffel
    make
            -- Run application.
        do
            print ("Hello, world!%N")
            set_inkview_main ($Current, $handle_event)
        end

    handle_event (a_type, a_par1, a_par2: INTEGER): INTEGER
        do
            print ("Event: " + a_type.out + ", " + a_par1.out + ", " + a_par2.out + "%N")
            if a_type = Evt_show then
                clear_screen
                fill_area (100, 200, 300, 400, Black)
                full_update
            elseif a_type = Evt_keydown then
                close_app
            end
        end
    ```


TODO: discuss why it is so complex

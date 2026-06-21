GodotGetImage plugin for Godot 4.2+ (Photo Picker fork)
=======================================================
_______________________________________________________


Android plugin for Godot 4.2+.
Pick one or more images from gallery (via the Android Photo Picker) or capture an
image from the camera.

> **This fork uses the Android Photo Picker and declares no storage read
> permissions, to comply with Google Play's
> [Photo and Video Permissions](https://support.google.com/googleplay/android-developer/answer/14115180)
> policy.** Apps that only need one-off / transactional image selection (e.g. the
> user picks a single photo) are rejected by Google Play when they request the
> restricted `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` permissions. The Photo
> Picker needs none of them.

What is the Android Photo Picker?
---------------------------------

The Android Photo Picker is a system UI for choosing photos and videos, provided
by the OS (Android 13+) and backported to Android 4.4+ through Google Play
Services. The user browses their images inside this Google-provided picker and
hands your app only the item(s) they explicitly select, so your app never gets
access to the rest of the gallery. Because access is granted per picked item,
**no storage permission is required**. That is exactly why it satisfies Google
Play's policy for apps that only pick an occasional image (such as a jigsaw
puzzle game where the user picks one photo). On devices below Android 13 without
the backport, the plugin falls back to the Storage Access Framework document
picker, which is also permission-free.

See demo project [`plugin/demo/`](plugin/demo/) (Godot 4.6.2).

**_NOTE:_** Starting in Godot 4.2, Android plugins built on the v1 architecture are now deprecated. Instead, Godot 4.2 introduces a new Version 2 (v2) architecture for Android plugins. This plugin is from now on build on Godot Plugin v2 system.

More information about v2 architecture: [official documentation](https://docs.godotengine.org/en/stable/tutorials/platform/android/android_plugin.html "documentation")

What changed vs upstream
========================

- **Removed** `READ_MEDIA_IMAGES` and `READ_EXTERNAL_STORAGE` from the plugin
  manifest. They no longer merge into your app and no longer trigger Play Store
  rejections.
- **Gallery selection now uses the system Photo Picker** (`ACTION_PICK_IMAGES`,
  API 33+) with an `ACTION_OPEN_DOCUMENT` (Storage Access Framework) fallback on
  older devices. Both are permission-free. The legacy `ACTION_PICK` +
  runtime-permission flow is gone.
- Multi-select uses the device's maximum allowed count
  (`MediaStore.getPickImagesMaxLimit()`).
- `android.hardware.camera` is now `required="false"`, so devices without a
  camera can still install the app. The `CAMERA` permission is unchanged.
- **Added** a `hasCamera()` runtime check.

The GDScript API is otherwise unchanged, so existing game code keeps working
with the same method names, options, and signals.

Upgrade from Godot plugin v1 to v2 architecture
===============================================

1. Remove GodotGetImageRelease.arr and GodotGetImage.gdap in the folder: *[you project]/android/plugin*

2. Follow installation instructions below

Installation
============

1. If exists, unzip the precompiled release zip from the `release/` folder into your project's `addons/` folder:
*release/godotgetimageplugin_photopicker_v[version]_for_godot_[your Godot version].zip* -> *[your godot project]/addons/*
(e.g. `godotgetimageplugin_photopicker_v0.3.0_for_godot_4.6.2.zip` for Godot 4.6.2)

2. Activate plugin in Godot by enable Navigate to `Project` -> `Project Settings...` -> `Plugins`, and ensure the plugin "GodotGetImage" is enabled.

Build plugin .aar file
----------------------

If there is no GodotGetImage release for your Godot version, you need to generate new plugin .aar file.

1. Set correct Godot version by edit the gradle file [`plugin/build.gradle.kts`](plugin/build.gradle.kts):

```
dependencies {
    // Update this to match your Godot engine version
    implementation("org.godotengine:godot:4.6.2.stable")
```

2. Compile the project:

	Open command window and *cd* into plugin root directory and run command below

	* Windows:

		gradlew.bat assemble

	* Linux:

		./gradlew assemble

3. On successful completion of the build, the output files can be found in
  [`plugin/demo/addons/GodotGetImage`](plugin/demo/addons/GodotGetImage). Copy that
  `addons/GodotGetImage` folder into your game project's `addons/`.

Old-device behavior
-------------------

The gallery picker works without any permission on every supported API level:

- **API 33+**: the system Android Photo Picker (`ACTION_PICK_IMAGES`).
- **API 21 to 32**: the Storage Access Framework document picker
  (`ACTION_OPEN_DOCUMENT`).

On devices with Google Play Services, the modern Photo Picker UI is additionally
backported to older releases. Either way, no storage permission is requested.

# Plugin API

It is preferable to set the image size to the maximum desired size before any image requests. This minimize the risk of getting "out of memory" when loading image with unknown sizes.

**_NOTE:_** "idle_frame" (or "process_frame" as it was called in Godot 4) is NOT necessary in Godot 4.x

Quick example
-------------

Pick one image from the gallery and show it on a `Sprite2D`:

```gdscript
extends Node2D

var plugin
@onready var sprite: Sprite2D = $Sprite2D

func _ready():
	if Engine.has_singleton("GodotGetImage"):
		plugin = Engine.get_singleton("GodotGetImage")
		plugin.connect("image_request_completed", _on_image_request_completed)

func _on_button_pressed():
	# Opens the permission-free Photo Picker
	if plugin:
		plugin.getGalleryImage()

func _on_image_request_completed(images: Dictionary):
	# A single pick returns one entry, so use the first image buffer
	for buffer in images.values():
		var image := Image.new()
		if image.load_jpg_from_buffer(buffer) == OK:
			sprite.texture = ImageTexture.create_from_image(image)
		break
```

Permissions
-----------

Gallery selection requires **no permission**, so do not add any storage
permission to your export preset.

The only permission this plugin uses is *Camera*, and only for `getCameraImage()`.
The plugin requests it at runtime when needed.

Methods
-------

***getGalleryImage()***
Select one image from gallery (permission-free Photo Picker)

***getGalleryImages()***
Select multiple images from gallery (permission-free Photo Picker)

***getCameraImage()***
Capture image from camera

***hasCamera()***
Returns `true` if the device has any camera. Useful because the camera feature is
declared optional (`required="false"`); check this before showing a camera button.

***resendPermission()***
If user has declined permission this needs to be called for a new permission is requested.

***setOptions(*** *Dictionary with options* ***)***
If you would like a specific size of the images, it can set via this feature.
This will apply to all images until options are set again or ***setOptions(*** *{ }* ***)*** is called with an empty dictionary.

*This will not make the image larger than the original, in which case the original size will be kept.*

#### Available options:
* *"image_height"* (Int): Sets maximum image height
* *"image_width"* (Int): Sets maximum image width
* *"keep_aspect"* (Bool): Keep aspect ratio or not
* *"image_quality"* (Int 0-100): Sets image compression quality, default is 90. 100 is best quality with least compression.
* *"image_format"* (String): Set the compression format returned by plugin (supported formats: *"jpg"* , *"png"*). Default "jpg".
* *"auto_rotate_image"* (Bool): Plugin will try to set correct orientation of the image. This is not 100% but will mostly return a correct oriented camera image.
* *"use_front_camera"* (Bool): Plugin will try to use front camera.

**_NOTE:_** Remember to load correct format in your code: ***load_jpg_from_buffer()*** or ***load_png_from_buffer()***

```python
e.g.
dict = {
	"image_height" : 1000,
	"image_width" : 600,
	"keep_aspect" : true,
	"auto_rotate_image" : true
}
or
dict = {
	"image_quality" : 40,
	"image_format" : "png"
}
```



Emitted signals
---------------

***image_request_completed***
Returns a Dictionary with images as PackedByteArray

**_NOTE:_** If non supported image is selected this will return null value

***permission_not_granted_by_user***
User declines Android permission request (camera only).
It's a good practice to explain why permission is important and then call resendPermission()

***error***
Returns any error as string

Migration note for existing users
=================================

The GDScript API is unchanged, so you do not need to touch your game code.
Replace the old plugin AAR with the build from this fork and rebuild your app.

The storage read permissions you saw before came from the old plugin's own
manifest, which Godot merged into your app at export time. They were never set
in your export preset, so there is nothing to remove by hand. After you rebuild
with this fork, the merged manifest of your app contains no storage read
permissions.

Credit & license
=================

Original plugin by [Lamelynx](https://github.com/Lamelynx/GodotGetImagePlugin-Android).
Photo Picker fork by [cenullum](https://github.com/cenullum).
The original MIT license is preserved, with both authors credited, see [`LICENSE`](LICENSE).

# Donation
You can buy my games on [https://cenullum.itch.io](https://cenullum.itch.io)

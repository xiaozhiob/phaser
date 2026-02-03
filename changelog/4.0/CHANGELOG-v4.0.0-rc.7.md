# Version 4.0.0 - Release Candidate 6

## New Features

- `GameObject#isDestroyed` flag helps you avoid errors when accessing an object that might have removed expected properties during destruction.
- `Tint` is overhauled.
  - `tint` and `setTint()` now purely affect the color settings.
    - Previously, both would silently deactivate fill mode.
  - `tintFill` and `setTintFill()` now set the tint fill mode, not fill color.
  - `Phaser.TintModes` enumerates valid tint fill modes.
    - `MULTIPLY`
    - `FILL`
    - `ADD`
    - `SCREEN`
    - `OVERLAY`
    - `HARD_LIGHT`
  - FILL mode now treats partial alpha correctly.
  - BitmapText tinting now works correctly.
- `CombineColorMatrix` filter for remixing alpha and other channels between images.
- `Key` filter for removing or isolating colors.
- `ImageLight` filter for image-based lighting, a soft, highly realistic form of illumination.
- `PanoramaBlur` filter for adjusting images for `ImageLight`.
- `NormalTools` filter for manipulating normal maps.
- `TextureSource#setSource` method for updating the source of a texture. Note that, while the source will update, derived values such as object sizes will not. It's advisable to switch between textures of identical size to avoid unexpected transforms.
- `TextureSource#setDataSource` method already existed, but has been changed to be more useful like `setSource`.
- `TextureManager#addFlatColor` method for creating a flat texture with custom color, alpha, width, and height. This is intended to act as a temporary stand-in for textures you might not have loaded yet.
- `TextureSource#updateSource` method for switching sources directly.
- New `Phaser.Types.Textures.TextureSource` and `Phaser.Types.Textures.TextureSourceElement` types to simplify the increasing number of sources for a texture.

## Fixes

- Fix `SpriteGPULayer` creation time handling getting confused by 0.
- Fix blend modes leaking onto siblings within a `Container`. Thanks to @saintflow47, @tickle-monster and @leemanhopeter for reporting this.
- Fix texture offsets in `ParseXMLBitmapFont`. Thanks to @leemanhopeter.
- Fix `TextureManager.addUint8Array` method, which got premultiplied alpha wrong and flipY wrong.

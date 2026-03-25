# Cameras
> Camera system in Phaser 4 -- CameraManager, main camera, viewport vs scroll, zoom, bounds, following sprites, camera effects (fade, flash, shake, pan, zoomTo, rotateTo), ignore lists, filters, and keyboard controls.

**Key source paths:** `src/cameras/2d/CameraManager.js`, `src/cameras/2d/BaseCamera.js`, `src/cameras/2d/Camera.js`, `src/cameras/2d/effects/`, `src/cameras/controls/`
**Related skills files:** game-setup-and-config.md, sprites-and-images.md, filters-and-postfx.md

## Quick Start

```js
// In a Scene's create() method:

// Access the default camera (created automatically)
const cam = this.cameras.main;

// Scroll the camera to look at a different part of the world
cam.setScroll(200, 100);

// Center camera on a world coordinate
cam.centerOn(400, 300);

// Zoom in (2x) -- values < 1 zoom out, > 1 zoom in
cam.setZoom(2);

// Follow a sprite with smooth lerp
cam.startFollow(player, false, 0.1, 0.1);

// Constrain the camera to the world bounds
cam.setBounds(0, 0, 2048, 2048);

// Fade in from black over 1 second
cam.fadeIn(1000);

// Add a filter to the camera (v4 feature)
cam.filters.external.addBlur(1, 2);
```

## Core Concepts

### CameraManager

Every Scene has a `CameraManager` accessible via `this.cameras`. It manages all cameras for that Scene and is registered as a plugin under the key `'CameraManager'`.

```js
// The manager is at this.cameras (not this.camera)
this.cameras              // CameraManager instance
this.cameras.cameras      // Array of Camera objects (render order)
this.cameras.main         // Reference to the "main" camera (first one by default)
this.cameras.default      // Un-transformed utility camera (not in the cameras array)
```

**Key methods on CameraManager:**

| Method | Signature | Description |
|---|---|---|
| `add` | `(x?, y?, width?, height?, makeMain?, name?)` | Create a new Camera. Defaults to full game size at 0,0. Returns `Camera`. |
| `addExisting` | `(camera, makeMain?)` | Add a pre-built Camera instance. Returns the Camera or `null` if it already exists. |
| `remove` | `(camera, runDestroy?)` | Remove and optionally destroy a Camera or array of Cameras. If main is removed, resets to cameras[0]. |
| `getCamera` | `(name)` | Find a Camera by its `name` string. Returns Camera or `null`. |
| `getTotal` | `(isVisible?)` | Count cameras. Pass `true` to count only visible ones. |
| `fromJSON` | `(config)` | Create cameras from a config object or array. Used for scene-level camera config. |
| `resetAll` | `()` | Destroy all cameras and create one fresh default camera. |
| `resize` | `(width, height)` | Resize all cameras to given dimensions. |

**Camera limit:** The manager supports up to 31 cameras that can use `ignore()` for Game Object exclusion (IDs are bitmasks). Cameras beyond 31 get ID 0 and cannot exclude objects.

### Main Camera

The `main` property is a convenience reference to a Camera, typically `cameras[0]`. It is set automatically when:
- The scene boots (first camera created becomes main)
- You pass `makeMain: true` to `add()` or `addExisting()`
- The current main camera is removed (falls back to `cameras[0]`)

### Viewport vs World (Scroll)

A Camera has two independent coordinate concepts:

1. **Viewport** -- The physical rectangle on the canvas where the Camera renders. Controlled by `setPosition(x, y)`, `setSize(w, h)`, or `setViewport(x, y, w, h)`. By default, fills the entire game canvas.

2. **Scroll** -- Where the Camera is "looking" in the game world. Controlled by `scrollX` / `scrollY` properties or `setScroll(x, y)`. Scrolling does not affect the viewport rectangle.

```js
// Viewport: a 320x200 mini-map in the top-right corner
const miniCam = this.cameras.add(480, 0, 320, 200);

// Scroll: make the mini-map look at a different world area
miniCam.setScroll(1000, 500);

// Zoom: the mini-cam shows more of the world
miniCam.setZoom(0.25);
```

**worldView** is a read-only `Rectangle` updated each frame that reflects what area of the world the camera can currently see, accounting for scroll, zoom, and bounds. Use it for culling or intersection checks.

```js
const view = cam.worldView; // { x, y, width, height }
```

## Common Patterns

### Scrolling the Camera

```js
// Direct property access
cam.scrollX = 100;
cam.scrollY = 200;

// Chainable setter
cam.setScroll(100, 200);

// Center the camera on a world coordinate
cam.centerOn(500, 400);

// Get the scroll values needed to center on a point (without moving)
const point = cam.getScroll(500, 400); // returns Vector2
```

### Following a Sprite

```js
// Instant follow (lerp = 1, the default)
cam.startFollow(player);

// Smooth follow with lerp (0..1, lower = smoother)
cam.startFollow(player, false, 0.1, 0.1);

// Full signature:
// startFollow(target, roundPixels?, lerpX?, lerpY?, offsetX?, offsetY?)
cam.startFollow(player, true, 0.05, 0.05, 0, -50);

// Change lerp or offset after starting follow
cam.setLerp(0.08, 0.08);
cam.setFollowOffset(0, -50);

// Dead zone: camera only scrolls when target leaves this rectangle
cam.setDeadzone(200, 150);

// Stop following
cam.stopFollow();
```

`startFollow` accepts any object with `x` and `y` properties -- it does not have to be a Game Object. Lerp of `1` snaps instantly; `0.1` gives smooth tracking. A lerp of `0` on an axis disables tracking on that axis.

When a deadzone is set, the camera does not scroll while the target remains inside the deadzone rectangle. The deadzone is re-centered on the camera midpoint each frame.

### Zoom

```js
// Uniform zoom
cam.setZoom(2);       // 2x zoom in
cam.setZoom(0.5);     // zoom out (see twice as much)

// Independent horizontal/vertical zoom
cam.setZoom(2, 1);    // stretch horizontally

// Read current zoom
cam.zoom;    // shortcut -- reads zoomX (assumes uniform)
cam.zoomX;
cam.zoomY;
```

**Never set zoom to 0.** The minimum is clamped to 0.001.

### Bounds

```js
// Constrain scrolling to a world area
cam.setBounds(0, 0, worldWidth, worldHeight);

// Center on the new bounds immediately
cam.setBounds(0, 0, 2048, 2048, true);

// Temporarily disable bounds without removing them
cam.useBounds = false;

// Remove bounds entirely
cam.removeBounds();

// Read the current bounds
const rect = cam.getBounds(); // returns a new Rectangle copy
```

Bounds only restrict scrolling. They do not prevent Game Objects from being placed outside the bounds, and they do not affect the viewport position.

### Multiple Cameras

```js
// Full-screen main camera
const main = this.cameras.main;

// Mini-map in top-right corner
const minimap = this.cameras.add(600, 0, 200, 150).setZoom(0.2).setName('minimap');
minimap.setScroll(0, 0);
minimap.setBackgroundColor('rgba(0,0,0,0.5)');

// HUD camera that doesn't scroll (ignores world objects, shows HUD layer only)
const hudCam = this.cameras.add(0, 0, 800, 600).setName('hud');
hudCam.setScroll(0, 0);

// Make the main camera ignore HUD objects
main.ignore(hudGroup);
// Make the HUD camera ignore world objects
hudCam.ignore(worldGroup);

// Find a camera by name
const found = this.cameras.getCamera('minimap');

// Remove a camera
this.cameras.remove(minimap);
```

### Camera Effects

All effects are on the `Camera` class (not `BaseCamera`). Each returns `this` for chaining. Effects that are already running will not restart unless you pass `force: true`.

#### Fade

```js
// Fade out to black over 1 second
cam.fadeOut(1000);

// Fade out to red
cam.fadeOut(1000, 255, 0, 0);

// Fade in from black over 500ms
cam.fadeIn(500);

// Lower-level: fade (out direction) and fadeFrom (in direction)
// with a force parameter
cam.fade(1000, 0, 0, 0, true);       // force start even if running
cam.fadeFrom(1000, 0, 0, 0, true);

// With per-frame callback
cam.fadeOut(1000, 0, 0, 0, (cam, progress) => {
    // progress goes from 0 to 1
});
```

Fade direction: `fadeOut` / `fade` goes transparent-to-color. `fadeIn` / `fadeFrom` goes color-to-transparent.

#### Flash

```js
// White flash over 250ms (default)
cam.flash(250);

// Red flash over 500ms
cam.flash(500, 255, 0, 0);

// Full signature: flash(duration?, r?, g?, b?, force?, callback?, context?)
cam.flash(300, 255, 255, 255, true, (cam, progress) => {});
```

#### Shake

```js
// Default shake: 100ms at intensity 0.05
cam.shake(100);

// Stronger shake for 500ms
cam.shake(500, 0.02);

// Independent x/y intensity using a Vector2
cam.shake(300, new Phaser.Math.Vector2(0.1, 0.02));

// Full signature: shake(duration?, intensity?, force?, callback?, context?)
```

The `intensity` value is a small float. The default 0.05 means the camera shifts up to 5% of the viewport size.

#### Pan

```js
// Pan camera center to world coordinate (400, 300) over 2 seconds
cam.pan(400, 300, 2000);

// With easing
cam.pan(400, 300, 2000, 'Power2');

// Full signature: pan(x, y, duration?, ease?, force?, callback?, context?)
cam.pan(800, 600, 1500, 'Sine.easeInOut', false, (cam, progress, x, y) => {});
```

Pan scrolls the camera so its viewport center finishes at the given world coordinate.

#### ZoomTo

```js
// Animate zoom to 2x over 1 second
cam.zoomTo(2, 1000);

// With easing
cam.zoomTo(0.5, 2000, 'Cubic.easeOut');

// Full signature: zoomTo(zoom, duration?, ease?, force?, callback?, context?)
```

#### RotateTo

```js
// Rotate camera to 45 degrees (in radians) over 1 second
cam.rotateTo(Phaser.Math.DegToRad(45), false, 1000);

// Shortest path rotation
cam.rotateTo(Math.PI, true, 1500, 'Quad.easeInOut');

// Full signature: rotateTo(angle, shortestPath?, duration?, ease?, force?, callback?, context?)
```

The angle is in **radians**. Set `shortestPath` to `true` to take the shortest rotation direction.

#### Reset All Effects

```js
cam.resetFX(); // stops and resets all running effects (rotate, pan, shake, flash, fade)
```

### Keyboard Controls

Phaser provides two built-in camera control classes. Both require you to call `update(delta)` in your scene's `update` method.

#### FixedKeyControl

Moves at a fixed speed per frame. No smoothing.

```js
// In create():
const cursors = this.input.keyboard.createCursorKeys();
this.camControl = new Phaser.Cameras.Controls.FixedKeyControl({
    camera: this.cameras.main,
    left: cursors.left,
    right: cursors.right,
    up: cursors.up,
    down: cursors.down,
    speed: 0.5          // can also be { x: 0.5, y: 0.3 }
});

// In update(time, delta):
this.camControl.update(delta);
```

#### SmoothedKeyControl

Applies acceleration, drag, and max speed for smooth camera movement.

```js
this.camControl = new Phaser.Cameras.Controls.SmoothedKeyControl({
    camera: this.cameras.main,
    left: cursors.left,
    right: cursors.right,
    up: cursors.up,
    down: cursors.down,
    zoomIn: this.input.keyboard.addKey('Q'),
    zoomOut: this.input.keyboard.addKey('E'),
    zoomSpeed: 0.02,
    acceleration: 0.06,
    drag: 0.0005,
    maxSpeed: 1.0
});

// In update(time, delta):
this.camControl.update(delta);
```

Both controls support `zoomIn` and `zoomOut` keys and a `zoomSpeed` config value.

### Ignore Lists

The `ignore` method updates a Game Object's `cameraFilter` bitmask so the camera skips rendering it.

```js
// Ignore a single object
cam.ignore(uiText);

// Ignore an array
cam.ignore([scoreText, livesIcon, pauseButton]);

// Ignore all children in a Group
cam.ignore(uiGroup);

// Ignore a Layer and all its children
cam.ignore(uiLayer);
```

This is the primary mechanism for HUD-style setups: one camera ignores world objects, another ignores HUD objects.

### Filters on Cameras (v4)

In Phaser 4, `Camera` has a `filters` property with two `FilterList` instances:

```js
cam.filters.internal   // FilterList -- applied by the system
cam.filters.external   // FilterList -- for user-added filters
```

Add post-processing effects to a camera the same way you add them to Game Objects:

```js
// Add a blur filter to the camera
cam.filters.external.addBlur(1, 2);

// Add a glow
cam.filters.external.addGlow(0xff0000, 4, 0, false, 0.1, 10);

// Add a color matrix filter
const fx = cam.filters.external.addColorMatrix();
fx.grayscale();

// Remove all external filters
cam.filters.external.clear();
```

Filters require WebGL. The camera must render via a framebuffer when filters are active. See `setForceComposite(true)` to explicitly enable this even without filters.

## Events

Camera events are emitted on the Camera instance itself (it extends `EventEmitter`). Listen with `cam.on(event, handler)`.

| Event constant | Dispatched when |
|---|---|
| `Phaser.Cameras.Scene2D.Events.FADE_IN_START` | `fadeIn` / `fadeFrom` begins |
| `Phaser.Cameras.Scene2D.Events.FADE_IN_COMPLETE` | Fade-in finishes |
| `Phaser.Cameras.Scene2D.Events.FADE_OUT_START` | `fadeOut` / `fade` begins |
| `Phaser.Cameras.Scene2D.Events.FADE_OUT_COMPLETE` | Fade-out finishes |
| `Phaser.Cameras.Scene2D.Events.FLASH_START` | Flash begins |
| `Phaser.Cameras.Scene2D.Events.FLASH_COMPLETE` | Flash finishes |
| `Phaser.Cameras.Scene2D.Events.SHAKE_START` | Shake begins |
| `Phaser.Cameras.Scene2D.Events.SHAKE_COMPLETE` | Shake finishes |
| `Phaser.Cameras.Scene2D.Events.PAN_START` | Pan begins |
| `Phaser.Cameras.Scene2D.Events.PAN_COMPLETE` | Pan finishes |
| `Phaser.Cameras.Scene2D.Events.ZOOM_START` | Zoom effect begins |
| `Phaser.Cameras.Scene2D.Events.ZOOM_COMPLETE` | Zoom effect finishes |
| `Phaser.Cameras.Scene2D.Events.ROTATE_START` | RotateTo begins |
| `Phaser.Cameras.Scene2D.Events.ROTATE_COMPLETE` | RotateTo finishes |
| `Phaser.Cameras.Scene2D.Events.FOLLOW_UPDATE` | Camera updates its follow position (each frame while following). Args: `(camera, target)` |
| `Phaser.Cameras.Scene2D.Events.PRE_RENDER` | Before the camera renders |
| `Phaser.Cameras.Scene2D.Events.POST_RENDER` | After the camera renders |
| `Phaser.Cameras.Scene2D.Events.DESTROY` | Camera is destroyed |

```js
cam.on(Phaser.Cameras.Scene2D.Events.FADE_OUT_COMPLETE, () => {
    this.scene.start('GameOver');
});
```

## API Quick Reference

### BaseCamera Properties (inherited by Camera)

| Property | Type | Default | Description |
|---|---|---|---|
| `scrollX` / `scrollY` | number | 0 | World position the camera is looking at |
| `zoomX` / `zoomY` | number | 1 | Zoom level per axis |
| `zoom` | number | 1 | Shortcut: sets/gets both zoomX and zoomY |
| `rotation` | number | 0 | Rotation in radians |
| `originX` / `originY` | number | 0.5 | Rotation origin within the viewport (0..1) |
| `x` / `y` | number | 0 | Viewport position on the canvas |
| `width` / `height` | number | game size | Viewport dimensions |
| `worldView` | Rectangle | -- | Read-only rect of visible world area |
| `midPoint` | Vector2 | -- | Read-only center of camera in world coords |
| `alpha` | number | 1 | Opacity of everything rendered by this camera |
| `visible` | boolean | true | Whether the camera renders at all |
| `backgroundColor` | Color | transparent | Background fill color |
| `transparent` | boolean | true | False when backgroundColor has alpha > 0 |
| `roundPixels` | boolean | false | Round positions to integers during render |
| `name` | string | '' | Optional name for lookup |
| `id` | number | bitmask | Assigned by manager for ignore-list filtering |
| `useBounds` | boolean | false | Whether scroll bounds are active |
| `inputEnabled` | boolean | true | Whether objects through this camera receive input |
| `disableCull` | boolean | false | Skip culling before input hit tests |
| `mask` | GeometryMask | null | Canvas-only mask. In WebGL, use filters instead. |
| `forceComposite` | boolean | false | Force framebuffer rendering (WebGL only, v4) |

### BaseCamera Methods

| Method | Signature |
|---|---|
| `setScroll` | `(x, y?)` |
| `setZoom` | `(x?, y?)` |
| `setRotation` | `(radians?)` |
| `setAngle` | `(degrees?)` |
| `setPosition` | `(x, y?)` -- viewport position |
| `setSize` | `(width, height?)` -- viewport size |
| `setViewport` | `(x, y, width, height?)` -- position + size |
| `setOrigin` | `(x?, y?)` -- rotation origin |
| `setBounds` | `(x, y, width, height, centerOn?)` |
| `removeBounds` | `()` |
| `getBounds` | `(out?)` -- returns Rectangle copy |
| `centerOn` | `(x, y)` |
| `centerOnX` | `(x)` |
| `centerOnY` | `(y)` |
| `getScroll` | `(x, y, out?)` -- returns Vector2 |
| `getWorldPoint` | `(x, y, output?)` -- screen to world coords |
| `ignore` | `(entries)` -- GameObject, array, Group, or Layer |
| `setBackgroundColor` | `(color?)` |
| `setAlpha` | `(value?)` |
| `setVisible` | `(value)` |
| `setName` | `(value?)` |
| `setRoundPixels` | `(value)` |
| `setMask` | `(mask, fixedPosition?)` -- Canvas only |
| `clearMask` | `(destroyMask?)` |
| `setForceComposite` | `(value)` -- WebGL only, v4 |

### Camera Methods (extends BaseCamera)

| Method | Signature |
|---|---|
| `startFollow` | `(target, roundPixels?, lerpX?, lerpY?, offsetX?, offsetY?)` |
| `stopFollow` | `()` |
| `setLerp` | `(x?, y?)` |
| `setFollowOffset` | `(x?, y?)` |
| `setDeadzone` | `(width?, height?)` -- omit args to clear |
| `fadeIn` | `(duration?, r?, g?, b?, callback?, context?)` |
| `fadeOut` | `(duration?, r?, g?, b?, callback?, context?)` |
| `fade` | `(duration?, r?, g?, b?, force?, callback?, context?)` |
| `fadeFrom` | `(duration?, r?, g?, b?, force?, callback?, context?)` |
| `flash` | `(duration?, r?, g?, b?, force?, callback?, context?)` |
| `shake` | `(duration?, intensity?, force?, callback?, context?)` |
| `pan` | `(x, y, duration?, ease?, force?, callback?, context?)` |
| `zoomTo` | `(zoom, duration?, ease?, force?, callback?, context?)` |
| `rotateTo` | `(angle, shortestPath?, duration?, ease?, force?, callback?, context?)` |
| `resetFX` | `()` |

### Effect Handler Properties

Each effect is accessible as a property on Camera: `fadeEffect`, `flashEffect`, `shakeEffect`, `panEffect`, `zoomEffect`, `rotateToEffect`. All have `isRunning` (boolean) and `progress` (0..1) properties you can check.

## Gotchas

1. **`this.cameras` is the CameraManager, not an array.** The array of cameras is `this.cameras.cameras`. The main camera is `this.cameras.main`.

2. **Viewport vs Scroll confusion.** `setPosition` / `setViewport` move the camera's rendering rectangle on screen. `setScroll` / `scrollX` / `scrollY` move what the camera is *looking at* in the world. These are independent.

3. **Zoom of 0 will break rendering.** `setZoom(0)` is clamped to 0.001 internally, but avoid passing 0.

4. **Effects do not restart by default.** Calling `cam.fade()` while a fade is running does nothing unless you pass `force: true`. The same applies to flash, shake, pan, zoomTo, and rotateTo.

5. **`startFollow` snaps immediately on first call.** The camera jumps to the target position, then lerps from there. To avoid a visible snap, set scroll to the target position before calling `startFollow`.

6. **Ignore list limit of 31 cameras.** Only the first 31 cameras created get unique bitmask IDs for `ignore()`. Camera 32+ gets ID 0 and cannot use Game Object exclusion.

7. **`roundPixels` and non-integer zoom.** Setting `roundPixels: true` only works correctly when zoom is an integer. Non-integer zoom with `roundPixels` causes jitter.

8. **Camera rotation does not rotate the viewport rectangle.** The viewport is always axis-aligned. Rotation is applied during rendering only.

9. **Masks vs Filters in v4.** `setMask` only works with Canvas renderer (GeometryMask). For WebGL, use `cam.filters.external` (FilterList) instead.

10. **Keyboard controls require manual `update` call.** Both `FixedKeyControl` and `SmoothedKeyControl` must have their `update(delta)` called in your Scene's `update` method -- they do not auto-update.

11. **`pan()` overrides follow.** While a pan effect is running, the follow logic is paused. It resumes after the pan completes.

12. **Camera setViewport caveats.** The viewport is limited to an axis-aligned rectangle, cannot be rotated, and filters/masks may render incorrectly with non-default viewports. For mini-cam effects, consider using `RenderTexture` or `DynamicTexture` instead.

## Source File Map

| File | Description |
|---|---|
| `src/cameras/2d/CameraManager.js` | Scene plugin managing all cameras. Access via `this.cameras`. |
| `src/cameras/2d/BaseCamera.js` | Base class with scroll, zoom, bounds, viewport, ignore, mask, background. |
| `src/cameras/2d/Camera.js` | Extends BaseCamera. Adds effects (fade, flash, shake, pan, zoomTo, rotateTo), follow, deadzone, filters. |
| `src/cameras/2d/effects/Fade.js` | Fade effect implementation. |
| `src/cameras/2d/effects/Flash.js` | Flash effect implementation. |
| `src/cameras/2d/effects/Shake.js` | Shake effect implementation. |
| `src/cameras/2d/effects/Pan.js` | Pan effect implementation. |
| `src/cameras/2d/effects/Zoom.js` | Zoom effect implementation. |
| `src/cameras/2d/effects/RotateTo.js` | RotateTo effect implementation. |
| `src/cameras/2d/events/` | Event constant modules (FADE_IN_START, FLASH_COMPLETE, etc.). |
| `src/cameras/controls/FixedKeyControl.js` | Fixed-speed keyboard camera control. |
| `src/cameras/controls/SmoothedKeyControl.js` | Smoothed (acceleration/drag) keyboard camera control. |

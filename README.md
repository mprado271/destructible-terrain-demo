# Destructible Terrain Demo

A Unity 5 demo of Worms-style destructible 2D terrain. The player aims and fires a shot; the impact punches a transparent hole in the ground sprite and rebuilds the collision shape so the character can fall into the crater.

[See a video of the project](https://www.youtube.com/watch?v=GKJsSi6irko)

The project was saved with Unity **5.0.0f4**. Open it in that editor, or another Unity 5 release that can upgrade the project. The product name in the player settings is `Worms__`.

Two scenes are in the build, in this order:

1. `Assets/Scenes/menuScene.unity` — start screen. The button calls `MenuController.OnClick`, which loads `wormScene` with `Application.LoadLevel`.
2. `Assets/Scenes/wormScene.unity` — the level.

## Controls

| Input | What it does |
| --- | --- |
| Click the menu button | Load `wormScene`. |
| `W` | Show the gun and start aiming. |
| Mouse | Aim. The body flips toward the cursor and the gun rotates to face it. |
| Hold the left mouse button, then release | Charge a shot and fire. Speed scales with how long the button was held, up to `maxTimeShooting`. Releasing `Space`, or reaching that time limit, also fires a shot that is already charging. |
| Left / Right arrows | Walk. Movement is applied only while the player is touching a collider tagged `Ground`. |

At the start of the level, `SpawnerController` spawns five oil drums at random X positions between `-10` and `10`.

Public fields on `PlayerController` (`velocity`, `bulletMaxInitialVelocity`, `maxTimeShooting`, the bullet prefab, and the gun, body, muzzle, and spawn transforms) are set on the player in the scene.

## Scripts

| Script | Role |
| --- | --- |
| `Assets/Scripts/GroundController.cs` | Clones the ground texture and carves holes in it. |
| `Assets/Scripts/BulletController.cs` | Flies, orients the sprite along the velocity, and destroys ground on impact. |
| `Assets/Scripts/PlayerController.cs` | Walking, aiming, and shooting. |
| `Assets/Scripts/MenuController.cs` | Loads the level from the menu button. |
| `Assets/Scripts/SpawnerController.cs` | Spawns the oil drums. |
| `Assets/Scripts/V2int.cs` | Integer pixel coordinate used while carving. |

## Destructible terrain

The level is built around two objects: the ground, and a bullet prefab.

**Ground** has a `SpriteRenderer`, a `PolygonCollider2D`, and `GroundController`. Its collider is tagged `Ground`. On `Start` the controller loads `ground2` from `Assets/Resources`, instantiates a copy, and creates the sprite from that copy (center pivot, 100 pixels per unit), so edits stay off the imported asset. It then stores itself in `BulletController.groundController` so every bullet can reach the ground.

**Bullet** is a prefab. `Sprite` is a child of `Bullet` and holds the graphic (`bulletSpriteTransform`). `BulletSmoke` is a child of `Sprite`: the particle trail, switched off on impact. `DestructionCircle` has the `CircleCollider2D` assigned to `destructionCircle`, and that circle is the region removed from the ground. The bullet also has a `Rigidbody2D` and a `PolygonCollider2D`, so the physics step moves it and `OnCollisionEnter2D` can fire when it touches the ground.

### Texture import

`Resources.Load("ground2")` only finds a texture inside a `Resources` folder. This project uses `Assets/Resources/ground2.png`.

The texture has to be readable on the CPU, because `SetPixel` writes into it. In the import settings that means:

- Texture Type: **Advanced**
- **Read/Write Enabled**
- Format: **RGBA 32 bit**

`ground2.png` is already imported that way (`isReadable: 1`, RGBA32).

### What happens when the bullet hits the ground

`BulletController.OnCollisionEnter2D` handles a hit against the `Ground` tag. It stops rotating the sprite to match the velocity, hides the smoke trail, asks the ground to carve out `destructionCircle`, and destroys the bullet.

`GroundController.DestroyGround` then:

1. Converts the circle's center from world space to a pixel on the ground texture.
2. Converts the circle's world width into a pixel radius with `bounds.size.x * widthPixel / widthWorld`. The rasterizer treats that value as `r`.
3. Sets every pixel inside the circle to transparent black, `(0, 0, 0, 0)`.
4. Calls `Texture2D.Apply` so the sprite shows the hole.
5. Destroys the current `PolygonCollider2D` and adds a new one. Unity rebuilds the outline from the sprite alpha, so the collision surface matches the hole.

### World position to pixel

`InitSpriteDimensions` caches the sprite size in world units (`SpriteRenderer.bounds`) and in pixels (`texture.width` / `texture.height`).

`World2Pixel` treats the sprite center as the texture center. An offset from the ground transform is scaled by pixels per world unit:

```csharp
pixelX = 0.5f * widthPixel  + (worldX - groundX) * widthPixel  / widthWorld;
pixelY = 0.5f * heightPixel + (worldY - groundY) * heightPixel / heightWorld;
```

`V2int` is a small integer pair for that pixel coordinate.

### Carving the circle

For each column `x` from `0` to `r`, the inner loop runs `y` from `0` to `sqrt(r² − x²)`. Each `(x, y)` is written into all four quadrants:

```csharp
texture.SetPixel(centerX + x, centerY + y, transparent);
texture.SetPixel(centerX - x, centerY + y, transparent);
texture.SetPixel(centerX + x, centerY - y, transparent);
texture.SetPixel(centerX - x, centerY - y, transparent);
```

`Apply()` uploads the texture. Replacing the `PolygonCollider2D` rebuilds the collision outline so the hole is part of the ground's shape.

### Aiming and shot power

Aiming stays off until `W` is pressed. After that, every frame takes the mouse position, converts it with `Camera.main.ScreenToWorldPoint`, and normalizes the vector from the player to the cursor.

The gun's Z angle is `asin(direction.y)` in degrees. When the cursor is to the left (`direction.x < 0`), the angle becomes `180 - angle`. `bodyTransform.localScale.x` is flipped so the worm faces the same side as the cursor.

Holding the left mouse button sets `shooting`, enables the muzzle effect, and resets the charge timer. `UpdateShooting` adds `Time.deltaTime` while the button is held. The shot is released when the button comes up, when `Space` comes up during a charge, or when `timeShooting` passes `maxTimeShooting`.

The fired bullet is an instance of `bulletPrefab`, placed at `bulletInitialTransform`. Its velocity is:

```csharp
direction * bulletMaxInitialVelocity * (timeShooting / maxTimeShooting);
```

A full charge reaches `bulletMaxInitialVelocity`. A shorter hold is a fraction of that speed.

While the bullet is in the air, `BulletController` reads `Rigidbody2D.velocity`, computes the same `asin` angle (plus the left-side `180 - angle` case), and sets the sprite's Z rotation to `angle + 45` so the graphic lines up with the trajectory. After a ground hit, that update stops.

Walking runs from `OnCollisionStay2D`, and only while the other collider is tagged `Ground`. Right arrow sets velocity to `Vector2.right * velocity` and faces the body right; left arrow does the opposite. The animator bool `moving` follows those keys. With neither arrow held, velocity is cleared and `moving` is false.

## License

MIT. See [LICENSE](LICENSE).

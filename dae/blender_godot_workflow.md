# 3D Asset Creation (from Blender to Godot 3.x)

- Using `Godot 3.x` PBR engine.
- `Blender 2.79` or `Blender 2.8` should be fine, since materials are not exported.
- `Better Collada Exporter` (latest version)

### Preamble

In Godot 2.1.x the importer uses external graphics files, rather than the Godot 3.x way of keeping these files inside the project directory.
This made it very convenient to export COLLADA (\*.dae) files with a Blender material mapping the image files to the collada file.
Then one only needs to import the DAE file, which imports the DAE as SCN, and the images as TEX.

Now with the PBR engine in Godot, and no way yet of mapping Cycles or EEVEE materials to COLLADA nicely, it's better to create the material in Godot, which simply combines the model with the textures, with a couple extra variables to define roughness and such.

- Workflow is always important when making 3D assets, planning ahead is everything.
  - The hard part in this case is planning animation track names. (Code logic has a much easier time dealing with assets that use uniform animation track names)
  - Forcing uniform track names is limiting and cumbersome, so a good-enough code sample to handle name variation might be preferred. This only applies if you want to mark animation tracks as loops from Blender (the Godot importer reads this from the animation track name), while otherwise you can just avoid tags in track names and keep a uniform system for simplicity (plus your asset exports are guaranteed to be uniform by definition if you follow a uniform model scale and animation playback framerate).

***

### Base Assets

**asset.blend**
- Mesh, UV, armature, animations.
- Create the mesh and UV unwrap it. Now you can use the model for 3D painting (useful for hand-painting heightmaps to avoid using more triangles).
- Export to DAE (remove materials from mesh before export).

**asset_textures.xcf** (XCF is a GIMP save file, but you can use Krita (KRA) or whichever you prefer)
- Create diffuse, specular, and height *layer groups*. Create height first and derive the others from that, especially the normal map (using gimp-normalmap plugin; "Height to normal map" pre-included if you use Krita but I like the procedural texture rendering available in GIMP).
- Export with specular and height packed into alpha channels. Make sure to save color values for alpha pixels when using the GIMP PNG exporter. All other options can be disabled, but use the highest compression (Godot converts this image anyway, so might as well make the project files smaller).
- You can also export a layer as a PNG, load into Blender for 3D viewport painting, then save it and import the result to the XCF for further editing.

***

**asset.dae**
- Exclude material.
- This can be a Wavefront (OBJ) file if no animations are included (it has basically everything but animations and materials).
  - Reimport the DAE without animation player otherwise (if you don't want to use OBJ for static assets the \*.dae.import file will remember in case you make changes from Blender).

**asset_diffuse_specular.png**
- RGBA

**asset_normal_height.png**
- RGBA
- Disable normal-map detection on Godot importer (that feature is for SpatialMaterial, we can swap Y and Z ourselves in our shader instead of using the importer (also I'm scared of what it might do to the alpha channel, although sensibly it probably preserves the channel)).

### res://assets/

**diffspec_normdpth.shader**

**asset.material** (`ShaderMaterial`)
- *diffspec_normdpth.shader*
- *asset\_diffuse_specular.png*
- *asset\_normal_height.png*
- Set `metal` and `rough` (shader values) to appropriate values for asset.

### res://scenes/

**asset.tscn** (inherits *asset.dae* (`Spatial`, `res://.import/~asset.scn`))
- Apply *asset.material* to mesh.
- *\*.gd* (some script needs to add logic, and handle animation playback)

***

### res://assets/diffspec_normdpth.shader

Code sample (Unlicense):

```
shader_type spatial;
uniform sampler2D diffuse_specular;
uniform sampler2D normal_height;
uniform float metal = 0.0;
uniform float rough = 1.0;
void fragment() {
	METALLIC = metal;
	ROUGHNESS = rough;
	ALBEDO = texture(diffuse_specular, UV).xyz;
	SPECULAR = texture(diffuse_specular, UV).a;
	NORMALMAP = texture(normal_height, UV).xzy;
	NORMALMAP_DEPTH = texture(normal_height, UV).a;
}
```

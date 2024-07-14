# The Godot Engine Survival Guide

### Assorted Workarounds and Dirty Hacks for the Godot Engine

## Importing a 3D asset (GLTF, Blender, etc.) with a single root node

When you import a 3D asset into Godot, the generated "virtual" scene tree will always have a root node to group all of the imported nodes under. In 3D, by default this root node is a `Node3D` node. Godot's import settings allow you to override the type of this node.

This setup works well if you're importing a large scene with many objects; you may even select individual objects to use physics (either as static or dynamic bodies). However, you may only be importing a single object, and especially if it's going to be a physics object, you will probably want to make it the root node of the generated scene tree.

Unfortunately, Godot does not provide a way to do this. A bunch of issues, proposals, and forum threads have been opened to request this feature: 

- [Godot Proposal #7157](https://github.com/godotengine/godot-proposals/discussions/7157)
- [Godot Issue #79086](https://github.com/godotengine/godot/issues/79086)
- [Godot Forum Threads](https://forum.godotengine.org/t/make-a-node-root-of-tree-in-the-godot-editor-from-gdscript/7823)

There are probably more.

Solution: Luckily, we can work around this with a few lines of GDScript. Godot allows the creation of Import Scripts that are executed after the import process. These scripts can be used to modify the generated scene tree.

Plop the following script into your project and make sure that your assets' import settings reference it (there's an "Import Script" field in the import settings that can be weirdly easy to miss):

```gdscript
@tool
extends EditorScenePostImport

func _post_import(scene):
	# If the scene has more (or fewer) than a single child node, we can't do anything,
	# so just return the unmodified scene instead.
	if scene.get_child_count() != 1:
		return scene

	var new_root: Node = scene.get_child(0)

	# Keep the original name so instances of this scene will have the
	# imported asset's filename by default
	new_root.name = scene.name

	# Recursively set the owner of the new root and all its children
	_set_new_owner(new_root, new_root)

	# That's it!
	return new_root

func _set_new_owner(node: Node, owner: Node):
	# If we set a node's owner to itself, we'll get an error
	if node != owner:
		node.owner = owner

	for child in node.get_children():
		_set_new_owner(child, owner)
```

Conceptually, this is extremely simple -- the script will check if the imported scene has exactly one child node, and if so, it will return it instead of the original root node. The only complication is that we also need to fix the `owner` property of the new root node and all of its children. This is necessary because the `owner` property is used to determine which scene a node belongs to, and if it's not set correctly, the asset will not be imported correctly.

## Importing a 3D asset (GLTF, Blender, etc.) with a single root node

**Problem:** When you import a 3D asset into Godot, the generated "virtual" scene tree will always have a root node to group all of the imported nodes under. In 3D, by default this root node is a `Node3D` node. Godot's import settings allow you to override the type of this node.

This setup works well if you're importing a large scene with many objects; you may even select individual objects to use physics (either as static or dynamic bodies).

However, you may be importing a scene with only a _single_ root node (representing a single game asset). You might even have configured it to be imported with physics (causing Godot to create a `RigidBody3D` node for it.) Now you have a problem: even though the object is a `RigidBody3D` node, the root node of the imported scene isn't. You don't even see the `RigidBody3D` node until you select "Editable Children"!

Sure, you can configure the scene's root node to be a `RigidBody3D` node, too, but now you'll have two `RigidBody3D` nodes. But you only need one!

You might try to work around this using an inherited scene, but Godot won't actually let you do that, unless you sever the connection between that scene and the original asset.

What you _want_ is to simply have Godot import that first node as the root node and discard the extra root node you don't need. Unfortunately, Godot does not provide a way to do this. A bunch of issues, proposals, and forum threads have been opened to request this feature: 

- [Godot Proposal #7157](https://github.com/godotengine/godot-proposals/discussions/7157)
- [Godot Issue #79086](https://github.com/godotengine/godot/issues/79086)

There are probably more.

**Solution:** Luckily, we can work around this with a few lines of GDScript. Godot allows the creation of Import Scripts that are executed after the import process. These scripts can be used to modify the generated scene tree.

Plop the following script into your project and make sure that your assets' import settings reference it (there's an "Import Script" field in the import settings that can be weirdly easy to miss):

```gdscript
@tool
extends EditorScenePostImport

func _post_import(scene):
	# If the imported asset has only a single root node, that's the only node
	# we're interested in:
	if scene.get_child_count() == 1:
		print("Imported asset only contains a single root note; discarding outer root node.")
		return _remove_root_node(scene)
		
	# If the asset contains animation, Godot's importer will put them into an
	# AnimationPlayer node. If there's only a single root object, but we also
	# have animations, we need to handle this explicitly:
	elif scene.get_child_count() == 2 and scene.get_child(1) is AnimationPlayer:
		print("Imported asset contains a single root note and an AnimationPlayer; discarding outer root node.")
		
		# First, convert the scene using our little trick.
		var new_scene := _remove_root_node(scene)
		
		# Now grab the AnimationPlayer that was generated from the asset's animations.
		var anim_player : AnimationPlayer = scene.get_child(1)
		
		# The following might seem a little verbose, but we have to be this 
		# exact in order to not trigger various Godot warnings.
		scene.remove_child(anim_player)
		anim_player.owner = null
		new_scene.add_child(anim_player)
		anim_player.owner = new_scene
		
		return new_scene
		
	# In all other cases, we will just return the scene as originally imported.
	else:
		return scene


func _remove_root_node(scene: Node) -> Node:
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

**Explanation:** Conceptually, this is extremely simple -- the script will check if the imported scene has exactly one child node, and if so, it will return it instead of the original root node. The only complication is that we also need to fix the `owner` property of the new root node and all of its children. This is necessary because the `owner` property is used to determine which scene a node belongs to, and if it's not set correctly, the asset will not be imported correctly.

This version of the script also handles the case where the imported asset contains animations. In this case, Godot will create an `AnimationPlayer` node as the second child of the root node. We need to move this node to the new root node, too.

# The Godot Engine Survival Guide

### Assorted Workarounds and Dirty Hacks for Stuff That's Missing, Broken, or Otherwise Weird in the Godot Engine

## Introduction

_TODO: motivation_

### Why not just package the scripts as Godot plugins and publish them to the Asset Library?

There are two reasons, really: first of all, in learning about these workarounds and the issues they solve, you will gain a deeper understanding of the (current) constraints and limitations of the Godot Engine (and, of course, how much you can bend them!)

But more importantly, I would love this repository to grow into a collaborative effort. I'm not the biggest Godot expert on the planet, so it will be good to have the snippets reviewed and improved by the community. Also, I'm sure that there is a breadth of other workarounds and hacks that I haven't even thought of yet.

### How and what to contribute:

If you have a workaround or hack that solves a specific Godot issue, please feel free to open a PR agains this repository.

**Please do** follow the structure of the existing snippets and provide a clear description of the problem that the workaround solves, the solution, and (optionally, but encouraged) some extra explanation on how it works. 

**Please do not** submit solutions to typical gameplay (or other) programming problems (e.g. "How do I make a character jump?" or "How do I make an object face another object?"). This repository is exclusively focused on providing workarounds for things that are missing or broken in the Godot Engine.

## Importing a 3D asset (GLTF, Blender, etc.) with a single root node

**Problem:** When you import a 3D asset into Godot, the generated "virtual" scene tree will always have a root node to group all of the imported nodes under. In 3D, by default this root node is a `Node3D` node. Godot's import settings allow you to override the type of this node.

This setup works well if you're importing a large scene with many objects; you may even select individual objects to use physics (either as static or dynamic bodies).

However, you may be importing a scene with only a _single_ root node (representing a single game asset). You might even have configured it to be imported with physics (causing Godot to create a `RigidBody3D` node for it.) Now you have a problem: even though the object is a `RigidBody3D` node, the root node of the imported scene isn't. You don't even see the `RigidBody3D` node until you select "Editable Children"!

Sure, you can configure the scene's root node to be a `RigidBody3D` node, too, but now you'll have two `RigidBody3D` nodes. But you only need one!

You might try to work around this using an inherited scene, but Godot won't actually let you do that, unless you sever the connection between that scene and the original asset.

What you _want_ is to simply have Godot import that first node as the root node and discard the extra root node you don't need. Unfortunately, Godot does not provide a way to do this. A bunch of issues, proposals, and forum threads have been opened to request this feature: 

- [Godot Proposal #7157](https://github.com/godotengine/godot-proposals/discussions/7157)
- [Godot Issue #79086](https://github.com/godotengine/godot/issues/79086)
- [Godot Forum Threads](https://forum.godotengine.org/t/make-a-node-root-of-tree-in-the-godot-editor-from-gdscript/7823)

There are probably more.

**Solution:** Luckily, we can work around this with a few lines of GDScript. Godot allows the creation of Import Scripts that are executed after the import process. These scripts can be used to modify the generated scene tree.

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

**Explanation:** Conceptually, this is extremely simple -- the script will check if the imported scene has exactly one child node, and if so, it will return it instead of the original root node. The only complication is that we also need to fix the `owner` property of the new root node and all of its children. This is necessary because the `owner` property is used to determine which scene a node belongs to, and if it's not set correctly, the asset will not be imported correctly.

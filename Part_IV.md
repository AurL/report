
# Blender development#
Blender is a professional free and open source software which provide a wide set of tools for creating animated films, visual effects, 3D models for printing or for interactive 3D applications and video games.
It as a huge community with very active forums, tutorials and independent developers who contribute to its improvement and provide addons to be used with.

Blender is also scriptable through a Python API. I spent the second part of my internship working with Blender and writing scripts to both fix and improve the support of some 3D files.

## Blender's specific pipeline ##
To process the .blend files in Sketchfab, we simply use blender calling it in command line. Command line calls allow to run Blender in background mode and run scripts. This allows to automatize operations on 3D data, created from scratch with a script or coming from a 3D file, and output the result in the same way. 

The first run script is used to convert (if needed) specific blender materials into classic ones, to be handled by OSG. After that, the exporter script is run and output a .osgt file 
The first step deals with the shader nodes (materials represented by graph). Its role is to flatten these materials to make it compatible with the current exporter version and also the internal structure of OSG files. Material flattened, a second instance of Blender is run with the exporter script, which convert all the relevant data and outputs an osg file (*.osgt*) which is then processed.

## Fixing Blender processing fails##
The stats showed that a lot of .blend files were not processed correctly and most of them were causing the processing to fail. I was assigned to the different tasks that had a link with blender, in order to find a solution.
The Sketchfab's Blender exporter works with a majority of files but doesn't handle some of them which have a specific structure. In some case, due to the specifications of the scene graph saved into the file, the processing failed.
The fail occured because no file was output by the exporter. The difficulty in this part was that the processing logs were not very accurate, so I needed to dig up using Blender and looking for the configuration of the scene graphs in order to find commonly unsupported patterns. 

The interface is very different of what I was used to use in 3DsMax, but the concepts are pretty the same. A scene graph is composed of objects (name, transform) linked to object data. The object data is in fact the active part of the object, which defines it's type (such as mesh, light or skeleton) and have the corresponding set of parameters. Blender uses a scene graph representation where the data can be shared between nodes.

### Encoding errors ####
The first issues were related to object names that were misencoded.

Since the version 2.5, Blender used Python 3.x instead of python 2.5 which was the case before. The API was migrated to be run with Python 3 including all the new data structures. For retro-compatibility purpose, Blender also has to deal with *.blend* files that come from older versions.

The encoding problem that the current sample files had was due to the fact that the newest Blender API had to deal with files that were created using the oldest version. Contrary to Python 2 which handled a lot of encodings, Python 3 encodes all the strings as sequences of Unicode characters, so the way names' characters are managed differs between the old and the new versions of blender.

The sample I had was made using a old version (2.46) and contained objects whose name had latin-1 characters. It is file when reading and managing the data with this version, but in the case of Sketchfab, all blender files are managed using a newer Blender version (2.70). Due to the new string representation by Python 3 (which is run in parallel with the new version of the C++ Blender core), scripts are not able to deal with the misencoded names.

Since in Blender, assets are managed through dictionaries whose keys are actually the element's names, it is impossible to access objects that have non Unicode characters. 

The only solution that worked for this was not really efficient, but allowed to solve the issue. The script tries to access the name property of all objects, catch the Unicode error if any and rename the object with a default, unicode name. I didn't find any other way to do it since every access has to be performed through the Python API.

Then, the fix was extended to all the Blender assets. 
This first issue concerned some isolated cases. I worked on more common issues related to some scene graph configurations that were not handled by the Blender exporter.

### Visibility check ###
By opening a bunch of failing blender files, I saw that some users tried to upload blender files that have no visible objects.

When reading and parsing the *blend* scene, the exporter actually cares about the visibility of the objects. It makes sense since the user doesn't want the hidden objects to be rendered in Sketchfab. If an object is not visible in the current scene, it is just skipped. If the entire scene has no visible objects, no data is loaded and no *.osgt* file is output.

Blender has two ways to modify the visibility of an object in a given scene. The first is to set the object on the active scene to set it visible, since only the active scene is shown in the viewer (in the case of a file with several scenes).

The first way is to manage the visibility of the object within the scene graph, using the "eye button" which shows or hide the object in both the viewer and the renderer. The second way is to manage visibility using the scene layers. Layers are used to regroup objects, which allows to hide or show entire parts of the scene to facilitate scene edition. Layer can be simultaneously activated and their object are all visible.

[Illustratio : layers UI, eye icon Blender]

The user actually recieves an error message telling him that its file can't be processed correctly. This is not very accurate and can refer to a Sketchfab internal error. A better solution is to be able to tell the user that its *.blender* file was not processed because it doesn't have any visible object. 


Instead of having to make the processing fail in this case, we preferred to detect empty scenes, set all its objects to visible and return a warning message to the user to explain what was detected and what was changed in its scene. A second visibility check if done after that and ends with an error if the scene is physically empty.

### Hidden armatures ###
The only output I had for blender process fails was the error of the empty output. I had to open files within blender and look at their data and the structure of the model they contained to try to find a common pattern. A lot of them were very nice models, and the scenes were quite complexe and used a lot of elements. I found that each of them used an Armature object so I began to check that were actually these objects and if they were responsible of the crash.

To apply non destructives modification on objects (mostly on meshes), Blender uses modifier stacks. Both the viewer and the renderer take into account the modifications of the stack, but the data is internally conserved unchanged.
Armature are a type of modifier which allow to deform a mesh according to a skeleton object. Armature are used for rigging, a technique allowing set the mesh into a given pose. 

[Illustration : model without pose and with pose]

Armature object are often hidden from the scene, as the user uses it to determine a pose and not to render it (bones are only rendered to be set, they have no rendering purpose). This was a problem when going through the exporter.

When a mesh has an *Armature* modifier applied to, it references the previously mentioned object, which contains the skeleton data. The problem here was that *Armature* objects were not visible so ignored by the exporter, but when reading a geometry that referenced it through it's modifiers, it threw an exception and Blender was closed.

Armature's data were retrieved and saved in the output file since OSG handles animations and rigging. For Sketchfab, animations are not useful so we can skip them and avoid to write all this data in the output file.

To fix it, I firstly discarded the Armatures in the visibility check. It fixed the crash but was not sufficient, output geometries were not affected by the armature modifier.

### Flattening modifiers on geometries ###
The blender exporter has a parameter to decide whether the modifiers had to be applied (flattened) or not. By default, this parameter was set to true in the Blender command line call, but armatures were discarded since they had to be written in the output file. Sketchfab doesn't need this data, but the goal was to render the model as it appeared in Blender, so modifiers simply needed to be applied in the geometry.

Avoiding to write skeleton data was a specific behavior for Sketchfab, so It has to be done separatly from the main code. When calling Blender with command line, scripts can be piped to be executed sequentially, so I wrote the new code in a separated script. This *pre-process* script had the responsibility of preparing the data to be exported for Sketchfab, and resolving all the problematic patterns mentioned before.
[Illustratino Scema preprocess]

To simplify the blend data before writing it, we decided to flatten all the modifiers stacks on the concerned geometries. With this, the model is fixed into its final form and appears as expected on Sketchfab. The modifier application was a bit tricky and we needed to handle several cases of disabled modifiers, and other conditions that made Blender to silently cancel the modifier application. 

At the end, the *pre-process* script gave good results, fixing almost 75% of the failing models. The script was a solution to fix without spending much time on it, keeping Sketchfab specific code apart. The next step is to update and refactor the main exporter to make it handle all the previously mentioned case, but I was asked to work on a project more primary.

## Kerbal Space Program ##
[Illustration : Kerbal Space Program]
One of the last projects I worked on during this internship was the support of files coming from Kerbal Space Program, a spatial simulation game.
Kerbal Space program is a space flight simulator developed by squad using Unity3D, released on June 2011. KSP is still in public beta development but has already a huge community and  reached the top 3 best sold games on Steam (currently only through the early access program). The game was made in a such way players can create and import their own 3D modeled parts into the game to build their spaceship. A lot of mods and packs are shared between the users to improve the game or propose new parts to extend the game experience.

Sketchfab is a very good alternative for these users to show their work and share it in the KSP's forums. We had a lot of requests in the exporter section from KSP users who wanted to share their work and working on this may make a lot of new productive users to join the community and use the platform. 

A KSP member developed a Blender plugin to manage the import and the export of the parts into Blender, that Sketchfab integrated in its pipeline.

[Illustration : KSP -> blender -> osgt -> SKFB]
[Illustration : parts]

The current version of the exporter is limited to the support of parts only, but doesn't not allow to manage entire crafts. It was included within the Sketchfab pipeline to allow users to upload their custom or handmade parts and share them with the KSP community. 

As said, KSP is a very good opportunity for Sketchfab to get new users, so the Sketchfab community team contacted the developer of the importer to propose a collaboration. With my previous work on Blender, I was assigned to this task and began to work on a full spaceship importer. 
The structure is pretty simple : parts are given within .mu files which are actually imported in Sketchfab, and ships are set by a .craft which contains data to puts parts together. With the help of some explanations from the developper and some samples found on the web, I started to work on. A this moment, I had the experience and the understanding of Blender I acquired during the previous works. The most difficult part was the lack of data I had about craft structure and parameters. I only focused on the data that are relevant for static 3d rendering, but some details were not very clear and searching on the web didn't helped much more, so I had to dig it blindly.

To render the ship, I only needed the data relative to the transform (position, rotation, scale) since the parts are already read by the existing code. I had to differentiate data that was internal to the game and data that concerned the 3D models.

[Illustration : part composition -> craft assemblage]

### Structure of the script ###

#### Input ####
The question of the input was important since we didn't have any informations about the copyrights of the KSP data we need to render a spaceship. Before choosing an input structure, we needed to know if parts were sharable, or if we needed to only take into account custom parts (that would not be very efficient since spaceships are not fully built with custom parts)
[Illustration : zip with sketchfab.craft + repertories with craft]

#### Output ####
Since parts are output as blend file, I was more interesting and time saving to continue working with blender and its Python API to do the job.


#### Process ####
After having taken a look around KSP data and what was needed, I established the main process of importing a craft file: 

I.  Get all the given parts : check the directories to list all the available parts

II.  Parse the craft and get the useful data
		- Get the ship name
		- Craft parts that have no given sources (.mu) must be skipped

III. Get the prefabs (usef parts), read the .mu files and generates the corresponding blender objects (prefabs)

IV. Loop on the craft's parts list :
	1. duplicate the prefab (without duplicating the data, linked=true)
	2. set the right transform
	3. Set the name (unique) of the object
	4. Put the generated object as children of the main object (with the name of the ship)
	5. Delete all the prefabs

V. Write and process the .blend file.

#### Blender part ####
I spent some time to understand the behavior of Blender's operators, to know which operator applies on selected objects, which one applies only on active object, and the software context management.

After having defined the main structure of the script, I had to spend time on Blender's documentation. For each operation, I checked which solutions were possible, looking for the most optimized and quick way to do it. Blender was very useful to manage data thanks to its API and its internal function, but it has also some drawbacks : the API is very close to the user interface and it has some constraints. Scripts need to manage active objects and object selection in order to apply operators on it. At the beginning, I had to deal with the concepts of active objects and with object selection and deselection.Operations are based on selected or active objects which can have a lot of side effects (like unwanted object duplication), and operators need a specific Blender context to be applied (and throw an exception if it is not the case). 

#### KSP part ####
In KSP, the data is packed within nodes, which made it very easy to read. The .craft uses this structure so the parsing part was almost easy to write. The first expected input was a .zip archive containing a .craft file and a set of directories containing the data of each part. Each part is composed of a .cfg file where the part meta data (parameters) is given, with a value referencing the mesh file. A mesh is a .mu file which is given with one or more textures with a *.mbm* extension. From a part to another, the file names are the same, which can cause the data to be incrementally overridden in blender. The meshes were not problematic, since the name of the imported object was correctly set. The problem occurred with the materials and the textures, so texture and materials names needed to be set accordingly to the last imported object they belong, before importing the next one. 

[Illustration : structure of a .craft file]
[ Move or delete this : ]
#### Data duplication
For Sketchfab, the data duplication is a very recurrent problematic, because it leads to bigger and more complex scene graphs which are very slow to traverse and manage in the JS side and drop down the FPS at the rendering (and this is very frustrating). Moreover, web browsers have 3D data restrictions, and client's hardware is not always "performant" enough to deal with all the data. In the case of editable data like the materials, it can make the edition very hard and fastidious to do and impact the user experience.
The pipeline uses a lot of data and scene graph optimization like geometry merging and compression, which need to traverse several times the scene graph. These operations are useful but can be very slow and increase significantly the processing time in the case of a very complex and large scene graph.

## Plugins refinements ##

#### Options and refactoring ####
When adding new behaviors to an existing code, It is important to keep it consistent and keep its logic. For huge changes, like FBX, It was very important to maintain the logic of the whole plugin. In a lot of cases, I had to refactor a part of the code before adding my changes. In software engineering, we need to follow some guidelines to make an efficient refactoring. With the help of my colleagues and good references on the web, I learnt how to refactor to obtain a code that makes sens. I also understood all the interest of the regression tests in this work : they are basically required to prevent from breaking the existing behaviors.

Avoiding to break behaviors in also a problematic when working on code snippets that are used by a lot of developers. For this, I managed to put the new behaviors as optional, in order to prevent any issue with the existing code based on OSG when needed.

### FBX ###
[Illustration(inline) : FBX logo + autodesk logo]

FBX is a very powerful 3D format own by Autodesk, which is the most influent 3D softwares editor. It owns all the most used and powerful 3D authoring tools, such as 3DsMAx, Maya, Mudbox, but also AutoCAD. FBX (for FilmBox) was designed to serve as a bridge format between these softwares, and is very complete. Contrary to STL or OBJ, FBX contains rich sets of data about the whole scene graph and the different 3D assets. FBX was originally developed by Kaydara for its software Filmbox. Its purpose was to record animation data from motion capture devices. FBX uses a object-based model, allowing the storing animations along with 2D, 3D, audio and video data. Textures can also be embedded within the fbx file.

FBX is a proprietary format, but a SDK  is also provided by Autodesk to increase its use. FBX exists in both ASCII and binary formats, and the standards are updated yearly (a given FBX is written according to a given version). OSG provides a FBX plugin based on the C++ SDK but the file format is not fully parsed. As the other plugins, this one was developed to fit with some users' specific needs. At this moment, the plugin suffered of a big lack of supported features, but was widely used by Sketchfab users.

I improved the plugin on both the way it manages topologies and the way it supports materials and textures maps. I began to work on the primitives support. In fact, FBX was able to parse any type of topology but the output was always composed of triangles.

#### Geometries and wireframe ####
The GPU is only capable of dealing with triangles, so each sent geometry is internally tessellated (triangulates) before being sent to the GPU. When rendered or managed by softwares, models can be composed of more types of primitives : it can also have quads or polygons, which have some advantages for building and editing purposes. 

The wireframe is a rendering mode which only renders the lines that compose these primitives. Artists care a lot with it, because it is a way to show the quality of the work done. 

The support reported some mails from users that asked why the wireframe of their models were still broken into triangles instead of being the same as the one they had in their authoring tool, so I managed to improve this in the FBX plugin. 

I worked on the FBX plugin to make it support the other primitive types to avoid modifying the original topology of the model. Looking at the FBX format specifications, all primitives are supported, but taking a look into the plugin code showed that everything was automatically tessellated into triangles. This is not a design problem or a bad choice, the developer just didn't care of the topology when writing the plugin. For our purpose, we had to handle the quads and the polygons in order to keep the wireframe unchanged. To keep it modular and prevent the existing code to break, I made the new behavior to be executed according to a *keepPrimitive* option. By default, the behavior remains the same. This option is also added in the processing variables to be enabled for our own use at Sketchfab.

[Illustration : difference with wireframe . Create a custom piece for this]


One of the difficulty of this part was to add the new behavior without breaking the logic and keeping a readable code. As said previously, changes have to be done in a such way the submission has to be easily accepted. This feature was made to be pushed in the official OSG repository.

After that, I focused on the way materials and textures were retrieved from FBX files. We had several reports from users about textures that were lost when uploading on Sketchfab, causing the model to appear very differently.

#### Maps and Materials ####
Map is a term to define textures of different types that have an influence on the material they are assigned to. The term refers to the fact that values are *mapped* onto an image through the RGBA values. Maps can be used to affect floats (factors) and vectors (colors,normals).

Textures are mapped on geometry according UV coordinates that are generally represented as 2D vectors. For a given triangle, vertex are mapped onto the texture according to their own coordinates, and the rest of the face's points are mapped by interpolation between these values. 
[illustration : triangle, texture, mapping]

In 3D, a material determines the way an object reacts with the lighting and its appearance. When assigned to a geometry, a material is used to defines the colors of the object's surface. In most of the cases, the material has several textures that are mapped onto the geometry and mixed to produce the given result. OSG doesn't support multi-material, so the geometry has to have a texture coordinates (also called UV) channel for each used map.

More precisely, a material is composed of a set of parameters and maps whose combination is defined within a shader. A shader is globally a small program that takes 3D data as input and outputs a pixel color. So the appearance of a material and its behavior with the lighting depends on both the input data and the way the shader manage them.

[Illustration : material main schema with shader and result]

Parameters can be colors, factors, booleans depending on the type of values they encode. Object are generally textures, which can be considered as an object composed by an image and a set of parameters defining how to utilize it.

A material is divided in several channels, having their own parameters and interpretation within the shader. Even if each software may have its own channel naming and definition, it is always possible to find relations between them. The channels that are going to be detailed are relative to real time rendering. Some channels are reserved for off-line rendering since they require a lot of calculations and are very difficult to use in real-time.

Here is the list of channels that are used in Sketchfab and their function:

[Illustration : Material editor screenshot at the left]

* Diffuse:
	The diffuse channel defines the main color of the material. Even if it can basically be a simple color, it often uses a texture.
	The lighting shades the color according to the model's surface and the light(s) position(s).

* Specular:
	The specular is used to simulate the reflection of the light by a shiny surface. Here, the channel has two parameters : the color of the specular spot and the glossiness, which can be identified as the "radius" of the spot. Glossiness is also called shininess since it defines the shininess of the surface.

* Normal/Bump Map:
	This channel is used to add details on the surface by altering its lighting. Given a flat surface, the shader will use the set texture to perturb the lighting on the surface, altering its normals, which allows to fake bumps and dents.
	This channel uses two types of texture : the bump texture, which is a heightmap, and a normal texture which defines the surface's normal (x, y, z) by translating them into the (r, g, b) space. Normal mapping is detailed further in the tangent space part.

* Lightmap:
	As its name indicate, the lightmap is used to define the lighting of a surface using a texture. It is commonly used to add shadows to the model without having to generate them by calculation. This process is called texture "baking".

* Emission:
	The emission is used to make a surface glow, or making it insensitive to the lighting. It is commonly used for object that release light (even if the light is not often casted into surrounding objects)

* Environmental Reflection : 
	Defines what is reflected by the surface. They are several ways to generate reflections such as ray-tracing (which consists of throwing rays and getting the point they are reflected to), but they are very expensive to compute to be efficient in real-time rendering. In the case of real-time rendering, reflection is made by pre-computing a special texture (cube map) and mapping it to the object. Reflections and specular behavior are linked here. The level of reflectiveness varies according to the specular level.

* Face Rendering:
	This is a material parameters defining if the shader has to render only the front face, or both the front and back faces. If front-face only is set and the face's normal points in the same direction as the camera, the face will not be rendered.

[Illustration : make a sphere with incremental rendering while adding maps]

At the beginning, only half of these map were supported by the FBX plugin, other were just skipped(lost). With the help of the FBX SDK documentation, I added the code to retrieve the missing maps and manage their related data in order to have them in Sketchfab. The corresponding texture coordinates also needed to be retrieved from the geometries to prevent mapping issues in the viewer.

At this step, map were retrieved correctly but the user had to set all of them using the editor.

#### Setting metadata to keep track of maps ####
Maps were here, but the link between them and the channels was lost so the front end part didn't had enough data to automatically re set the maps into their respective channels.

Looking to the internal structure of OpenGL (and so OSG), there are no existing semantic to determine how to map the textures onto channels. The concept of channel is abstract : it is actually a texture unit. 

In openGL, a texture unit is a texture object which packs together an image (texture) and a set of sampling and texture parameters. They are identified and linked to the OpenGL context using indexes. The order they are set is arbitrary, so it varies depending of the implementation.

In our case, connections are lost since Sketchfab interprets differently the texture units than the FBX plugin does. OSG has some containers which allows to store custom data on objects. These containers are very helpful to keep track of the texture units' mapping through the processing. 

The *User values* consist on *(key, values)* pairs that are serialized within the geometry data. The JavaScript code has specific scene processors to parse the generated 3D files, retrieve and interpret these values. These data are specific to Sketchfab 's interpretation of the channels, so they were not pushed to the OSG trunk.

#### FBX Data optimizations ####
The newly added maps and vertex data (texture coordinates) highlighted data duplication problems. For geometries having materials with several channels, the shared UV arrays were duplicated for each UV channel. The FBX file structure handles references and shared data, so this was due to a legacy behavior of the plugin. The duplication was fixed by keeping track of the internal references within the FBX and restoring them while building the OSG output.

The FBX plugin also appeared to duplicate materials. The way the plugin read the FBX file was not efficient. A FBX file is structured as follow :

[Illustration : SCHEMA FBX nodes, main hierarchy of a FBX file]   

To understand where the duplication was made, I had to take a look on how FBX deals with data and nodes.
Within FBX, a geometry may have several materials (contrary to OSG which doesn't support this feature). A FBX Node which contains a geometry also contains a list of references of scene materials. At this step, the plugin generated a material for each reference found in the FBX node. If the scene contains only one material that is used by several FBX nodes, the plugin generated an OSG material for each node, that is the origin of the duplication issue. 

To avoid unnecessary memory consumption and make the material edition more efficient, I recast the way the OSG plugins handles FBX material.

Materials are now generated at the beginning of the parsing according to the scene data. In fact, the used materials are referenced within the FBX scene description before being referenced by the nodes. The changes made on the FBX plugin at this step were important enough to worth a refactoring. This part of the plugin is now cleaner and more modular.

#### Normal mapping and Tangent spaces ####
The goal of the normal mapping is to simulate normal perturbations of a 3D surface with new (fake) normal values that are encoded within a texture. In fact, normal maps are commonly stored as RGB image where the RGB component correspond to the X, Y and Z coordinates respectively, of the surface normal. 

When rendering using normal mapping, the normal texture is mapped onto the face according to the UV coordinates of the geometry. Image's data is not used to define the color at a given pixel, but the color values are converted into a normal vector which is used instead of the existing normal to compute lighting.

Tangent spaces are computed for each vertices and are used to apply normal mapping effects on geometries. Their data is used to convert normal map's data from face's space to model space, to be then converted to world space and used in lighting calculations.

[Illustration : triangle, normal map, drawing and result or ]
[Illustration : normal map, RGB normal, tangent space and normal map result with lighting (2x2)]
To build the tangent space of a given vertex (that is the interpolated through the face), we need a normal, a tangent and a binormal vector. Even if normals are handled by the majority of the 3D file formats, this is rarely the case for tangent and binormals, except for a few ones, such as FBX.

In the Sketchfab's pipeline, tangent spaces are systematically recomputed for each geometries in order to support the existing normal mapping (or to allow the further application of the normal mapping), but it is useless when we can
get the data from the 3D file.

(Tangent, Binormal, Normal) 

After retrieving these tangent and binormal vectors, tangent spaces can be *compressed* by merging both in a single 4D vector. The binormal vector is usually recomputed by cross product at the shading step (for performances) using the normal and the tangent vectors. The purpose of using a 4D vector is to store the tangent data in the three first component and use the fourth to indicate the sign of the binormal, in order to be able to choose between the two possible results of the cross product.

I added the tangent and binormal retrieval to the FBX plugin, and the conversion into 4D vector. To allow more flexibility on their retrieval and conversion, I added two option so that the user can disable the tangent and binormal parsing, and also disable the use of a single 4D vector (tangent and binormals are so stored a two different vertex attributes arrays)

### OBJ ###
Obj is one of the most commonly used 3D file format, since it is human readable and very easy to write and parse. Developed by Wavefront Technologies, it contains the basic geometry data (positions, normals, texture coordinates and vertex colors), materials and geometry groups, but all scene or transform data are ignored. For each object, a first part gives the vertex data, and the second part describes the geometry by referencing them.

[Illustration : obj screenshot + plane]

Materials are also given within .mtl files. A single .mtl file can define multiple materials with :

Ka : ambient color (RGB)
Kd : diffuse color (RGB)
KS : specular color (RGB) + Ns : specular exponent (float)

The MTL also supports multiple illumination models, configurable by setting values using the field "d" or "Tr" depending on the implementation. It also support referencing textures maps (map_Kd, ...) and handles textures parameters.


OBJ are straightforward to read and write, so some vendors don't hesitate to alter it or add custom contents. For example, zBrush, a well known sculpting tool, exports OBJ files with a custom layer of data for its polypaint and masking data.

[Illustration : zBrush icon]
ZBrush is a famous digital sculpting tool developed by Pixologic, widely used in the Video game and movie industry. It provides a very rich set of tools to manage 3D objects as if they were real sculpting materials, which is more adapted for biological structures such as characters or animals. 3D sculpt doesn't have to deal with scene graph or matrices since the model is most of the time composed of a single big mesh, for which OBJ is adapted.

zBrush proposes a polypaint tool which allow to paint directly on a 3D mesh. Instead of painting onto a texture and mapping it into the model, the color data is directly stored for each vertex. During a sculpting session, the mesh is constantly modified, and it topology is often rebuilt by *remeshing*. The workflow is not adapted for the use of a texture because during a *remeshing* operation (that homogenize and simplify the structure of the mesh without altering its look), UV are lost.

So instead of using a texture, vertex colors are more commonly used in zBrush.
Masks are also used to improve the painting by modifying the way the paint is applied to the surface. These data added when exporting an OBJ file from zBrush, in order to be saved for further use.

[illustration : screenshot zBrush vertex colors]

Even if the data is very specific, there are a lot of zBrush artists that upload obj models on Sketchfab, so handling the colors is worthing.

### Primitive support for PLY ###
PLY (for Polygon File Format) is very similar to OBJ. Their structure is very adapted for 3D scanning and both are mostly used in this area. 

Sample of ply file :
[Illustration  : screenshot of ply file]


One of the principal lack that suffered the OSG PLY plugin in OSG was that it only supported triangles primitives. When a model had quads or polygon primitives, the output model was rendered as a point cloud(consisting on printing the vertexes, without building faces between). Some 3D scans and sculpts were uploaded using PLY file and were rendered as point cloud.

As for obj, I improved the ply plugin to manage these unsupported types of primitives to allow the result to be more accurate with the source. I also worked on it to make it more flexible about certain properties since they can be some delta from one file to another.
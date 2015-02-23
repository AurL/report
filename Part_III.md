# Fixes and Improvements #
Working on plug-in issues was a little different of working on a project. It requires a lot of reflexion and understanding of the existing code to find a solution. During these different works, I spend the most of the time reading the code and trying to understand the whole process before being able to spot where the issue were located and looking for a solution. The solutions are finally not very complicated, but the difficult part is to find where to fix it, taking into account the whole process and avoiding all the side effect the modifications can cause. I personally found this very interesting for learning 3D concepts.

Moreover, in some cases I had to begin by refactoring the existing code before adding the new one. Working on a shared project require the developers to care more about the readability, the quality and the maintainability of the code. In this case, it has to be easily understandable to allow OSG users to use or improve it if needed. These are some rules that are primary in software engineering, whatever the concerned project.

## Normals consistency and correctness ##
Normals correctness and consistency was the first topic I work at Sketchfab.
A lot of uploaded models had rendering issues that were generally related to these data, so Sketchfab needed to have some code to solve this.

###Normals in geometry data###
A 3D model is composed of vertexes that define faces. Basically, a vertex is defined by its position (is local space), with a set of data such as normals, vertex colors or texture coordinates. Vertex data can be mapped per faces, but this mapping is more generally done per vertex.
[Illustration : model wireframe -> vertex -> vertex data]

As the name suggests, these data are used to define the normal of a surface at a given point : they are always represented as 3D vectors. At the render step, normal are mostly taken into account for lighting et reflection calculation. They can also be used to compute effects such as refraction or in more complex effects such ad BRDF (for Bidirectionnal reflectance diffusion functions) that varies according to the surface properties. 

In the case of **flat shading**, all the vertexes of a given face have the same normal and the face is lit uniformly in all its surface. For **smooth shading**, these vertices may have different normal values, that are applied to the whole surface by linear interpolation. This generates a continuous and homogeneous lighting of the surface and allows to create smooth surfaces without the need of a lot of vertexes.

[Illustration : Flat vs smooth] 

### Deviation tolerance ###
Faces are considered co-planar, which means that all of its vertices are contained in the same plane. The normals of each can be oriented in whatever direction, but needs to be included in the hemisphere oriented towards the implicit face normal.

In the reality, there are some cases where the data doesn't obey this rule, leading to unnatural and weird lighting on the corresponding surface.

[Illustration : bad normal orientation]

I first focus on this case and wrote some code to visit all the geometries of the scene and check the normal orientation. The goal was to be able to tell the user that its models has normals that don't respect this rule and may not render correctly. A bad normal data can have a huge impact on the model, being responsible of rendering issues such as black parts or weird face appearance. For this purpose, we cannot automatically recompute the bad values for the users, since this misorientation can be willingly set to achieve some special effects.

So the bad normal detection consists of checking the deviation of each vertex normal with the face's one and ensure that it is lower than 90°. It it deviates too far, a warning is output.

[Illustration : arc with normal and illumination rendering]

[Illustration : bad normals showed and problem targeted] 

### Normal consistency ###
To go further in the problem, some models actually have normal data that are inconsistent and leads to more important rendering issues.

Normals are mostly bound PER_VERTEX (per face is deprecated), that means that we need a one to one mapping between the vertexes and normals arrays. Some models don't have any normal data, others have some but not enough, which leads to vertexes without normal data, and some other have nulls normals. Exporters don't always write normal data, it is often optional and values are recomputed during the reading step.

Basically, the lighting of a surface is computed using the dot product between the light vector and the normalized normal of the surface, so a null normal leads to an unlit (or partially unlit) surface. 

OSG provides a lot of tools to deal with geometries and the scene graph in general, based on the design pattern *Visitor* and *Functor*. The visitor is in this case very useful and powerful since it allows to apply a function in a set of similar objects. 

To smooth a geometry, a visitor will visit the geometry's normals and recompute them according to its topology.

We differentiate face normals and vertex normals. Each face of a geometry is supposed to have its vertexes coplanar. In most of the cases, primitives are triangles, which ensure that this rule is respected, and only quads are supposed to be planar. As a consequence, a face has a unique normal which is the normal of the plane it belongs to.
Another way to define the normal of a face is by interpolating the normal values of its vertexes. This allows to obtain what is called *smooth shading*, which avoids the model to appear faceted by giving the illusion of a smooth surface.
For each triangle :

1. Compute the faces' normal and add the value to the three vertices's normal

2. Normalize the normal

3. Check for normals that deviate too far, according to a crease angle value
The crease angle is used to determine the maximum angle allowed between two adjacent faces. 

4. If the normals of two adjacent triangles deviate too far, the shared vertexes are duplicated.
[Illustration : triangles duplicated]

5. Recomputes the normals for the new topology

In order to be run on all models, this code was put in a cleaner pseudo loader which is called after the 3D plug-ins. Pseudo loaders are plug-ins that can be plugged right after any reader to perform post operations on the model or more generally on the data that was previously parsed. 

Normals were the first data I dealt with, and were a good point to begin learning to understand and manage 3D data.

### Warning messages through the processing###
The cleaner's purpose is firstly to clean the data that is not consistent, but also to do some checks and inform the user that the data he submitted has some issues. When detecting a bad normal or performing a data correction, a warning message is written into the standard output and is then flushed into a log file.

The problem of a such output is that it can be very tedious to parse as it contains every single processing output. To simplify the process, a python script has the responsibility of parsing the processed.log file and report the messages into a JSON file which is more readable by JavaScript code. To do so, I added functions in the existing processing python code code to allow to store warnings according to the type of data they refer to. It organizes the messages within categories which will be useful for the front-end part to return get and return the messages to the user

First, each warning message had to obey to a certain syntax to be retrieved by the meta logger script : 
`Warning: [class::method] [[type]] message`
This syntax is based on the common form of OSG log messages, containing both the class and the method's names from which it is emitted, but also including a structure adapted for our purpose : categorizing the type of data the warning is about.

For now, these logs are used to keep track of normal problems encountered in the uploaded models. A feature is currently in development and will add an intermediate *draft* step between the moment he file is processed and the moment it is visible on the website. This intermediate step will be useful for the user to spot eventual issues and have a degree of freedom before having its model published. The processing warnings will be output to the user at this step, allowing him to reprocess or correct the model himself. 

The following code is responsible of parsing the processing log file, retrieving the warning message and logging them into the JSON file.


```
function parse_processing_log() {
    Expects the log format : "Warning: [class::method] [[label]] message"
    Removes the "Level: [class::method]" before calling metalogger.py

    if [ -e "${LOG}" ]
    then
        grep '^Warning:' "${LOG}" | sed 's/Warning:\s*//' | sed 's/^\[[a-zA-Z0-9 :()]*\]//' | while read -r line ; do
            ./metalogger.py "${metajson}" log_warning "${line}"
        done

        grep '^Anomaly:' "${LOG}" | sed 's/Anomaly:\s*//' | sed 's/^\[[a-zA-Z0-9 :()]*\]//' | while read -r line ; do
            ./metalogger.py "${metajson}" log_anomaly "${line}"
        done
    fi
}
```

[Screenshot : JSON part with corresponding warning messages]

This cleaning code now used in production, but a bunch of models were still rendered with black faces issues. After this small project, I focused more on issues that are related to the plugins themselves. 

## Plug-in fixes ##
Plugins are not the main part of OSG, the majority of them was developed to enhance the 3D files support. Actual plugins were developed for personal use by the OSG community member and were submitted a little bit later to be integrated in the main repository. As a consequence, code is very different from a plugin to another and the data they manage are very closed to the specific use the developed needed on the moment of the development. Thus, they are not close to the specification of the format they are used to read, and may have some buggy or week parts that can easily fail or break. 

With the huge variance in the sources and in the data contained in the files that Sketchfab receives, these plugins need to be improved to handle the more edge cases as possible. This what I aimed to do for the following tasks of this part.

### Bad vertex colors for STL files ###
I firstly focused on the plugin used to read STL files. A part of the models it output had black faces issues. The common approach for this type of issue is to first check the format specification to understand the structure of the files and the data that is contained. Then, the osg has to be check to see if it fits the specification of the supported format. This allows to dissociate the cases of badly exported 3D files with the case of a buggy plugin code which needs to be fixed and improved. As said previously, the variance is high at this step and a lot of uploaded files don't have consistent data or are not conform to the specifications. In this last case, the plugin doesn't have to handle it, and this is out of the responsibility of Sketchfab.

#### Looking for the file specification ####
STL (for STereoLithography) is a file format developed by 3D Systems for their stereolithography CAD software. Now, this 3D file format became a standard widely used for rapid prototyping and computed-aided manufacturing. STL is also widely used in 3D print because of its simple structure and the way it declares geometries.The STL format specifies both ASCII and binary representation, but the second is more common since it is more compact. 

Within the ASCII representation, 3D Geometries data is given between `solid/endsolid` bounds, with a sequence of triangular facets. For each facet, a facet normal is given, followed by the three positions of the vertexes that define the faces. On of the major drawback of this format for rendering purpose is that normals are given per faces (and not per vertexes) which is not in adequacy with the per-vertex mapping in OSG : STL are faceted.

[Illustration : screenshot of a simple STL file]

STL doesn't support any other feature such as materials, transforms or relationship between objects. However, contrary to ASCII files, binary ones can handle vertex colors. Given for each facets (such as normals), colors may have different meaning according to the 3D tool they come from. For common softwares, STL colors are encoded on 16 bits, with 5 bits for blue, green and red, using the last bit to indicate if the color is valid (i.e must be used) or not. 

Nonetheless, Binary STL files' structure may. differs from a software to another. For *iMaterialize 's Magic* software, the last 16 bits are differently used : they code the RGB color on 15 bits (so reversed compared to the common case) and the last one has another signification. The purpose of the last bit in the color field is to indicate if the given facet uses its own color or the main one which is declared in the header.

##### Finding the issue #####
Looking at the model in shadeless rendering (without lighting), parts were still black. After having asked about the shader equations, It seemed to be related to bas vertex colors, and not the normals as I thought first. Moreover, all these files were binary encoded STL.

[Code : shader equation with vertex colors component]

Normals are used within the shader to render face's illumination, but in the case of a shadeless render, lighting is not taken into account. According to the shader equation, bad vertexes colors (blacks for the worse case) have a very high weight on the final color.

The fix consisted on first, generating the right vertex colors according to the type of software the file came from, and setting these colors per vertexes and not per faces are it was originally made. Binding vertexes data per face (said per "primitives") is more and more deprecated in favor of per vertex binding. 

[Illustration : corrected STL]

Fixing this kind of plugin issues was a good introduction to understand the process of retrieving 3D data from file and formating it in order to be output through an OSG object. Data retrieving issues were pretty easy to fix when the whole process is understood. I managed to work on higher level issues that were related to the topologies. By topology, we define the layout of primitives that compose a 3D geometry.
Working on OSG plug-ins led me to deal with higher level, not related to vertex data but concerning the geometry itself.

### Geometry issues ###

##### Primitive sets #####
Basically, the render of an geometry is called by sending a set of vertex buffers to the GPU to be rendered. Buffers can be of several types according to the type of data they contain : normal vectors and position that are 3D vector, or vertex colors that are represented with 4D vectors (for R, G, B, A).
Buffers can also contain texture coordinates (2D vectors) or vertex attributes, that can be used to store tangents, binormal vectors, or any relevant data for rendering the geometry.

There are actually two types of structures to send this data. Draw arrays consists of sending buffers that contains all the values for each face. To reduce the size of these vertex buffers, OpenGL allows to use indexation. 
If a set of faces share vertex, their are not duplicate as in draw arrays, but the vertex data is unique in the buffers and each face will reference it through its indexes.
A primitive set is composed of one of these two structures, the type and the number of primitive to render. The following picture shows the primitives types that are supported by WebGL : 
[Illustration primitive types : webGL_primitives from http://math.hws.edu/eck/cs424/s12/notes/jan20.html]
POINTS, LINES, LINE_LOOP, LINE_STROP, TRIANGLES, TRIANGLES_STRIP, TRIANGLE_FAN

For each tuple of data, the GPU will draw the corresponding face, but introducing an offset in the indexes has huge consequences on the result.
I also had to fix this kind of issues with OSG plugins.

A topology issue was reported by a user which used to upload furnitures using the 3DS file format. The given model had some part that were broken.

[Illustration : broken model]

####3DS plug-in fix####
[Illustration : 3ds icon]
3DS if one of the file formats which is used to export 3D scenes from 
Autodesk's 3dsMax software. It was the native file format of the old Autodesk 3D Studio DOS. For legacy reasons, 3Ds is very limited on the data it can contain. Normals are not written, smoothing groups are used instead to help the software to generated them as good as possible. Only triangles are supported (so plug-ins may handle tessellation), and a mesh can't have more than 65536 vertexes or polygons. 

Importing the file in 3dsMax or Blender was fine, but this doesn't give much more informations since the importer can modify the data between the moment of it reads the file and the moment it loads it in the software. Doing this has sens since warnings messages can be output by the software and give some details about the potentially corrupted data.

As said previously, in OSG the loaded object needs to have its normals declared per vertex, but the 3Ds specification doesn't handle this structure. The plug-in needs to generated them by its own. The issue appeared to be a side effect of this process. When a shared vertex needs to have two different normal values, the plug-in just split it and remaps the faces indexes accordingly. The problem is that the code was written to fit the 3DS specification which doesn't allow more than 65536 vertexes or faces per mesh, so data types are used in consequence : index are short int. Introducing a duplication in this case is dangerous since it will potentially increase the number of vertexes, thus the indexes, and finally lead to an overflow. This bug was not really obvious on the first look, I first thought that it was the result of a tessellation operated on a bad geometry.

In this case, duplication made sens since normal has to be consistent within the whole geometry. A vertex cannot have its normal deviating too far from the normal of the adjacent faces. So the fix had to be done on the types, by changing shorts to int. Primitive sets also needed to be modified. OSG differentiates three types of draw elements objects, according to the indexes data types : *DrawElementsUByte*, *DrawElementsUShort* and *DrawElementUInt*.

Result : 
[Illustration : good result]

#### Issue with shared arrays ####
Even after having solved the 3DS plugin, these geometry issues were still found with some models. This looked to be an isolated case since I only had one sample to work on. Opening it in the 3D software was fine, but there was something in the pipeline that cause the geometry to be broken. The only thing that showed up was that the model had bad normals and was smoothed by the cleaner during the processing. 

To determine which stage causes the issues, we use to disabled step by step the processing operations. Most of them affects the geometry, in its vertex data or its topology, so I looked that one of the step was responsible of the rendering issue. Doing this, it appeared that disabling the cleaner solved the issue, but It was not obvious since it only has to deal with the normal data, not the topology. The only part that was not fully *under control* was the smoothing visitor step, so I took a look into the code.

The geometry had in fact shared buffers for texture coordinates. 
Texture coordinates art part of vertex data and are used to determine the way a texture is mapped onto a surface. Since OSG doesn't support multi texture materials, when several textures are mapped onto a geometry, its vertexes need to have a texture coordinates array for each, even if the values are the same. To avoid unnecessary duplications, both buffers share the same set of data.

Looking back to the behavior of the smoothing visitor, it duplicates vertexes if their adjacent faces have normals that deviate too far from each other. During the duplication step, vertex data are duplicated and indexes are updated, but it appeared that the existing duplication code didn't take into account the case of shared arrays. Managing two times the same set of data (for each texture coordinates buffer) introduced an offset when the remapping was done on the new vertex buffer. The solution was to deduplicate temporally the shared data before the vertex duplication and restore the state right after the process. It led to a very useful fix that was merged into the main OSG repository.

In this step, I had a great example of the issues side effects can cause.

The next part of the work done on the plugins had the purpose of improving the supported features and more generally improving the way plugins retrieve the data from 3D files. I essentially worked on the FBX file format, which is one of the most powerful and complete. Sketchfab aims to use it as its standard file format since it is very rich.

### Other fixes ###
The fixes I previously detailed were not the only I worked on. I also worked on small fixes and improved existing code in the plugins when It was needed or when it was worth enough. These works were about refactoring or changing the way operations were made (such as the normal retrieval or computation within each plugins), or simply cleaning the plugins of unwanted or buggy behaviors. For example, I improved the way options were retrieved within the STL plugin, and added an option to avoid the OBJ plugin to reverse faces when the normal was not consistent (that led to rendering issues to). 

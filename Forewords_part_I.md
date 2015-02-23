# Foreword #
## Why Sketchfab ##

At the end of my 5 years at UTBM, I wanted to put a first step in the 3D area, for which I have a real interest. After having discovered it through my first internship at Voxelia (Strasbourg), I became to self-learn the basics of 3D through books, modeling with 3dsMax and trying to understand how all of this works. This is during this period that I heard about Sketchfab, this really interesting 3D start-up, mixing 3D engineering and web and having offices in both Paris and New-York. Sketchfab was my first target while I was looking for my final internship. I was also very interested by the environment the start-up was involved in. Working to make 3D artists able to share their awesome work, companies that want to share models of their new products or prototypes, and 3D printing which is in current democratization. After an interview and some mail exchanges, I finally had the opportunity to work for Sketchfab.

## About the report ##
This report summarizes the 6 month spent at Sketchfab in Paris as 3D back-end engineer, from September 2014 to February 2015. I have discovered, learnt and deepened a lot of concepts around 3D rendering and 3D data processing that are required to explain the context of the different tasks I made at Sketchfab. Thereby, the report may have a more important part of materials as usual, to be more easily understandable and readable regardless of the technical knowing of the reader.

It begins with an presentation of the company, trough its history and its product, before introducing the internship and the main goals I set myself when starting. The following part will focus on the environment of work, the tools and the main workflow that are used at Sketchfab, before switching to the third part which explains the work done during the 6 months. I also took the freedom to make a dedicated part to mention the external activities I have participated in, before ending with an overview of the results, the observations and concluding.

# Introduction #
## What is Sketchfab ##
### History : from WebGL to Sketchfab ###

At the beginning of the second trimester of 2011, the web sphere made a step forward with the apparition of WebGL, allowing 3D rendering in the web browser without the need to set up any plugin. It was developed and released by the Khronos Group, consortium formed in 2000 and known for the famous cross-platform computer graphics API OpenGL. WebGL is a JS API for rendering 3D graphics into HTML5 canvas elements, and is based on OpenGL ES 2.0, an adaptation for Embedded Systems. 

3D models are medias that are difficult to share : they require the installation of specific plugins, or in the worst case, the user needs to have the 3D software (used to create the models) installed in its computer. So far, 3D artists had to render their work in 2D into images files, or render a lot of frames and build a video sharable on Youtube or other broadcast websites. In both cases, it is difficult to see the model as a 3D media, including rotating around, zooming, and having real-time effects like reflections. Such off-line renderings have the advantage of giving very good results since there is no frame-rate constraint, but they have the drawback of being static.


> "We bring 3D to the web"
[illustration : screenshot of the website]

With this need and the apparition of the native support of 3D in web browsers, 3D passionate Cédric Pinson released ShowwebGL in 2011, what we know now as Sketchfab. ShowwebGL has a very simple workflow : the artist just needs to upload a model using one of the supporter file formats, then he receives an url allowing him to share its work. At the beginning, Showebgl accepted a lot of 3D file formats (almost 20), and handled textures that were packed with the model into .ZIP archives. 

Later, Cédric's prototype gained a real interest for Alban Denoyel, sculptor and passionate by 3D printing, who joined him and put its business skills into the project. In March 2012, they create Sketchfab and are joined by Pierre-Antoine Passet (as CPO) six months later. At the end of the year, they integrate the Camping, a startup accelerator in Paris. 

### The 3D sharing platform ###

> "The Easiest Way to Share Your 3D Models"
[illustration : screenshot of the website]

Today, Sketchfab counts around 150K users and around 250K models with a few ones which have exceeded 1 Million of views.
The workflow remained pretty simple. After having created an account, the user just have to click on the "Upload" button, choose a 3D file and upload it. During the upload, he has the possibility to fill data fields if he wants to add informations about its model. He can provide a description, giving details about its workflow, the reason which motivate him to make the model, or what the model is made for. He can set a category for the model, and also add some tags to identify the content type and/or the tool used. 

[Illustration right inlined : upload popup]

To improve the exchanges between users, a feature was recently developed to allow Sketchfab members to make their models downloadable, while defining the rights and their own price. With the democratization of 3D printing, it is very interesting for Sketchfab to be used as a 3D model database.

Basic account doesn't allow to upload files whose size exceed 50 Mo. To overpass this limit and also have access to more features, Sketchfab proposes two types of paid accounts. 
The PRO account moves the size limit to 200MB, allow the user to set its models as private. Private models are not shown in both the global feed and the user's page, they are only accessible through the URL. The user can also restrict the access to the model with a password. Some edition feature are also available, such as the use of annotations (pins with caption that are mapped to the model, up to 20 per model), the upload of custom background or basic viewer settings. Pro users can also use Sketchfab as their portfolio with a pre-build dedicated page. 
To answer the needs of companies that base their products on 3D models, such as architecture or prototyping, a BUSINESS account allows bigger models (up to 500 MB) and advanced customization. 
PRO and BUSINESS accounts offer support in 24h and 12h respectively.
[illustration : include icons PRO and CORP in the text line]

In the community side, a forum was created to improve exchanges and conversations within Sketchfab members. It is a good users can ask for support or find the help they need, explain or get informations about workflows and techniques, and keep track of the Sketchfab's news and the events that are planned.

[Illustration : screen shot of the forum with fall-of at the bottom]

#### The editor ####

After uploading the model, the user has access to a bunch of tools to customize and improve it's render through the Sketchfab built-in editor.

The first part is dedicated to the scene in its general aspects, with an interface to customize the camera, the background/environment, allowing the user to adapt the viewer to get the best result for its model.
The very popular *camera effects* feature can be used to define post-processing effects, which are often considered as *Instagram like* filters affect the whole rendering.

The second tab contains the material settings. The user can tweak separately the scene material through a set of parameters. To be edited, the material can be selected trough the drop-down menu or by picking (double click) on one of the geometries in which it is applied to. This part is very useful since most of the time, the artist switches from the off-line render of its authoring tool to Sketchfab's real-time renderer. This implies a lot of changes due to the specificity of the 3D software (specific assets), and the performances constraints for real-time rendering. Moreover, Sketchfab offers built-in assets, such as backgrounds and environment textures (which are not so easy to use) to customize the environment around the model. 

The third part is dedicated to the management of annotations. As said before, this feature allows the user to set a bunch of points (called hot spots) which are points on the models that the camera can snap to, with a text field to put comments or notes about the focused point. Annotations are very useful to give details about some specific parts of the model, their story, meaning or just to build a path to run step by step through the scene.
[Illustration : sample of illustrated model]

After that, if the user is not happy about the render of its model, or if something is wrong with its 3D scene, he has the possibility to re-upload it, or delete it.

#### Browsing models ####

Including a browser, the website has a lot of part dedicated to the community, and the models feed. Models that have a certain level of views and likes are pushed up to the "Popular" section, including the "Models of the week" and some related categories. The best models can also be "Staff picked" by the Sketchfab team, which mean they are at the top of the feed.

#### Who use Sketchfab ? ####

Sketchfab is for anyone who want to share its 3D work. Architects or 3D artists from the video-game industry, design students or simple hobbyist, Sketchfab users are very diversified and come from all around the world. 3D printing companies use it to share their printable models, artists use it as portfolio to share their amazing work. Moreover, a lot of 3D authoring tools are natively plugged to Sketchfab through exporters or plugins, and several news or art websites include Sketchfab's embeds in the content of their web pages.

### The Team ###

[Team organigramme]

## Presentation of the internship ##

### Context of work ###
At the beginning of the internship, the subject was not very precise but it concerned all kind of work that can be done from my job, as 3D back-end developer. For the main part, it involves all the operations that are applied to a given model right after its loading. At the global scope, the 3D back-end is about all the pipeline the inputs pass through before being stored in the database and being able to be shown on the website. It is composed by 3 main parts : 

**File reading and data parsing :** The first stage the uploaded file passes through. It is globally managed by the OSG toolkit (detailed further in this report), and allow to both retrieve the file's data and generate metadata for the front-end part. 

**Data cleaning and normalization :** since the platform is public oriented, we have to deal with files that come from a lot of sources, with a lot of file format that respect more or less the specifications. Through this second step, the retrieved data is cleaned (value clamping, various correction) to ensure that the data is consistent before applying the operations of the third step.

**Model compression, preparation and optimizations :** This last step has the function to prepare the data to be rendered using WebGL in the client side. Since WebGL is based on openGL ES, it inherits all the constraints that are imposed by the development for embedded systems. In addition, web browsers are also limited about the data they are able to manage. Finally, a such web platform needs to lower as possible the quantity of data to send to the clients. To reduce the loading times and the amount of memory used, and also improves the rendering performances, a set of operations including deduplication, compression and optimizations, are applied to the data.

So the main subject of my internship was to work on these different parts, with the help of the back-end engineer.

### Prior Experience ###
What really helped me to get started working at Sketchfab is the ST00 I made the previous semester. During this personal project, I self-learned about the basis of 3D real-time rendering from both programmer books and 3D softwares. It was very important for me to have an big picture view of what is 3D actually is, because it allows to bypass the first feeling of complexness and abstractness about 3D. By working on both 3D engineering and artist sides, I had a useful feedback of how all of this works, and I can now better anticipate the result of what I do on the data. Practicing myself 3D modeling, I am also a Sketchfab user.

### My goals as Sketchfab intern ###
I firstly choose to do this internship at Sketchfab to begin to work in a 3D based product. More than acquiring work experience and improving my programming skills, I aimed to learn the more basics as possible about 3D rendering and models processing. The other motivation I had was to get and understand the "big picture" view of a 3D web platform and all the community part it involves. From a more personal point of view, I was interested by learning more about 3D modeling and texturing, which is easier when having the opportunity to work and share with artists and hobbyists. 

At the end of my studies, I am very interested by building my career as 3D software engineer, so I expected this to be my first start-up in the area. 

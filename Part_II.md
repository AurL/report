# Working at Sketchfab as 3D back-end engineer#

## The development environment ##
Every Sketchfab developer works on a virtual machine where all the required tools are built and run. Sketchfab uses Vagrant to host and managed virtual machines and a recipe software to set-up the environments. A local server is also run by the VM to be able to work on a local version of Sketchfab (very helpful for testing purpose).  

At the beginning, I had the possibility to choose the operating system with which I wanted to work. According to the compatibility advantages and the productivity I can have thanks to my habits, I chose to work on Windows. As a student, I have access to a lot of free 3D software licenses from Autodesk (3DsMax, Maya) and also Unity. Under Windows, I was able to use these softwares in my work, and also use my knowing about them to help other developers that would potentially need some specific files or sample.

Since the VM has no graphical interface, I firstly worked using *vim*. A little bit later, I chose to move use *Sublime Text* (which is more user-friendly and have a lot of very helpful features) in parallel with a SFTP protocol to make the bridge between the host and the virtual machine.

## The workflow ##

### Project management and versioning ###
The project is principally managed using the Pivotal Tracker. Developed by Pivotal Labs, this software is an agile project management tool used to manage tasks, host files, track velocity, and plan iterations. Through tags and labels, it is easy to keep track of the work done, the work which is currently done, and the tasks that are in the queue. 

Github, the well known web-based Git repository hosting service is used as versioning tool within Sketchfab. It provides a bunch of interesting tools for source code management and revision. 

To improve the traceability of the work which is done on the project, Github and Pivotal Trackers are linked by the task IDs. 

### Workflow template ###
When beginning to work on a given subject, the corresponding task is started. The task is commented when needed, to discuss possibilities, decide which solution to use and everything that can be relevant. When a task is finished, the corresponding Github branch is used to create a Pull Request, waiting to be reviewed and commented by some members of the team. This step is primordial to spot potential bugs, discuss and propose improvements when needed. From a personal point of view, these Pull request were very a helpful source of learning. Comments were very useful to know what are the better ways to do things, what is not to do, and so on. I have learnt a lot of good practices thanks to these reviews.

#### Pull request validation ####
For each pull request and after the code review, all the processing tests are run on a dedicated server to check if it can be merged without any problem. Jenkins, a continuous integration tool, has the responsibility of running the processing tests. When running a test on a OSG pull request, Jenkins checkouts both the OSG (work) and the ShowwebGL(tests) branches, updates its model base if new models were pushed into the model test repository. Then, it builds the tools (OSG) and set the environment before running all the processing tests. 
Tests are firstly run in local within the virtual machine to spot bugs. Jenkins test are then run to spot potential specific issues related to the environment or out-of-date plugin or tool. Jenkins is also useful for several tests that require a lot of ressources and that can't be tested using a local machine. 

#### To the production ####
Once the pull request is accepted, it is merged into the *develop* branch and the story is accepted. *develop* is used by every developer so it provides a first testing step. When a release is in preparation, the *develop* branch is built on a staging server which constitutes a second level of tests, with conditions that are closed to the production. After having tested on staging server, the code is pushed into production during the release. Everybody is asked to test the new features to spot potential bugs that were not spotted in the previous testing phases.

### Communication inside Sketchfab ###
#### Internal IRC ####
To improve the team communication and also reducing oral conversation which might disturb the other members of the co-working space, Sketchfab works with an internal IRC called Hipchat. It was very useful, especially to be aware of what is going on in the team, to share interesting articles, links or files, and ask for help if needed. It also permits to keep track of the conversations, and browse for missed or informations if needed.
This point was very interesting and useful in my opinion, because it made me hear about concepts and techniques that are not directly related to my work, but are relevant for the general culture and the context of 3D development software. It was like a passive learning tool, and I really appreciated it during my time at Sketchfab.

#### Weekly overview and meetings ####
Two meetings are planned on Monday. The first one is done in the morning and had the purpose of doing an overview of all the work that was done during the past week, ask question, highlight the encountered problems and discuss about what to do for the current week. They are the opportunity for each one to discuss, ask questions or propose solutions.

A team meeting was also planned each Monday afternoon, in video conference with the NYC business team, to talk about stats, the community work for the business side, and the features, the contents of the release and the incoming features for the technical side. 
I was also the moment to talk about the events, the feedbacks and all the other subjects that were done or improved at Sketchfab.

### Feature Friday ###
During the last part of my internship was introduced the "Feature Friday" work sessions. The concept is to dedicate the last hours of the week on a project that mixes both personal and company's interests. During this time, we are  free to experiment and work on a subject we are interested in, make some tries or prototyping, if it respects some rules. The first constraint is temporal and asks for the feature Friday to be realizable in one or up to two sessions. The goal is not to spend a lot of time on it since the subject out of the company's road map. The second rule is that the project must be "integrable" within Sketchfab, or at least represent an interest for it. 

# 3D back end work with OSG #
The beginning of the internship was a bit confusing for me since I had to understand and assimilate the Sketchfab's pipeline in its overall. Being able to place my work and myself in this process was primary to do work efficiently.

## Presentation of the pipeline ##
[Illustration pipeline with file screenshot and evolving small 3d model]

Right after the upload, the 3D file pass trough a pipeline before being stored in database and referenced to be showed on the website. This is actually a sequence of operations and processes whose goal is to read, correct if needed, and prepare the model to be registered and well rendered in the viewer.

Briefly, this pipeline is divided in three main parts :
    
1.  **Data retrieval and cleaning**
	This part reads the file using the right plugin and writes them into a .osgb (osg binary). Textures are listed and data are cleaned to be consistent for further processing steps.
    
2.  **Generic model building**
	The generic model contains the original data of the input file, rearranged in a more generic way. Its role is to have a homogeneous

5.  **OSGJS file building**
	Here is handled the conversion from the generic model to a OSGJS model which is easily readable by the JavaScript side, in order to be rendered. 
    A lot of operations are applied to optimize and compress the data to be rendered using WebGL.

## Technologies and concepts ##
The whole pipeline is a set of scripts that are written in Bash and Python.
These scripts are used to operate on the files, making them to pass through the adapted processed, and to manage the use of the different tools.

As base tool to manage 3D data, Sketchfab uses the OpenSceneGraph API. Open-source and cross-platform, this toolkit is written in C++ and distributed under a LGPL like free license. Based on OpenGL, OpenSceneGraph is a rendering-only tool, it doesn't provide higher functionality to be used for gaming purpose for example. OSG renders a 3D scene using a scene graph structure and offers a lot of tool to manage and optimize the data.


##### Some words about scene graphs #####
A scene graph is a data structure representing a 3D scene. It contains all the spatial and logical relationships between the scene object, leading to an efficient management and rendering of graphics data. The scene graph can be seen as a hierarchical graph composed of nodes, with a top root node linked to child nodes. In a scene graph, the transformation applied to a node also affects its children. This structure make it easier to transforms objects and create animations.0

## Place and responsibilities ##
As back-end engineer, my work concerns the whole pipeline but I mostly worked on the reading and cleaning parts. Working with OSG plugins, I had to ensure that the input file was correctly read and processed without missing or misreading data. The variance of the input is high at this step, so I had to manage a lot of edge cases that can make the processing to fail. We don't have any control of the data that are actually sent by the users, but the main goal is to correctly return the model if the file is consistent. If it is not, the goal is to be able to identify clearly what is wrong with the file and return an accurate error message to the user.

OSG plugins are used to read the input 3D files coming from the Sketchfab users. With the apparition and the development of 3D editing tools, some new features and data are added to these 3D file formats, in a specific or a more general way. The majority of the OSG plugins were written to read 3D files in a basic way, without handling all the data the file can contain. In the best cases, data are missed and the model didn't shows as expected in the viewer, and in the worst ones, the processing failed because of inconsistent or mishandled data in the files. Moreover, the front-end may need some specific meta data that are not useful for 3D rendering (so no parsed by the plugin) but are relevant in some cases to improve the model settings or the user experience.

## Retrieving models from Sketchfab ##
In addition to the issued models returned by the support, Sketchfab has some mechanisms to retrieve issued models from the production database. If a model is bad rendered, we can ask for it to be rsynced (copied from a server to another) copied on a server from which we can retrieve it. This system allows to identify the problem the user had, and to have useful resources for testing purpose. Moreover, a dedicated channel was set in Hipchat to monitor the activity on Github, Pivotal but also on production servers. When a model fails to process, an error message is shown in the channel with the Id of the model and the command to retrieve it. 

I managed these different sources to work efficiently and test my code. I also used to exchange with the community support to help in identifying more accurately the problems and explaining how to fix it, when is was possible.

## Public and specific code ##
Sketchfab uses contribute on OSG by reporting the changes made on the forked OSG repository. Including the changes into the main repository has the advantage : it makes it easier to update it by reducing the deltas between both versions. On the other hand, a part of the added code and behaviors is very specific to Sketchfab and needs to be distinguished and kept away from the submitted one.

This *dual* workflow adds a level of difficulty since these two parts need to remain separated. To facilitate the submission, the specific code has to be aggregated, that requires the developer to adapt the structure and also the logic of the new code. Moreover, the weight of the public modifications needs to be as minimized as possible to be more easily accepted and then merged into the main code. In some cases, it can require additional work and time in order to lesser the impact of the modification while keeping the code clear, optimize and logical. 


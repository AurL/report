
# Working on processing tests #
In order to avoid regressions or breaking existing behavior by side effects while merging a new Pull request, Sketchfab uses processing tests.
For all the work I made, in whether OSG or exporters, I also created tests to be run with all the existing ones in order to validate that my code doesn't break anything and works as expected (by asserting the expecting results). They will also be useful for further changes to check that these behavior are not affected. For example, It was very useful when I had to make huge refactors of the OSG plugins.

In this context, each pull request I submitted to Sketchfab's repositories had a corresponding one on the tests repository. 

### Tests structure ###
Basically, a test is composed of a model which is processed normally, followed by a set of assertions that check that the files were generated correctly.
Assertion are used to check both the file existence, and the processing logs that are output. The whole processing outputs logs and warnings into a 'processed.log' file, which contains all the detail of what was done during the entire process. The relevant part of this data is extracted and written into a JSON file, which is read by the front end code. 

#### Testing behaviors and logs ####
For testing purpose, this file is read and assertions are applied on it to validate or invalidate the processing tests. The messages that are logged into are used to check processing behavior. For example, when running the smoothing visitor on a geometry during the cleaner step, a warning is written into processed.log and reported into the JSON to be parsed. The presence of this warning message testifies that we passed into the cleaner and applied the normals generation because they were not consistency. Stats and textures related data are also written for the same purpose. The other role of this file is to have the data to be able to report more accurate warnings to the user when the front end will handle this feature.

#### Testing 3D data ####
A lot of tests are also made on the 3D data, running assertions on the output *OSGJS* file. OSGJS is a 3D file format coming from the OSG.JS WebGL framework, whose purpose is to offer an *OpenSceneGraph-like* toolbox to interact with WebGL using JavaScript. To make assertion on the 3D data, this file is read using a Python script (JSON parser) which allows to retrieve the relevant data using parameters:
    
    - Filter : the path that leads to the wanted JSON node

    - having/where : allow to retrieve nodes that fulfill a given query. *Where* only keeps nodes where the query is fulfilled. *Having* is more permissive : it keeps the parent node having a subnode fulfilling the query.

    - select : allows to refine the selection and only return the interesting part.

Here is an example of the test I wrote for the *blender-preprocess.py* script :

```
# Test the blender preprocess script
# The sample contains an armature, parent of three geometries:
#  - A cube having a name with latin-1 characters and two modifiers, invisible in the current scene layer
#  - A cylinder visible but in the layer 4 (so invisible in the current scene layer)
#  - A torus and a Cone sharing the same material
# It also contains two cubes having two booleans, whose operand are the cone and the cylinder.
# The whole scene has only one material defined.
# We check that the cube is renamed, the invisibility of the scene is detected and all models are then visible
# We also check that meshes are duplicated (to allow modifier application) and that the Armature is skipped (because disabled)
# We finally count the number of (named) materials to ensure that we don't duplicate it

process_test_model "${model_dir}/preprocess-test.blend"
assert_grep "SUCCESS" "${url_dir}/processed.log"
assert_meta_contains "warning.generic" "Meshes were duplicated to allow modifiers to be applied"
assert_meta_contains "warning.generic" "1 entities having misencoded names were renamed"
assert_meta_contains "warning.generic" "The scene has no visible objects"
assert_meta_contains "warning.generic" "All the hidden meshes were set to visible"
assert_meta_contains "warning.generic" "Error while applying disabled modifier Armature (removed)."
local geometry_count="$( ./json_parser.py --filter osg\\.Geometry --having '{"Mode":"TRIANGLE_STRIP"}' --count ${url_dir}/dist/file.osgjs )"
assert_equal "${geometry_count}" "6"
local material_count="$( ./json_parser.py --filter osg\\.Material --select Name --count ${url_dir}/dist/file.osgjs )"
assert_equal "${material_count}" "2" # The root material + the shared scene material 
```

# External activities #
During the internship, I also had some activities around Sketchfab's community. For example, I had the opportunity to participate in some meet-ups with artists and other 3D start-up that were interesting in both cultural and human aspects. Moreover, always outside of my engineering job, I managed the print orders Sketchfab had through the 3D Hubs platform. 3D Hubs allows to find owners of 3D printers in a given geographic zone and order 3D prints. Sketchfab was registered as one of these owners (hubs) and received several orders during the 6 months I was there. I had to download from the website and then prepare, run the print and check with the client to deliver the product. 3D printing became something very interesting for me, and I really appreciate what it brought to me. 
In parallel, I participate in community events where I made 3D scans of people. I went to the 3D print show which took place in Paris in October, to represent Sketchfab, scan people and also get informations about that was actually done in the 3D printing area. 
Finally, during my spare time, I created a few models for Sketchfab, to be used for Christmas or New Year cards.

# Summary of the results #

## OSG ##
The work I made on the plugins made the rendering issues decrease significantly on Sketchfab. The huge part of STL models used to be printed are now rendered without any issues. Models with normal data consistency issues are now handled and automatically corrected within the pipeline to be rendered correctly (this concerns about [part of smoothed models] models). 

FBX files are know better supported by Sketchfab and are much more easy to manage. Wireframes are kept, and the majority of the used maps are retrieved and automatically set within the editor, which improves significantly the user experience. Moreover, data are more efficiently retrieved since we don't duplicate it anymore. Scene graphs are lighter, since a lot of data is now shared. Materials are easier to edit, and tangent spaces are not systematically recomputed. 

At a lower level, OSG plugins were refactored and updated with a cleaner code. The updated versions were submitted to be merged into the main repository so users can enjoy the new supported features.

## Blender ##
The work I made for Blender has very interesting results too. We reprocessed all the failing models we had using the new version of the Blender pipeline, and almost 75% are know correctly processed. Blender has a huge community and blender artists often make very nice models, so this better support is interesting for Sketchfab.

## Kerbal Space Program ##
The KSP importer is usable and is able to import efficiently a complete spaceship without any trouble. We are waiting for news from the author of the mu importer and from the KSP community to have a bunch of models to test before communicating on it. At its final step, the importer will bring a lot of new users on Sketchfab and make the community to growth. I may have the opportunity to continue working on it to make it more efficient and maybe fix edge cases that might be highlighted with the coming samples.

# Observations and personal gain#

## Work and workflow ##
The first thing that I learnt during this internship was the gain in efficiency the use of the different tools provides. I didn't used such tools during my last internship, so this one was very interesting on this points. Using them is very helpful to adopt good techniques and good behavior to have when working on a project with several developers. Code review are a very good source of learning, and more generally, the workflow gives a good overview of the main steps that required for a good project managing.

## Communication ##
Working at Sketchfab also highlighted the importance of the communication, and how to communicate efficiently while working. Even if it looks straightforward, asking the good questions and being understandable can be difficult. Moreover, communicate is very helpful for efficiency : it allows to gain time by checking that the provided solutions are *approved*, and also to have advices from other developers, who can for example highlight some specific cases that we don't necessarily have in mind.

## Importance of tests ##
Another thing that I learnt by working at Sketchfab was the importance of tests in the software engineering process. In a few case, they helped me to spot side effects that I didn't though about while modifying some snippet of code in OSG. I also learnt how to write good and efficient tests.

## Coding ##
As mentioned before in this report, code reviews were very helpful for me. I received a lot of advices on *how to do things clearly and efficiently*, with a lot of good coding behaviors to have (and not to have). I also really appreciate to have discovered and began to learn Python language through my work with Blender. I also learnt when and how to refactor efficiently an existing code, and the importance of refactoring in general.

## 3D ##
Finally, I gain proficiency in 3D engineering trough the work I made at Sketchfab, as I expected at the beginning. I also learnt a lot from the conversation I had from other team members. Moreover, the environment was very interesting and helpful for discovering and learning. Knowing and understanding real-time rendering mechanism brought me a lot of skills and techniques for my personal 3D modeling practicing. I am able to anticipate and know what I have to do to reach a given result.
Finally, I had the opportunity to hear about advanced concepts, such as Physical Based Rendering, which are currently appearing and implemented within a lot of software, or more advanced real time rendering features. These will be very helpful for my further jobs. 

Conclusion :

Looking back to the beginning of the internship, I really feel that I have made a huge step further in 3D software engineering. Even if I regret a little bit to not had worked on advanced features, because of the lack of skills in the area, I acquired a strong basis in 3D software development, that I am now able to deepen by myself or during my future jobs. Sketchfab has taught me the good way to think, to communicate and to approach efficiently a problem. I also acquired good coding and working rules, and have an overview of what a such project can involve. I also really appreciate participating in the Sketchfab community and being involved in the environment and the company's culture. This experience was really enriching in both professional and personal aspects. To conclude, I think I reached my goals as intern.


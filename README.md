# Solar3D

## 1.Background
   Solar3D is a software application designed to interactively calculate solar irradiation and sky view factor at points on 3D surfaces. It is essentially a 3D extension of the GRASS GIS r.sun solar radiation model. According to the GRASS GIS documentation [1]:
   “r.sun - Solar irradiance and irradiation model. 
   Computes direct (beam), diffuse and reflected solar irradiation raster maps for given day, latitude, surface and atmospheric conditions. Solar parameters (e.g. sunrise, sunset times, declination, extraterrestrial irradiance, daylight length) are saved in the map history file. Alternatively, a local time can be specified to compute solar incidence angle and/or irradiance raster maps. The shadowing effect of the topography is optionally incorporated.”
   To calculate the solar irradiation over a certain time interval at a point on a 3D surface, Solar3D first derives the slope and aspect value from the surface normal, and then use a cube map to determine if the point is shaded at each time step for the entire duration. Instead of using the brute-force ray-casting which relies on ray-triangle intersection and therefore is computationally intensive, Solar3D generates a cube map-based panoramic view of the 3D scene at the point with sky and non-sky pixels encoded in different color values, and then determines if the point is shaded at each time step by looking up the intersected pixel in the corresponding cube map face (Figure 1). In this way, Solar3D can rapidly calculate daily to annual solar irradiation at arbitrary points in sufficiently small time-steps (smaller than one hour). However, a limitation with Solar3D is that it is designed specifically to rapidly calculate solar irradiation at discrete points and is not equipped with adequate performance to calculate for large areas where points are densely and uniformly distributed, and the main reason is that generating millions of cube maps is not computationally affordable.

## 2.Software Architecture and Business Logic
   The core framework is constructed by integrating the r.sun solar radiation model into a 3D graphics engine, OpenSceneGraph [2], an OpenGL-based 3D graphics toolkit widely used in visualization and simulation. OpenSceneGraph is essentially an OpenGL state manager with extended support for scene graph and data management. The reasons for choosing OpenSceneGraph are multifold: firstly, OpenSceneGraph provides user-friendly, object-oriented access to OpenGL interfaces; secondly, OpenSceneGraph provides built-in support for interactive rendering and loading of a wide variety of common 3D model formats including osg, ive, 3ds, dae, obj, x, fbx and flt; thirdly, OpenSceneGraph supports smooth loading and rendering of massive oblique airborne photogrammetry-based 3D city models (OAP3Ds or integrated meshes), which are already being widely used in urban and energy planning. Once exported from image-based 3D reconstruction tools such as Esri Drone2Map and Skyline PhotoMesh into OpenSceneGraph’s Paged LOD format, OAP3Ds can be rapidly loaded into OpenSceneGraph for view-dependent data streaming and rendering. The r.sun model in Solar3D also relies on OpenSceneGraph for supplying the key parameters needed for irradiation calculation: (1) location identified at a 3D surface; (2) slope and aspect angles of the surface. (3) time-resolved shadow masks evaluated from a cube map rendered at the identified position.
   The business logic of the core framework works in a loop triggered by user requests (Figure 2): (1) a user request is started by mouse-clicking at an exposed surface in a 3D scene rendered in an OpenSceneGraph view overlaid with the Solar3D user interface (UI) elements; (2) the 3D position, slope and aspect angle are derived from the clicked surface; (3) a cube map is rendered at the 3D position as described above; (4) all required model input [1], including the geographic location (latitude, longitude, elevation), Linkie turbidity factor, duration (start day and end day), temporal resolution (in decimal hours), slope, aspect and shadow masks for each time step, is gathered, compiled and fed to r.sun for calculating irradiation. The shadow masks are obtained by sampling the cube map with the solar altitude and azimuth angle for each time step; (5) the r.sun model is run with the supplied input to generate the global, beam, diffuse and reflective irradiation values for the given location; (6) the r.sun-generated irradiation results (in units of kWh/m2) are returned to the Solar3D UI for immediate display.
   To better facilitate urban and energy planning, the core framework is further extended by integrating into a 3D GIS framework, osgEarth [3], an OpenSceneGraph-based 3D geospatial library used to author and render planetary- to local-scale 3D GIS scenes with support for most common GIS content formats, including DSMs, DSMs, local imagery, web map services, web feature service and Esri Shapefile. With the 3D GIS extension, Solar3D can serve more specialized and advanced user needs, including: (1) hosting multiple 3D city models distributed over a large geographic region; (2) overlaying 3D city models on top of custom basemaps to provide an enriched geographic context in support of energy analysis and decision making; (3) incorporating the topography surrounding a 3D city model into shading evaluation; (4) interactively calculating solar irradiation with only DSMs.
   The code was written in C++ and complied in Visual Studio 2019 on Windows 10. The three main dependent libraries used, OpenSceneGraph, osgEarth and Qt5, were all pulled from vcpkg [4], a C++ package manager for Windows, Linux, and MacOS, and therefore Solar3D can potentially be complied on Linux and MacOS with additional work to set up the build environment.

Note: If Solar3D is not able to start due to missing runtime dlls, try to install the Microsoft Visual C++ Redistributable for Visual Studio 2015, 2017 and 2019 from https://aka.ms/vs/16/release/vc_redist.x64.exe.

## 3.Basic Usage and Workflow
   The Solar3D UI consists of three components respectively responsible for parameter settings, result display and status update. The parameter settings panels are located in the top left with UI elements for setting the Linkie factor, start day, end day, time step, latitude and base elevation overrides (used in case of a non-geoferenced scene). The result display UI consists of two panels located in the left side right below the parameter settings panel used for immediate display of feedback from the latest request and a pop-up panel used to display results at the cursor point. The status UI elements include a compass at the top right and a status bar displaying cursor and camera coordinates toward the bottom. 
   A first step in the workflow of Solar3D is scene preparation. The users are expected to prepare their own scenes with a least one 3D model and optionally some basemaps. An easy usage is to start Solar3D with the path of a single 3D model exported form CAD or an OAP3D exported from a photo-based 3D reconstruction software (Figure 3), but in this way, the scene will not be geoferenced and thus the users need to specify the latitude and base elevation override.
   To integrate 3D models into a geoferenced scene with basemaps (Figure 4), the users will need to follow the instructions [3] and examples (“Solar3D/bin/tests/”) provided by osgEarth. An inconvenience is that osgEarth does not offer a scene editor with a graphic UI for scene authoring, and instead, the users will need to manually add and configure scene layers in a text editor based on one of the example scene files (*.earth). An advantage with osgEarth is it can be used to author advanced scenes with 3D models overlaid on DEMs and DSMs distributed all over the Earth. Additionally, osgEarth also provides the ability to extrude building footprints from polygon features into 3D models for use in Solar3D.
   In a typical use scenario, after starting Solar3D with a single 3D model or an osgEarth scene, the user zooms to an area of interest with buildings on which PV arrays are planned to be deployed, and then irradiation estimates are obtained by interactively clicking at rooftops and facades to identify suitable surface areas for PV deployment. Solar3D processes calculation request upon mouse click (with Ctrl down) once at a time and typically finishes a request within a couple of seconds and displays the results immediately. A marker with text label will be displayed at the location at which a calculation request has been finished. The irradiation results obtained during a session can be exported in a batch to a comma-delimited text file for analysis. When exporting an irradiation record, the associated r.sun parameters and 3D coordinates will be packed in a single row.
    
 ## 4.Keyboard/Mouse Operations
(1) Add a point (start a calculation request).
Ctrl + left click at a surface in the scene, and the calculation will normally be finished instantly with feedback displayed in the left panel. All irradiation values are in units of kWh/m2. A point marker with a label will also be added at the clicked position.
(2) Undo Add.
Ctrl + Z
(3) Redo Add.
Ctrl + Y
(4) Show pop-up at a point.
Ctrl + Mouse Hover over a point marker
(5) Toggle display text labels.
Ctrl + T to hide/show the point text labels
(6) Hide/Show parameter settings/feedback panels.
Toggle the checkboxes in the top left panel
(7) Export.
Ctrl + E
All points will be exported in a .csv named with the current time stamp to the path “Solar3D/bin”. 
(8) Toggle scene lighting.
Key L.

 ## 5.Scene Preparation
   CAD models can be created in CAD software such as Autodesk 3ds Max, SketchUp and Blender and exported in a format that OpenSceneGraph supports [5]. OAP3Ds can be created using photo-based 3D reconstruction tools such as Esri Drone2Map and Skyline PhotoMesh and exported into OpenSceneGraph’s Paged LOD format. Building footprints-extruded 3D models need to be created in an osgEarth scene by apply 3D symbology to a feature layer. 
   To create an osgEarth scene, find a template under “Solar3D/bin/tests/” that best suits your needs and then make a copy and modify in a text editor. Table 1 is a selected list of examples demonstrating how to start Solar3D with different types of 3D models (OAP3D, CAD, building footprint extrusions) with or without osgEarth scenes (global or local). Solar3D DEMOs:
(1) Solar3D/bin/Example_OAP3D.bat. Start with an OAP3D model.
(2) Solar3D/bin/Example_CAD.bat. Start with an CAD model.
(3) Solar3D/bin/Example_Boston_Global.bat. Start with a global osgEarth scene. This example shows how to extrude build footprints from a shapefile.
(4) Solar3D/bin/Example_Boston_Projected.bat. Start with a local (projected) osgEarth scene. This example shows how to extrude build footprints from a shapefile.
(5) Solar3D/bin/Example_Solar3D_Global.bat. Start with a global osgEarth scene. This example shows how to position CAD and OAP3D models in a global scene.
(6) Solar3D/bin/Example_Solar3D_Projected.bat. Start with a local osgEarth scene. This example shows how to position CAD and OAP3D models in a local scene.

 ## References
1. GRASS GIS r.sun. Available online: https://grass.osgeo.org/grass78/manuals/r.sun.html
2. OpenSceneGraph. Available online: http://www.openscenegraph.org
3. osgEarth. Available online: http://osgearth.org
4. vcpkg. Available online: https://github.com/microsoft/vcpkg
5. OpenSceneGraph Plugins. http://www.openscenegraph.org/index.php/documentation/user-guides/61-osgplugins

## Contact:
Jianming Liang
jian9695@gmail.com


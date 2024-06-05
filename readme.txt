Eclipse
Eclipse's built in C++ indexing capabilities have gotten quite good in recent versions.

1. Installing Eclipse
   To use this tutorial, users should not "sudo apt-get install eclipse". Instead:

	- Go to eclipse web site
	- Click on "download now" from the top-right corner
	- Download eclipse for C/C++ developers
	- Extract eclipse into a folder of your choice
	- To make a nice launcher for Ubuntu, google for instructions or check e.g. 				http://www.blogs.digitalworlds.net/softwarenotes/?p=54


2. Creating the Eclipse project files
2.1 For a rosbuild package

	CMake can produce Eclipse project files automatically. These project files then set up all include paths correctly, so that auto completion and code browsing will work out of the box.

	However, currently, a small hack is required for generating these project files in the right folder. The problem is that ROS creates the Makefiles and all other build artifacts in the build/ folder, but Eclipse expects the source files to be located within (or below) the folder where its project files are stored.

	Fortunately, there is now a make target using a small script that circumvents this problem, by forcing CMake to create the project files in the directory of your package (and not the build/ folder). Proceed as follows:

	Open a terminal, roscd into the folder of your package, and execute:

		[ make eclipse-project ]

	You will now find two Eclipse files in your package. It is not recommended to add them to the repository, as they contain absolute links and will be different for different users checking out your software.

	Note that if you change anything to your manifest.xml, you will have to run this script again, which will overwrite your Eclipse project file and thereby reverting all manual changes to the project settings.

	Note: Simply running the cmake Eclipse generator like

		[ cmake -G"Eclipse CDT4 - Unix Makefiles" ]

	will overwrite the Makefile. This can also happen if make eclipse-project does not complete successfully. 
	If you already did this, you will want to restore the original Makefile, which should contain the following line:


		[ include $(shell rospack find mk)/cmake.mk ]

2.1.1 Creating eclipse files for multiple packages/stacks
	Go to the directory where your packages reside (which may be a stack-folder or just a simple folder) and run:

		[ rosmake --target=eclipse-project --specified-only * ]
	If you need to convert deeper nested packages or multiple stacks at once be encouraged to use this eclipse projects bash script for subdirectories.


2.2 Catkin-y approach
	If you are using catkin, you do not have the possibility to use make eclipse-project. You need to execute:

		[ catkin_make --force-cmake -G"Eclipse CDT4 - Unix Makefiles" ]
	
	to generate the .project file and then run:

		[ awk -f $(rospack find mk)/eclipse.awk build/.project > build/.project_with_env && mv build/.project_with_env build/.project ]

	to pass the current shell environment into the make process in Eclipse.

	After executing this command you will find the project files in the build/ folder. Now you can import your project as existing project into workspace.

	Maybe you will need to execute the following if you would like to debug your program. To execute this command cd to the build/ folder. You should do so if you e.g. get an error like "No source available for main()".

		[ cmake ../src -DCMAKE_BUILD_TYPE=Debug ]

	For information on the proper approach using catkin, start here

2.2.1 Catkin and Python
	For me, the above procedure didn't generate a .pydevproject file, like make eclipse-project ever did. Clicking Set as PyDev Project would create a config but without any Paths, so coding would be a hassle.

	Workaround: From within the package you want to program run:

		[ python $(rospack find mk)/make_pydev_project.py ]

	Now copy the created file .pydevproject (which has all dependency package paths included) to <catkin_ws>/build and import your catkin-project into eclipse or Set it as PyDev Project if already imported.


2.3 catkin tools
	With the new catkin_tools, there are few changed from the Catkin-y method described above. To generate eclipse-project you need to execute:

		[ catkin build  --force-cmake -G"Eclipse CDT4 - Unix Makefiles" ]

	to generate the .project files for each package and then run: the following script

		[
			ROOT=$PWD
			cd build
			for PROJECT in `find $PWD -name .project`; do
			    DIR=`dirname $PROJECT`
			    echo $DIR
			    cd $DIR
			    awk -f $(rospack find mk)/eclipse.awk .project > .project_with_env && mv .project_with_env .project
			done
			cd $ROOT
		]

	To debug use the following command and you can mention the name of the package to configure that specific project for debug instead of the entire workspace. Remember to run the script to modify .project to pass the current shell environment into the make process in Eclipse.

		[ catkin build  --force-cmake -G"Eclipse CDT4 - Unix Makefiles" -DCMAKE_BUILD_TYPE=Debug ]


3. Importing the project into Eclipse
	Now start Eclipse, select File --> Import --> Existing projects into workspace, hit next, then browse for your package's directory (select root directory). Do NOT select Copy projects into workspace. Then finish.

	You should now be able to browse through the code (hold down CTRL while clicking on a function/class name), get auto completion (automatically appears, or press CTRL-SPACE) et cetera.

3.1 Fixing unresolved includes
	There are many possible reasons why indexing cannot resolve all includes, functions, methods, variables, etc. Often, fixing the resolving of includes solves these errors. If you have any problems, these might be fixed by:

	- If the dependencies of your project have changed since first adding them to Eclipse, regenerate the project files and reimport the project into your workspace.
	- Making sure to load your .bashrc environment with Eclipse, by launching it using bash -i -c "eclipse" (see Reusing your shell's environment).

	- In Eclipse, right click the project, click properties -> C/C++ general -> Preprocessor Include Paths, Macros etc. Click the tab "Providers" and check the box next to "CDT GCC Built-in Compiler Settings [ Shared ]".

	- Afterwards, right click the project, select Index -> Rebuild. This will rebuild your index. Once this is done, usually all includes will resolve.

	- As a last resort, you can also manually add folders that will be searched for headers, using right click project -> properties -> C/C++ Include Paths and Symbols. This is usually not necessary though.


4. Building the project inside Eclipse
	The eclipse-project make target automatically tries to set the environment variables correctly such that building within Eclipse should work out-of-the-box. Especially if you follow Reusing your shell's environment from above.

	If not, this is where you need to start looking: Right click on the project, select Properties --> C/C++ Make Project --> Environment, and check whether the following environment variables are assigned to the correct values:

	- ROS_ROOT
	- ROS_PACKAGE_PATH
	- PYTHONPATH
	- PATH

	The easiest way to obtain the correct values for your installation is to open a terminal, and run

		[
			echo $ROS_ROOT
			echo $ROS_PACKAGE_PATH
			echo $PYTHONPATH
			echo $PATH
		]

	You should now be able to compile your package properly, for example by hitting CTRL-B (or selecting Project --> Build project in the menu).

	Note: When working with multiple projects, Eclipse won't be able to determine the correct build order or when dependent projects have to be rebuilt. You have to set the project interdependencies manually for each project in your workspace (see http://help.eclipse.org/helios/index.jsp?topic=/org.eclipse.cdt.doc.user/reference/cdt_u_prop_general_pns_ref.htm).


5. Running and debugging your executables within Eclipse
	As for building within Eclipse, the crucial step here is to set the required environment variables correctly in the launch configuration. As the same for building, this should work out-of-the-box, especially if you follow Reusing your shell's environment from above.

	Create a new launch configuration, right click on the project, select Run --> Run configurations... --> C/C++ Application (double click or click on New). Select the correct binary on the main tab (Search project should work when your binary was already built). Then in the environment tab, add (at least)

	- ROS_ROOT
	- ROS_MASTER_URI

	again with the values of your installation. If you are unsure about them, open a terminal and run

		[
			echo $ROS_ROOT
			echo $ROS_MASTER_URI
		]

	Finally, if you cannot save the configuration, remove the @ character in the name of the new run configuration.

	This should now allow you to run and debug your programs within Eclipse. The output directly goes into the Console of Eclipse. Note that the ROS_INFO macros use ANSI escape sequences, which are not parsed by Eclipse; therefore, the output might look similar to this one (from Writing a Publisher/Subscriber (C++)):


		[
			[0m[ INFO] [1276011369.925053237]: I published [Hello there! This is message [0]][0m
			[0m[ INFO] [1276011370.125082573]: I published [Hello there! This is message [1]][0m
			[0m[ INFO] [1276011370.325025148]: I published [Hello there! This is message [2]][0m
			[0m[ INFO] [1276011370.525034947]: I published [Hello there! This is message [3]][0m
		]

	You could use an ANSI console plugin (e.g. http://www.mihai-nita.net/eclipse/) to get rid of the "[0m" characters in the output.


6. More eclipse goodies
	- Setup a file template that pre-formats whenever a new source or header file is created. The template could for example contain the license header, author name, and include guards (in case of a header file). To setup the templates, choose in the Preferences C/C++->Code Style->Code Templates. 	- For example, to add the license header choose Files->C++ Source File->Default C++ source template and click on Edit.... Paste the license header and click OK. From now on, all source files while automatically contain the license header.

	- Enable Doxygen with the project properties clicking on C/C++ General, enabling project specific settings and selecting Doxygen as Documentation tool. This option automatically completes Doxygen style comments highlights them.

	- People that are used to the emacs key bindings can select emacs-style bindings in Windows->Preferences General->Keys and selecting the emacs Scheme. Also, other useful key bindings (e.g. make project) can easily be added.

	- To also index files that live outside the ROS tree (e.g. the boost include files) you can add them in project properties C/C++ General->Paths and Symbols.

	- The generated eclipse project also works great for Python code. Just install the PyDev plugin for syntax highlighting and code completion.


7. Auto Formatting
	Eclipse also has extensive formatting configuration capabilities. To add the ROS formatting profile to Eclipse, perform the following steps:

	- Download ROS_Format.xml to some location (versions: Indigo Kepler)
	- Start Eclipse
	- Select Window->Preferences->C/C++->Code Style
	- Click Import...
	- Select ROS_format.xml from the location used in step 1
	- Click OK

	As you edit a file, Eclipse should use this new profile to format your code following the ROS conventions. To reformat an entire file, select Edit->Format.
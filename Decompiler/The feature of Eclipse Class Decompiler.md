# The Feature of Eclipse Class Decompiler
**Eclipse Class Decompiler** is a plug-in for the Eclipse platform. It integrates JD, Jad, FernFlower, CFR, Procyon seamlessly with Eclipse. It displays all the Java sources during your debugging process, even if you do not have them all. And you can debug these class files directly without source code. It also integrates Javadoc and supports the syntax of JDK8 lambda expressions.
This project started in October 2012. When the Eclipse version was 3.x, I used the JadClipse plugin. But it didn’t support Eclipse 4.x, and the author maintained it no more. I decided to create a new decompiler plugin, and the initial version contained Jad and JD. The plugin support five decompilers now.
This plugin bases on the Eclipse JDT plgin. JDT provides a lot of features and has good scalability, so I can implement the decompiler plugin easily. The following graph is the decompiler plugin architecture diagram:

## Support Several Kinds of Decompiler
The first Java decompiler was Jad, the initial release was before 1999, 18 years ago. Jad was so old that it didn’t support Java generic type. Then JD appeared, it supported Java 7, much better than Jad. FernFlower, CFR, Procyon, they are modern decompilers, support Java 8. Eclipse Class Decompiler integrates all of them in one plugin. The recommended decompilers are FernFlower and JD. FernFlower which supports all Java versions and JD is faster. You can set the default decompiler for yourself at the Eclipse preference page.


## Support to Debug Code Without Source
If the class file contains debug attributes, you can debug it at the Eclipse Class Decompiler Viewer. There are two ways to debug class file. The first way is to set the decompiler preference,and to realign the line number. The secondary way is to check the debug mode at the decompiler menu bar. When your eclipse is at the debug perspective, the debug mode is default. The decompiler plugin will ignore your debug mode choice.


## Support JavaDoc and Java 8 Lambda Expression
The decompiler plugin implements the JavaDoc feature. If the jar binds javadoc at the Eclipse, the api document will display on the decompiler viewer.


Jad and JD don't support Java 8. If you choose them as the default decompiler, when the class compliance level is Java 8, the decompiler plugin will decompile the code by FernFlower automatically.


FernFlower, CFR and Procyon support Java 8 Lambda expression. But the decompiled source codes of these decompilers are not the same, you can choose the better one.


## Support to Export Decompiled Source Code
This is a utility feature. You can export the decompiled codes from one or more classes, even the whole jar.


## Support to DND and Decompile Class
The decompiler plugin can decompile the class file outside of Eclipse as well. Drag and drop the class file to the Eclipse editor area, the decompiled source code will display.


## Preference Settings
The decompiler preference settings page is below at **"Window > Preferences > Java > Decompiler"**. You can open the Eclipse perference dialog to set the preference for yourself. The most important setting is the "Default Class Decompiler", you may change it frequently.


Please also check the startup option and set the "Class Decompiler Viewer" as the default class viewer, otherwise the decompiler plugin is no longer effective.


## Install Eclipse Decompiler Plugin
* Click on "Help > Eclipse Marketplace...",
* Search "Eclipse Class Decompiler" in the Eclipse marketplace dialog,
* Find "Eclipse Class Decompiler" and click on button "install",
* Check "Eclipse Class Decompiler",
* Next, next, next... and restart.


## Status & Roadmap
The plugin is stable now, but it still has several issues to be fixed. Meanwhile, if the Eclipse upgrades or a new decompiler appears, I will also update this plugin.

Welcome to use the Eclipse Class Decompiler.

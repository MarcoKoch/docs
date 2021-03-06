.. _build_requires:


Build requirements
===================

There are some requirements that don't feel natural to add to a package recipe. For example, imagine that you had a ``cmake/3.4`` package in conan. Would you add it as a requirement to the ``ZLib`` package, so it will install cmake first in order to build ``Zlib``? In short:

- There are requirements that are only needed when you need to build a package from sources, but if the package binary already exists, you don't want to install or retrieve them.
- These could be dev tools, compilers, build systems, code analyzers, testing libraries, ..., etc.
- They can be very orthogonal to the creation of the package. It doesn't matter whether you build ZLib with cmake 3.4, 3.5 or 3.6. As long as the CMakeLists.txt is compatible, it will produce the same final package. 
- You don't want to add a lot of different versions (like those of cmake) to be able to use them to build the package. You want to easily change the requirements, without needing to edit the ZLib package recipe.
- Some of them might not be even be taken into account when a package like ZLib is created, such as cross-compiling it to Android (in which the Android toolchain would be a build requirement).

To address these needs, conan implements ``build_requires``

Declaring build requirements
------------------------------

Build requirements can be declared in profiles, like:

.. code-block:: text

  [build_requires]
  Tool1/0.1@user/channel
  Tool2/0.1@user/channel, Tool3/0.1@user/channel
  *: Tool4/0.1@user/channel
  MyPkg*: Tool5/0.1@user/channel
  &: Tool6/0.1@user/channel
  &!: Tool7/0.1@user/channel

Build requirements are specified by a ``pattern:``. If such pattern is not specified, it will be assumed to be ``*``, i.e. to apply to all packages. Packages can be declared in different lines or by a comma separated list.
In this example, Tool1, Tool2, Tool3 and Tool4 will be used for all packages in the dependency graph (while running ``conan install`` or ``conan test_package``)

If a pattern like ``MyPkg*`` is specified, the declared build requirements will only be applied to packages matching that pattern. Tool5 will not be applied to Zlib, for example, but it will be applied to MyPkgZlib.

The special case of a **consumer** conanfile, which might have no name or version, and thus impossible to match with a pattern, is handled with the ``&`` character. ``&`` means apply these build requirements to the consumer conanfile, while ``&!`` means apply the build requirements to all packages except the consumer one. Remember that the consumer conanfile is the one inside the ``test_package`` folder or the one referenced in the ``conan install`` command.


Build requirements can be also specified in a package recipe, with the ``build_requires`` attribute and the ``build_requirements()`` method:

.. code-block:: python

    class MyPkg(ConanFile):
        build_requires = "ToolA/0.2@user/testing", "ToolB/0.2@user/testing"

        def build_requirements(self):
            # useful for example for conditional build_requires
            if self.settings.os == "Windows":
                self.build_requires("ToolWin/0.1@user/stable")

The above ``ToolA`` and ``ToolB`` will be always retrieved and used for building this recipe, while the ``ToolWin`` one will only be used in Windows.

If some ``build_require`` defined inside ``build_requirements()`` has the same package name than one defined in the ``build_requires`` attribute, the one inside the ``build_requirements()`` method will prevail.

Also, as a rule of thumb, downstream defined values always override upstream dependencies values. If some build requirement is defined in the profile, it will overwrite the build requirements defined in package recipes that have the same package name.

Properties of build requirements
---------------------------------

The behavior of ``build_requires`` is the same, irrespective if they are defined in the profile, or if defined in the package recipe:

- Build requirements will only be retrieved and installed if some package that has to be built from sources matches the declared pattern. Otherwise, they will not be even checked for existence.
- Options and environment variables declared in the ``profile`` as well as in the command line will affect the build requirements packages. In that way, you can define, for example for the ``cmake_installer/0.1`` package, which cmake version will be installed
- Build requirements will be activated for matching packages via the ``deps_cpp_info`` and ``deps_env_info`` members. So, include directories, library names, compile flags (CFLAGS, CXXFLAGS, LINKFLAGS), sysroot, etc. will be applied from the build requirement package ``self.cpp_info`` values. The same for ``self.env_info``: variables such as PATH, PYTHONPATH, and any other environment variables will be applied to the matching patterns and activated as environment variables.
- Build requirements can also be transitive. They can declare their own requirements, both normal requirements and their own build requirements. Normal logic for dependency graph resolution applies, such as conflict resolution and dependencies overriding.
- Each matching pattern will produce a different dependency graph of build requiremens. These graphs are cached so that they are only computed once. If a build requirement applies to different packages, with the same configuration, it will only be installed once (same behavior as normal dependencies. Once they are cached locally, there is no need to retrieve nor build them again).
- Build requirements do not affect the binary package ID. If using a different build requirement produces a different binary, you should considering adding an option or a setting to model that (if not already modeled)
- Build requires can also use version-ranges, like ``Tool/[>0.3]@user/channel``
- Build requirements are not listed in ``conan info`` nor are represented in the graph (with ``conan info --graph``)


Testing libraries
------------------

One example of build requirement could be a testing framework, which is implemented as a library. Let's call it ``mytest_framework``, an existing conan package.

Build requirements can be checked for existence (whether they've been applied) in the recipes, which can be useful for conditional logic on the recipes. In this example, we could have one recipe with the following ``build()`` method:

.. code-block:: python

    def build(self):
        cmake = CMake(self)
        enable_testing = "mytest_framework" in self.deps_cpp_info.deps
        cmake.configure(defs={"ENABLE_TESTING": enable_testing})
        cmake.build()
        if enable_testing:
            cmake.test()

And the package ``CMakeLists.txt``:

.. code-block:: cmake

    project(PackageTest CXX)
    cmake_minimum_required(VERSION 2.8.12)

    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup()
    if(ENABLE_TESTING)
        add_executable(example test.cpp)
        target_link_libraries(example ${CONAN_LIBS})

        enable_testing()
        add_test(NAME example
                  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/bin
                  COMMAND example)
    endif()


This package recipe will not retrieve the ``mytest_framework`` nor build the tests, for normal installation:

.. code-block:: bash

    $ conan install

But if the following profile—let's call it ``mytest_profile``—is defined:

.. code-block:: text

  [build_requires]
  mytest_framework/0.1@user/channel

Then, the following command will retrieve the ``mytest_framework`` and build and run the tests:

.. code-block:: bash

    $ conan install --profile=mytest_profile


Common python code
-------------------

The same technique can be even used to inject and reuse python code in the package recipes, without having to declare dependencies to such python packages.

If a conan package is defined to wrap and reuse the ``mypythontool.py`` file:

.. code-block:: python

    import os
    from conans import ConanFile

    class Tool(ConanFile):
        name = "PythonTool"
        version = "0.1"
        exports_sources = "mypythontool.py"

        def package(self):
            self.copy("mypythontool.py")

        def package_info(self):
            self.env_info.PYTHONPATH.append(self.package_folder)

Then, if a profile is defined:

.. code-block:: text

  [build_requires]
  PythonTool/0.1@user/channel
  
  
such package can be reused in other recipes, such as the following:

.. code-block:: python

    def build(self):
        self.run("mytool")
        with tools.pythonpath(self):
            import mypythontool
            self.output.info(mypythontool.hello_world())

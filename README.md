# MAME on CHERI

[MAME](www.mamedev.org) is a project to create software emulations of gaming hardware, such as
arcade gaming machines. These emulations allow the playing of games from the original ROM images
dumped from hardware. This document contains some notes on a partial port of MAME to run in
purecap mode on [CheriBSD](www.cheribsd.org) on an ARM Morello machine.

Please note that MAME is a registered trademark of Gregory Ember. See
[here](https://www.mamedev.org/legal.html) for details about the ownership and licensing of MAME.

## Versions used

We used CheriBSD version 24.05, and MAME version 0.257 from the
[CheriBSD ports tree](https://github.com/CTSRD-CHERI/cheribsd-ports).

The work described below was carried out in October and November 2024.

## Background

CheriBSD maintains two separate package management systems, one for purecap and one for hybrid mode, as explained
[here](ctsrd-cheri.github.io/cheribsd-getting-started/packages/index.html). For an explanation of
purecap vs hybrid mode on CheriBSD, see [here](ctsrd-cheri.github.io/cheribsd-getting-started/features/processes.html).
In order to build MAME from source as a purecap application, we found that it was necessary to install a number of
development tools. We installed all of these tools using the hybrid package manager, `pkg64`. We installed one package,
`portconfig`, via the purecap package manager `pkg64c`.

In order to build MAME in purecap, it is also necessary to build a number of dependencies of MAME in purecap. The
ports build system deals with the problem of resolving dependencies, building them, and installing them, but some
of the dependencies need manual changes to build for purecap.

The build process follows the standard pattern of building software for FreeBSD using the ports system, but with a
number of alterations to make the build work on CheriBSD purecap. These alterations include both changes to the
makefiles used to control the build, as well as some changes to the source code.

MAME as a whole is a very large piece of software, providing emulation of a vast array of different hardware. A full
build of all components would (we assume) take a very long time. However, we only wished to demonstrate the possibility
of running ROMs via MAME in purecap, and so we only compiled support for a small number of MAME's emulations. This meant
that we only needed to port a small portion of the MAME code to purecap, making the task feasible.

## Outcome

We were able to compile a purecap version of MAME with support for running a small collection of ROM images by making
only a few small changes to the source code of MAME and its dependencies. Some of the ROMs ran without any problems on
the purecap emulators, while others were less reliable and did often crash. We believe that, with further debugging and
resulting small changes to source code, we could make all the ROMs run reliably. However, our purpose here is simply to
provide proof of concept that compiling MAME components for purecap is possible, and so we have not taken this any further.


## Method

We now lay out in detail the steps to reproduce our work. Except where otherwise noted, all commands were run as root.

The work described here was carried out in October and November 2024. It may be necessary to adapt the
method given below to take account of subsequent changes to the code in the ports systems, or other changes.

#### Installation of required packages
```
pkg64 install git llvm-base cmake ninja nano textproc/py-sphinxcontrib-svg2pdfconverter
pkg64c install portconfig
```

### Installation of ports tree
```
mkdir /usr/ports
cd /usr/ports/
git clone --depth 1 https://github.com/CTSRD-CHERI/cheribsd-ports.git .
```

### Change tools used by ports building
In the file `/usr/ports/Mk/bsd.commands.mk`, it is necessary to change two lines
to force the ports build system to use the tool `bsddialog` instead of `dialog`, and
`portconfig` instead of `dialog4ports`. The replaced tools are not compatible with a
purecap build.

Edit the file to change the lines
```
DIALOG?=		/usr/bin/dialog
DIALOG4PORTS?=		${LOCALBASE}/bin/dialog4ports
```
to
```
|DIALOG?=                /usr/bin/bsddialog
|DIALOG4PORTS?=          ${LOCALBASE}/bin/portconfig
```

### Edit to code of `pugixml`
The only dependency of MAME which requires any alteration is `pugixml`, and this change
is limited to a single line.

First, fetch the source code
```
cd /usr/ports/textproc/pugixml
make fetch extract
```
then edit the file `/usr/ports/textproc/pugixml/work/pugixml-1.13/src/pugixml.hpp`
by replacing the size of `_memory` on line 1033 to `300`

### Build MAME dependencies
Now go to `/usr/ports/emulators/mame/`, and edit the Makefile as follows.
1. Add the string `aarch64c` to the list of values on the line which begins `ONLY_FOR_ARCHS= ` (line 20),
   so that it reads `ONLY_FOR_ARCHS=  aarch64c aarch64 amd64 armv7 i386 powerpc powerpc64 powerpc64le`
2. On the line which begins `BUILD_DEPENDS= ` (line 23), remove the rest of this line after `BUILD_DEPENDS= `
   including the newline,
   so that the line now reads `BUILD_DEPENDS= glm>0:math/glm \`. The removed text is
   `${PYTHON_PKGNAMEPREFIX}sphinxcontrib-svg2pdfconverter>0:textproc/py-sphinxcontrib-svg2pdfconverter@${PY_FLAVOR} \`
3. Modify the code block on lines 110 to 125 (inclusive) to read
```
	.if ${OPSYS} == FreeBSD && ${OSVERSION} > 1400000
	MAKE_ENV+=      OVERRIDE_AR="llvm-ar-morello" \
	                OVERRIDE_CC="clang-morello" \
	                OVERRIDE_CXX="clang++-morello" \
	                OVERRIDE_LD="lld-morello"
	.else
	MAKE_ENV+=      OVERRIDE_AR="${AR}" \
	                OVERRIDE_CC="${CC}" \
	                OVERRIDE_CXX="${CXX}" \
	                OVERRIDE_LD="${LD}"
	.endif
```
4. Add a new line 71 which reads `CFLAGS_aarch64c= -march=morello+crc+crypto` (the previous line, line 70,
   starts `CFLAGS_aarch64=`)

Now set up the build environment by running
```
export USE_PACKAGE_DEPENDS=yes USE_PACKAGE_DEPENDS_REMOTE=yes USE_PACKAGE_64_DEPENDS_ONLY=yes
```
For an explanation of these environment variables, see [this document](https://freebsdfoundation.org/wp-content/uploads/2023/05/CheriBSD_ports.pdf).

Finally, build the dependencies by running `make depends` (still in the directory `/usr/ports/emulators/mame/`). During the build,
you will be asked to select from some lists of options. It is necessary to make some changes to the selections to avoid building
some documentation which has dependencies that we did not port, and which make the build fail.
The necessary changes are shown in the following table. Any other options should be left unchanged.

|Package name| Options to uncheck if checked|
|-----------|-----------|
|`utf8proc` | `docs` |
|`sdl2_ttf` | `docs` and `harfbuzz` |
|`glm`| `docs` |
|`rapidjson`| `doxygen`|

### Building MAME

Before building MAME, it is necessary to make 3 small modifications to the source code. Note that these changes were simply aimed at getting the
code to compile, and to run at least one ROM without crashing. Thus these changes are very crude, and do not make any effort to fix the underlying
logic for use in a CHERI environment, but rather just disable things which cause problems with compilation. This was sufficient for our needs in
this project.

First, `cd` to `/usr/ports/emulators/mame` and fetch the source code with `make fetch extract`. Running this command will eventually cause a menu
of options to appear. Uncheck the `DOCS` and `NLS` options, so that no options remain checked on the menu.

There are three source files which need to be edited. This can either be done directly before running the build, or else the changes can be made and
then saved as patches using the `makepatch` functionality of the ports system, which allows these changes to be automatically re-applied when rebuilding
after a `make clean` command. This section describes making the changes directly before building, see the next section for the use of `makepatch`.

Make the following edits.
1. In the file `work/mame-mame0257/3rdparty/bx/include/bx/file.h`, find the two instances of the string `m_internal[64]` and change them to
   `m_internal[128]`
2. In the file `work/mame-mame0257/src/emu/validity.cpp`, find the block
```
        // check pointer size
#ifdef PTR64
	static_assert(sizeof(void *) == 8, "PTR64 flag enabled, but was compiled for 32-bit target\n");
#else
	static_assert(sizeof(void *) == 4, "PTR64 flag not enabled, but was compiled for 64-bit target\n");
#endif
```
and either remove it or comment it out.
3. In the file `  work/mame-mame0257/src/lib/util/dynamicclass.h`, find the `static_assert` with the message
   `Pointer and pointer difference must be the same size` and remove or comment it out (i.e. remove or comment
   out the whole line).

Now set up the build environment by running
```
export USE_PACKAGE_DEPENDS=yes USE_PACKAGE_DEPENDS_REMOTE=yes USE_PACKAGE_64_DEPENDS_ONLY=yes
```
and then use `make` to run the build. We used the following `make` command
```
make -j4 SOURCES=capcom/1943.cpp,nintendo/dkong.cpp,namco/galaga.cpp,pacman/jrpacman.cpp,pacman/pacman.cpp,capcom/cps1.cpp,konami/simpsons.cpp,namco/namcos12.cpp,taito/taito_h.cpp
```
where the `-j4` tells `make` to use four jobs (this makes the build run faster, the exact value 4 here was chosen fairly arbitrarily) and the `SOURCES=` argument tells the MAME
build system to build only what is necessary to support the emulators provided by those source files. We chose which source files to include based on the ROM images available to
us. Restricting what is included allows for a faster build, and avoids the need to make changes to the whole body of MAME source code to get it to compile in purecap.

### Using `makepatch` for reproducible modifications to source code (optional)

The ports build system allows changes made to source code to be saved as reusable patches which can be automatically re-applied when rebuilding after a `make clean` (which will
undo any changes made to the source code). We have found this to be a convenient means of avoiding making the same edits repeatedly. We describe here how to use `makepatch` in
this way. Note that the steps in this section are not necessary to build MAME for purecap, but may be convenient if you are doing your own work on the source code in addition
to the changes described above.

Before making any changes to the MAME source code (e.g. immediately after running `make fetch extract` in `/usr/ports/emulators/mame`), it is necessary to save a copy of each
file which will be modified, to serve as a basis for the patch which will be created later. This is done by making a copy of the file in the same directory, with `.orig` suffixed
to the file name. For example, when making the changes from the last section, do the following.
```
cp work/mame-mame0257/3rdparty/bx/include/bx/file.h work/mame-mame0257/3rdparty/bx/include/bx/file.h.orig
cp work/mame-mame0257/src/emu/validity.cpp work/mame-mame0257/src/emu/validity.cpp.orig
cp work/mame-mame0257/src/lib/util/dynamicclass.h work/mame-mame0257/src/lib/util/dynamicclass.h.orig
```
Then make modification to the files as described above. Then run `make makepatch` from `/usr/ports/emulators/mame` to create the patches.

Now, before running the `make` command to build MAME after modifying the source code (but after running the `export` command as above), run `make clean`
(this is necessary to undo the changes made to the working copies of the source files, so that the patches generated above will apply cleanly).

After doing the above steps, the source code modifications will be automatically re-applied when running a build via `make` after invoking `make clean`.

### Running purecap MAME

To run the purecap MAME, do the following. It is assumed below that there is an ordinary user on the CheriBSD system called `user`, and that you will
be running MAME from a graphical login as this user.

1. Copy the necessary ROM files to `/home/user/roms`
2. As `user`, `cd` in a shell to `/usr/ports/emulators/mame/work/stage/usr/local/bin`
3. Run `./mame -rompath /home/user/roms -uifontprovider none` . The `-uifontprovider none` is needed to make the purecap MAME run without immediately
   crashing with a CHERI capability error.

MAME can be exited nicely by sending it the `HUP` signal, `kill -s HUP <procnum>`







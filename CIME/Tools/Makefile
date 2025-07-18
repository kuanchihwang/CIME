#===============================================================================
#
# Common Makefile: a framework for building all CIME components and more
#
#===============================================================================
ifdef MODEL
  ifndef COMP_NAME
    $(warning "Variable MODEL is deprecated, please use COMP_NAME instead")
    COMP_NAME:=$(MODEL)
  else
    ifneq ($(MODEL), $(COMP_NAME))
      $(warning "MODEL is inconsistent with COMP_NAME $(MODEL) $(COMP_NAME)")
      COMP_NAME:=$(MODEL)
    endif
  endif
endif

# Set up special characters
null  :=
comma := ,

# Load dependency search path.
dirs := .
dirs += $(shell cat Filepath)

cpp_dirs := $(dirs)
# Add INCROOT to path for Depends and Include
MINCROOT :=
ifdef INCROOT
  cpp_dirs += $(INCROOT)
  MINCROOT := $(INCROOT)
endif

# Expand any tildes in directory names. Change spaces to colons.
VPATH := $(foreach dir,$(cpp_dirs),$(wildcard $(dir)))
VPATH := $(subst $(space),:,$(VPATH))

RM    := rm
CP    := cp

exec_se: $(EXEC_SE)  Depends
complib: $(COMPLIB)  Depends

# Determine whether to compile threaded or not
# Set the THREADDIR for the shared build
# based on the threaded build status
ifeq ($(strip $(SMP)),TRUE)
   THREADDIR = threads
   compile_threaded = TRUE
else
   ifeq ($(strip $(BUILD_THREADED)),TRUE)
      THREADDIR = threads
      compile_threaded = TRUE
   else
      THREADDIR = nothreads
      compile_threaded = FALSE
   endif
endif

# set the debug directory based on the debug status
ifeq ($(strip $(DEBUG)),TRUE)
   DEBUGDIR = debug
else
   DEBUGDIR = nodebug
endif

SLIBS ?= $(USER_SLIBS)

ifndef MOD_SUFFIX
   MOD_SUFFIX := mod
endif

#===============================================================================
# set CPP options (must use this before any flags or cflags settings)
#===============================================================================

CPPDEFS := $(USER_CPPDEFS) -D$(OS)

-include $(CASEROOT)/Macros.make

# Unless DEBUG mode is enabled, use NDEBUG to turn off assert statements.
ifeq ($(strip $(DEBUG)),TRUE)
  # e3sm still has components that cannot build with -DDEBUG
  ifneq ($(CIME_MODEL),e3sm)
    CPPDEFS += -DDEBUG
  endif
else
   CPPDEFS += -DNDEBUG
endif

# USE_ESMF_LIB is currently only defined in env_build.xml
ifeq ($(USE_ESMF_LIB), TRUE)
   CPPDEFS += -DUSE_ESMF_LIB
endif

ifeq ($(COMPARE_TO_NUOPC), TRUE)
   CPPDEFS += -DCOMPARE_TO_NUOPC
endif

ifeq ($(strip $(MPILIB)),mpi-serial)
  CPPDEFS += -DNO_MPI2
else
  CPPDEFS += -DHAVE_MPI
endif

ifeq (,$(SHAREDPATH))
  SHAREDPATH = $(COMPILER)/$(MPILIB)/$(DEBUGDIR)/$(THREADDIR)
  INSTALL_SHAREDPATH = $(EXEROOT)/$(SHAREDPATH)
endif

#===============================================================================
# User-specified INCLDIR
#===============================================================================
INCLDIR := -I.
ifdef USER_INCLDIR
  INCLDIR += $(USER_INCLDIR)
endif

FTORCH_DIR = $(SHAREDLIBROOT)/$(SHAREDPATH)/FTorch/
ifneq "$(wildcard $(FTORCH_DIR) )" ""
  INCLDIR  += -I$(FTORCH_DIR)/modules -I$(FTORCH_DIR)
  SLIBS  +=  -Wl,-rpath,$(FTORCH_DIR) -L$(FTORCH_DIR) -lftorch_wrapper
endif
ifeq ($(strip $(USE_FTORCH)), TRUE)
  CPPDEFS  += -DUSE_FTORCH
  SLIBS  +=  -Wl,-rpath,$(TORCH_DIR)/lib -L$(TORCH_DIR)/lib -ltorch_cpu -lc10 -lstdc++ -Wl,-rpath,$(FTORCH_DIR)/src -L$(FTORCH_DIR)/src -lftorch
endif

ifdef FSTDLIB_PKGCONFIG
  STDLIB_INC  := `pkg-config --cflags fortran_stdlib`
  STDLIB_LIBS := `pkg-config --libs fortran_stdlib`
  INCLDIR += $(STDLIB_INC)
  SLIBS += $(STDLIB_LIBS)
endif

ifeq ($(strip $(USE_FMS)), TRUE)
  SLIBS += -lfms
  INCLDIR += -I$(SHAREDLIBROOT)/$(SHAREDPATH)/FMS/
endif


CPPDEFS += -DNUOPC_INTERFACE
INCLDIR += -I$(SHAREDLIBROOT)/$(SHAREDPATH)/CDEPS/fox/include -I$(SHAREDLIBROOT)/$(SHAREDPATH)/CDEPS/dshr
# FoX libraries are provided and built by CDEPS
FoX_LIBS := -L$(SHAREDLIBROOT)/$(SHAREDPATH)/CDEPS/fox/lib -lFoX_dom -lFoX_sax -lFoX_utils -lFoX_fsys -lFoX_wxml -lFoX_common -lFoX_fsys
ULIBS += -L$(SHAREDLIBROOT)/$(SHAREDPATH)/CDEPS/dshr -ldshr -L$(SHAREDLIBROOT)/$(SHAREDPATH)/CDEPS/streams -lstreams
SLIBS += $(FoX_LIBS)

CPPDEFS += -DPIO2

# Not clear how to escape commas for libraries with their own configure
# script, and they don't need this defined anyway, so leave this out of
# FPPDEFS.
ifeq ($(HAS_F2008_CONTIGUOUS),TRUE)
  CONTIGUOUS_FLAG := -DUSE_CONTIGUOUS=contiguous,
else
  CONTIGUOUS_FLAG := -DUSE_CONTIGUOUS=
endif

ifdef CPRE
  CONTIGUOUS_FLAG := $(subst $(comma),\\$(comma),$(CONTIGUOUS_FLAG))
  CONTIGUOUS_FLAG := $(patsubst -D%,$(CPRE)%,$(CONTIGUOUS_FLAG))
endif

AR ?= ar
ARFLAGS ?= -r

ifdef NETCDF_C_PATH
  ifndef NETCDF_FORTRAN_PATH
    $(error "NETCDF_C_PATH specified without NETCDF_FORTRAN_PATH")
  endif
  NETCDF_SEPARATE:=TRUE
  ifndef INC_NETCDF_C
    INC_NETCDF_C:=$(NETCDF_C_PATH)/include
  endif
  ifndef INC_NETCDF_FORTRAN
    INC_NETCDF_FORTRAN:=$(NETCDF_FORTRAN_PATH)/include
  endif
  ifndef LIB_NETCDF_C
    LIB_NETCDF_C:=$(NETCDF_C_PATH)/lib
  endif
  ifndef LIB_NETCDF_FORTRAN
    LIB_NETCDF_FORTRAN:=$(NETCDF_FORTRAN_PATH)/lib
  endif
else ifdef NETCDF_FORTRAN_PATH
  $(error "NETCDF_FORTRAN_PATH specified without NETCDF_C_PATH")
else ifdef NETCDF_PATH
  NETCDF_SEPARATE:=FALSE
  ifndef INC_NETCDF
    INC_NETCDF:=$(NETCDF_PATH)/include
  endif
  ifndef LIB_NETCDF
    LIB_NETCDF:=$(NETCDF_PATH)/lib
  endif
else
  # No Netcdf is an error unless target is clean or DEP
  ifneq ($(MAKECMDGOALS), db_files)
  ifneq ($(MAKECMDGOALS), db_flags)
  ifeq (,$(findstring clean,$(MAKECMDGOALS)))
      $(error NETCDF not found: Define NETCDF_PATH or NETCDF_C_PATH and NETCDF_FORTRAN_PATH in config_machines.xml or cmake_macros)
  endif
  endif
  endif
endif

ifeq ($(MPILIB),mpi-serial)
  ifdef PNETCDF_PATH
    undefine PNETCDF_PATH
  endif
else
  ifdef PNETCDF_PATH
    ifndef $(INC_PNETCDF)
      INC_PNETCDF:=$(PNETCDF_PATH)/include
    endif
    ifndef LIB_PNETCDF
      LIB_PNETCDF:=$(PNETCDF_PATH)/lib
    endif
  endif
endif

# Set PETSc info if it is being used
ifeq ($(strip $(USE_PETSC)), TRUE)
  ifdef PETSC_PATH
    ifndef INC_PETSC
      INC_PETSC:=$(PETSC_PATH)/include
    endif
    ifndef LIB_PETSC
      LIB_PETSC:=$(PETSC_PATH)/lib
    endif
  else
    $(error PETSC_PATH must be defined when USE_PETSC is TRUE)
  endif

  # Get the "PETSC_LIB" list an env var
  include $(PETSC_PATH)/lib/petsc/conf/variables
endif

# Set MOAB info if it is being used
ifeq ($(COMP_INTERFACE), moab)
  ifdef MOAB_PATH
    CPPDEFS += -DHAVE_MOAB
    ifndef INC_MOAB
      INC_MOAB:=$(MOAB_PATH)/include
    endif
    ifndef LIB_MOAB
      LIB_MOAB:=$(MOAB_PATH)/lib
    endif
  else
    $(error MOAB_PATH must be defined when using moab driver)
  endif
endif

# Set HAVE_SLASHPROC on LINUX systems which are not bluegene or Darwin (OSx)

ifeq ($(findstring -DLINUX,$(CPPDEFS)),-DLINUX)
  ifneq ($(findstring DBG,$(CPPDEFS)),DBG)
    ifneq ($(findstring Darwin,$(CPPDEFS)),Darwin)
      CPPDEFS += -DHAVE_SLASHPROC
    endif
  endif
endif

ifdef LIB_PNETCDF
   CPPDEFS += -D_PNETCDF
   SLIBS += -L$(LIB_PNETCDF) -lpnetcdf
endif

# Set esmf.mk location with ESMF_LIBDIR having precedence over ESMFMKFILE
CIME_ESMFMKFILE := undefined_ESMFMKFILE
ifdef ESMFMKFILE
   CIME_ESMFMKFILE := $(ESMFMKFILE)
endif
ifdef ESMF_LIBDIR
   CIME_ESMFMKFILE := $(ESMF_LIBDIR)/esmf.mk
endif
# For compiling and linking with external ESMF.
# If linking to external ESMF library then include esmf.mk
# ESMF_F90COMPILEPATHS
# ESMF_F90ESMFLINKPATHS
# ESMF_F90LINKRPATHS
# ESMF_F90ESMFLINKLIBS
# ESMF_F90LINKPATHS
ifeq ($(USE_ESMF_LIB), TRUE)
  -include $(CIME_ESMFMKFILE)
  CPPDEFS += -DESMF_VERSION_MAJOR=$(ESMF_VERSION_MAJOR) -DESMF_VERSION_MINOR=$(ESMF_VERSION_MINOR)
  FFLAGS += $(ESMF_F90COMPILEPATHS)
  SLIBS += $(ESMF_F90ESMFLINKPATHS) $(ESMF_F90ESMFLINKRPATHS) $(ESMF_F90ESMFLINKLIBS) $(ESMF_F90LINKPATHS)
endif

# Stub libraries do not need to be built for nuopc driver
# so it will override these settings on the command line
ATM_PRESENT ?= TRUE
ICE_PRESENT ?= TRUE
LND_PRESENT ?= TRUE
OCN_PRESENT ?= TRUE
ROF_PRESENT ?= TRUE
GLC_PRESENT ?= TRUE
WAV_PRESENT ?= TRUE
ESP_PRESENT ?= TRUE
IAC_PRESENT ?= TRUE
MED_PRESENT ?= TRUE
ifeq ($(ULIBDEP),$(null))
  ifneq ($(LIBROOT),$(null))
    ifeq ($(ATM_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libatm.a
      CPPDEFS += -DATM_PRESENT
    endif
    ifeq ($(ICE_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libice.a
      CPPDEFS += -DICE_PRESENT
    endif
    ifeq ($(LND_PRESENT),TRUE)
      ULIBDEP += $(LNDLIBDIR)/$(LNDLIB)
      CPPDEFS += -DLND_PRESENT
    endif
    ifeq ($(OCN_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libocn.a
      CPPDEFS += -DOCN_PRESENT
    endif
    ifeq ($(ROF_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/librof.a
      CPPDEFS += -DROF_PRESENT
    endif
    ifeq ($(GLC_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libglc.a
      CPPDEFS += -DGLC_PRESENT
    endif
    ifeq ($(WAV_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libwav.a
      CPPDEFS += -DWAV_PRESENT
    endif
    ifeq ($(ESP_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libesp.a
      CPPDEFS += -DESP_PRESENT
    endif
    ifeq ($(IAC_PRESENT),TRUE)
      ULIBDEP += $(LIBROOT)/libiac.a
    endif
    ifeq ($(MED_PRESENT),TRUE)
      CPPDEFS += -DMED_PRESENT
    endif
  endif
endif

CPPDEFS += -DPIO$(PIO_VERSION)

ifdef CPRE
  FPPDEFS := $(subst $(comma),\\$(comma),$(CPPDEFS))
  FPPDEFS := $(patsubst -D%,$(CPRE)%,$(FPPDEFS))
  EXTRA_PIO_FPPDEFS := $(subst $(comma),\\$(comma),$(EXTRA_PIO_CPPDEFS))
  EXTRA_PIO_FPPDEFS := $(patsubst -D%,$(CPRE)%,$(EXTRA_PIO_FPPDEFS))
else
  FPPDEFS := $(CPPDEFS)
  EXTRA_PIO_FPPDEFS := $(EXTRA_PIO_CPPDEFS)
endif

#===============================================================================
# Set config args for pio to blank and then enable serial
#===============================================================================
ifndef CONFIG_ARGS
  CONFIG_ARGS :=
endif
ifeq  ($(findstring pio,$(COMP_NAME)),pio)
  CONFIG_ARGS+= --enable-timing
  ifeq ($DEBUG,TRUE)
     CONFIG_ARGS+= --enable-debug
  endif
endif

#===============================================================================
# MPI-serial library
#===============================================================================

ifeq ($(strip $(MPILIB)), mpi-serial)
  CC	  := $(SCC)
  FC	  := $(SFC)
  CXX	  := $(SCXX)
  MPIFC	  := $(SFC)
  MPICC	  := $(SCC)
  MPICXX  := $(SCXX)
  ifndef MPI_SERIAL_PATH
    CONFIG_ARGS += MCT_PATH=$(SHAREDLIBROOT)/$(SHAREDPATH)/mpi-serial
  else
    CONFIG_ARGS += MCT_PATH=$(MPI_SERIAL_PATH)
    INC_MPI := $(MPI_SERIAL_PATH)/include
    LIB_MPI := $(MPI_SERIAL_PATH)/lib
  endif
else
  CC  := $(MPICC)
  FC  := $(MPIFC)
  CXX := $(MPICXX)
  ifdef MPI_PATH
    INC_MPI := $(MPI_PATH)/include
    LIB_MPI := $(MPI_PATH)/lib
  endif
endif
LD := $(MPIFC)

CSM_SHR_INCLUDE:=$(INSTALL_SHAREDPATH)/include
# This is needed so that dependancies are found
VPATH+=$(CSM_SHR_INCLUDE)

#===============================================================================
# Set include paths (needed after override for any model specific builds below)
#===============================================================================
INCLDIR += -I$(INSTALL_SHAREDPATH)/include -I$(INSTALL_SHAREDPATH)/include -I$(INSTALL_SHAREDPATH)/finclude

ifeq ($(NETCDF_SEPARATE), FALSE)
  INCLDIR += -I$(INC_NETCDF)
else ifeq ($(NETCDF_SEPARATE), TRUE)
  INCLDIR += -I$(INC_NETCDF_C) -I$(INC_NETCDF_FORTRAN)
endif
ifdef MOD_NETCDF
  INCLDIR += -I$(MOD_NETCDF)
endif
ifdef INC_MPI
  INCLDIR += -I$(INC_MPI)
endif
ifdef INC_PNETCDF
  INCLDIR += -I$(INC_PNETCDF)
endif
ifdef INC_PETSC
  INCLDIR += -I$(INC_PETSC)
endif
ifdef INC_MOAB
  INCLDIR += -I$(INC_MOAB)
endif

ifeq ($(COMP_NAME),driver)
  INCLDIR += -I$(EXEROOT)/atm/obj -I$(EXEROOT)/ice/obj -I$(EXEROOT)/ocn/obj -I$(EXEROOT)/glc/obj -I$(EXEROOT)/rof/obj -I$(EXEROOT)/wav/obj -I$(EXEROOT)/esp/obj -I$(EXEROOT)/iac/obj
# nagfor and gcc have incompatible LDFLAGS.
# nagfor requires the weird "-Wl,-Wl,," syntax.
  ifeq ($(strip $(COMPILER)),nag)
     ifeq ($(NETCDF_SEPARATE), false)
       SLIBS += -Wl,-Wl,,-rpath=$(LIB_NETCDF)
     else ifeq ($(NETCDF_SEPARATE), true)
       SLIBS += -Wl,-Wl,,-rpath=$(LIB_NETCDF_C)
       SLIBS += -Wl,-Wl,,-rpath=$(LIB_NETCDF_FORTRAN)
     endif
  endif
else
  ifeq ($(strip $(COMPILER)),nag)
    ifeq ($(DEBUG), TRUE)
      ifneq (,$(filter $(strip $(MACH)),hobart izumi))
       # GCC needs to be able to link to
       # nagfor runtime to get autoconf
       # tests to work.
	 CFLAGS += -Wl,--as-needed,--allow-shlib-undefined
	 SLIBS += -L$(COMPILER_PATH)/lib/NAG_Fortran -lf62rts
       endif
    endif
  endif
endif

ifdef PIO_LIBDIR
  ifeq ($(PIO_VERSION),$(PIO_VERSION_MAJOR))
    INCLDIR += -I$(PIO_INCDIR)
    SLIBS += -L$(PIO_LIBDIR)
  else
    # If PIO_VERSION_MAJOR doesnt match, build from source
    unexport PIO_LIBDIR
  endif
endif
PIO_LIBDIR ?= $(INSTALL_SHAREDPATH)/lib

ifndef GPTL_LIBDIR
  GPTL_LIBDIR=$(INSTALL_SHAREDPATH)/lib
endif

ifndef GLC_DIR
  GLC_DIR=$(EXEROOT)/glc
endif
ifndef CISM_LIBDIR
  CISM_LIBDIR=$(GLC_DIR)/lib
endif
ifndef GLCROOT
  # Backwards compatibility
  GLCROOT=$(CIMEROOT)/../components/cism
endif

INCLDIR +=	-I$(INSTALL_SHAREDPATH)/include

CFLAGS+=$(CPPDEFS)
CXXFLAGS+=$(CPPDEFS)
CONFIG_ARGS +=	CC="$(CC)" FC="$(FC)" MPICC="$(MPICC)" \
		MPIFC="$(MPIFC)" FCFLAGS="$(FFLAGS) $(FREEFLAGS) $(INCLDIR)" \
		CPPDEFS="$(CPPDEFS)" CFLAGS="$(CFLAGS) -I.. $(INCLDIR)" \
		LDFLAGS="$(LDFLAGS)"

ifeq ($(NETCDF_SEPARATE), FALSE)
  CONFIG_ARGS += NETCDF_PATH=$(NETCDF_PATH)
else ifeq ($(NETCDF_SEPARATE), TRUE)
  CONFIG_ARGS += NETCDF_PATH=$(NETCDF_C_PATH)
endif

FFLAGS += $(FPPDEFS)
FFLAGS_NOOPT += $(FPPDEFS)

ifeq ($(findstring -cosp,$(CAM_CONFIG_OPTS)),-cosp)
  # The following is for the COSP simulator code:
  COSP_LIBDIR:=$(abspath $(EXEROOT)/atm/obj/cosp)
  ifeq ($(COMP_NAME),driver)
    INCLDIR+=-I$(COSP_LIBDIR)
  endif
endif

COMP_ATM ?= $(shell $(CASEROOT)/xmlquery --caseroot $(CASEROOT) COMP_ATM --value)
ifeq ($(strip $(COMP_ATM)),cam)
  CAM_DYCORE ?= $(shell $(CASEROOT)/xmlquery --caseroot $(CASEROOT) CAM_DYCORE --value)
  ifeq ($(CAM_DYCORE),fv3)
    FV3CORE_LIBDIR:=$(abspath $(EXEROOT)/atm/obj/atmos_cubed_sphere)
    INCLDIR+=-I$(FV3CORE_LIBDIR) -I$(FV3CORE_LIBDIR)/../ -I../$(INSTALL_SHAREDPATH)/include -I../$(CSM_SHR_INCLUDE) -I$(abspath $(EXEROOT)/FMS) -I$(CIMEROOT)/../libraries/FMS/src/include
  endif

  ifeq ($(CAM_DYCORE),mpas)
    # For building CAM with the MPAS dycore
    MPAS_LIBDIR:=$(abspath $(EXEROOT)/atm/obj/mpas)
    ifeq ($(COMP_NAME),driver)
      # This is needed by some compilers when building the driver.
      INCLDIR+=-I$(MPAS_LIBDIR)
    endif
  endif
endif

ifeq ($(COMP_NAME),cam)
  # These RRTMG files take an extraordinarily long time to compile with optimization.
  # Until mods are made to read the data from files, just remove optimization from
  # their compilation.
rrtmg_lw_k_g.o: rrtmg_lw_k_g.f90
	$(FC) -c $(FPPFLAGS) $(INCLDIR) $(INCS) $(FREEFLAGS) $(FFLAGS) $(FFLAGS_NOOPT) $<

rrtmg_sw_k_g.o: rrtmg_sw_k_g.f90
	$(FC) -c $(FPPFLAGS) $(INCLDIR) $(INCS) $(FREEFLAGS) $(FFLAGS) $(FFLAGS_NOOPT) $<

ifdef COSP_LIBDIR
INCLDIR+=-I$(COSP_LIBDIR) -I$(COSP_LIBDIR)/../ -I../$(INSTALL_SHAREDPATH)/include -I../$(CSM_SHR_INCLUDE)
$(COSP_LIBDIR)/libcosp.a: cam_abortutils.o
	$(MAKE) -C $(COSP_LIBDIR) F90='$(FC)' F90FLAGS='$(INCLDIR) $(INCS) $(FREEFLAGS) $(FFLAGS) $(FC_AUTO_R8)' \
	F90FLAGS_noauto='$(INCLDIR) $(INCS) $(FREEFLAGS) $(FFLAGS)' \
	F90FLAGS_fixed='$(INCLDIR) $(INCS) $(FIXEDFLAGS) $(FFLAGS) $(FC_AUTO_R8)'

cospsimulator_intr.o: $(COSP_LIBDIR)/libcosp.a
endif

ifdef FV3CORE_LIBDIR
$(FV3CORE_LIBDIR)/libfv3core.a: $(LIBROOT)/libfms.a
	$(MAKE) -C $(FV3CORE_LIBDIR) complib COMPLIB='$(FV3CORE_LIBDIR)/libfv3core.a' F90='$(FC)' CC='$(CC)' FFLAGS='$(FFLAGS) $(FC_AUTO_R8)'  CFLAGS='$(CFLAGS)' INCLDIR='$(INCLDIR)' FC_TYPE='$(COMPILER)'

dyn_grid.o: $(FV3CORE_LIBDIR)/libfv3core.a
endif

ifdef MPAS_LIBDIR
  ABS_INSTALL_SHAREDPATH = $(abspath $(INSTALL_SHAREDPATH))
  ABS_ESMF_PATH = $(abspath $(INSTALL_SHAREDPATH))
  ifeq ($(PIO_VERSION),2)
    PIODEF := -DUSE_PIO2
  endif
  INCLDIR+=-I$(MPAS_LIBDIR)
  INCLDIR+=-I$(abspath $(EXEROOT)/atm/obj)

# To ensure that the MPAS-A dycore library is always updated whenever dycore source
# files are changed, the libmpas target here is declared as PHONY; under ideal circumstances
# this isn't necessary, since libmpas should never be an actual file (the library that is created
# is named libmpas.a), but adding the PHONY declaration provides an extra bit of safety
.PHONY: libmpas
# The CASEROOT, COMPILER and MACH are added so that the Depends file could be visible to
# the MPAS dycore.
# The GPUFLAGS is added so that the GPU flags defined in ccs_config_cesm could also be
# used to build the MPAS dycore if needed.
libmpas: cam_abortutils.o physconst.o
	$(MAKE) -C $(MPAS_LIBDIR) CC="$(CC)" FC="$(FC)" PIODEF="$(PIODEF)" \
	FFLAGS='$(FREEFLAGS) $(FFLAGS)' GPUFLAGS='$(GPUFLAGS)' \
	CASEROOT='$(CASEROOT)' COMPILER='$(COMPILER)' MACH='$(MACH)' \
	FCINCLUDES='$(INCLDIR) $(INCS) -I$(ABS_INSTALL_SHAREDPATH)/include -I$(ABS_ESMF_PATH)/include'

dyn_comp.o: libmpas
dyn_grid.o: libmpas
endif

endif

# System libraries (netcdf, mpi, pnetcdf, esmf, trilinos, etc.)
ifeq ($(NETCDF_SEPARATE), FALSE)
   ifeq (,$(findstring $(LIB_NETCDF),$(SLIBS)))
      SLIBS += -L$(LIB_NETCDF) -lnetcdff -lnetcdf
  endif
else ifeq ($(NETCDF_SEPARATE), TRUE)
    ifeq (,$(findstring $(LIB_NETCDF_C), $(SLIBS)))
      SLIBS += -L$(LIB_NETCDF_FORTRAN) -L$(LIB_NETCDF_C) -lnetcdff -lnetcdf
    else ifeq (,$(findstring $(LIB_NETCDF_FOTRAN), $(SLIBS)))
      SLIBS += -L$(LIB_NETCDF_FORTRAN) -L$(LIB_NETCDF_C) -lnetcdff -lnetcdf
    endif
endif

ifdef LAPACK_LIBDIR
   SLIBS += -L$(LAPACK_LIBDIR) -llapack -lblas
endif
ifdef LIB_MPI
   ifndef MPI_SERIAL_PATH
     ifndef MPI_LIB_NAME
	SLIBS += -L$(LIB_MPI) -lmpi
     else
	SLIBS += -L$(LIB_MPI) -l$(MPI_LIB_NAME)
     endif
   endif
endif

# Add xios libraries for NEMO
ifdef XIOS_PATH
  SLIBS += -L$(XIOS_PATH)/lib -lxios -lstdc++
endif

# Add PETSc libraries
ifeq ($(strip $(USE_PETSC)), TRUE)
  SLIBS += ${PETSC_LIB}
endif

# Add MOAB libraries.  These are defined in the MOAB_LINK_LIBS env var that was included above
ifeq ($(strip $(USE_MOAB)), TRUE)
  SLIBS += $(IMESH_LIBS)
endif

# Remove arch flag if it exists
F90_LDFLAGS := $(filter-out -arch%,$(LDFLAGS))
ifdef GPUFLAGS
  F90_LDFLAGS += $(GPUFLAGS)
endif

# Machine stuff to appear last on the link step
ifndef MLIBS
     MLIBS  :=
endif

#------------------------------------------------------------------------------
# Drive configure scripts for support libraries
#------------------------------------------------------------------------------

ifneq ("$(wildcard $(SRCROOT)/libraries)","")
    EXTERN_PATH = $(SRCROOT)/libraries
else
    EXTERN_PATH = $(SRCROOT)/externals
endif

$(SHAREDLIBROOT)/$(SHAREDPATH)/mpi-serial/Makefile.conf:
	@echo "SHAREDLIBROOT |$(SHAREDLIBROOT)| SHAREDPATH |$(SHAREDPATH)|"; \
	$(CONFIG_SHELL) $(EXTERN_PATH)/mpi-serial/configure $(CONFIG_ARGS) --srcdir $(EXTERN_PATH)/mpi-serial

ifndef IO_LIB_SRCROOT
  ifndef PIO_SRCROOT
    PIO_SRCROOT = $(EXTERN_PATH)
  endif

  PIO_SRC_DIR = $(PIO_SRCROOT)/parallelio
else
  PIO_SRC_DIR = $(IO_LIB_SRCROOT)/$(IO_LIB_v$(PIO_VERSION)_SRCDIR)
endif

# This is a pio2 library
PIOLIB = $(PIO_LIBDIR)/libpiof.a $(PIO_LIBDIR)/libpioc.a
PIOLIBNAME = -lpiof -lpioc

GPTLLIB = $(GPTL_LIBDIR)/libgptl.a

ULIBS += -L$(INSTALL_SHAREDPATH)/lib

ULIBS += -lcsm_share -L$(INSTALL_SHAREDPATH)/lib $(PIOLIBNAME) -lgptl

#------------------------------------------------------------------------------
# Drive cmake script for cism and pio
#------------------------------------------------------------------------------

ifndef CMAKE_OPTS
  CMAKE_OPTS :=
endif
# note that the fortran flags include neither the FREEFLAGS nor the
# FIXEDFLAGS, so that both free & fixed code can be built (cmake
# doesn't seem to be able to differentiate between free & fixed
# fortran flags)
CMAKE_OPTS += -Wno-dev -D CMAKE_Fortran_FLAGS:STRING="$(FFLAGS) $(EXTRA_PIO_FPPDEFS) $(INCLDIR)" \
	      -D CMAKE_C_FLAGS:STRING="$(CFLAGS) $(EXTRA_PIO_CPPDEFS) $(INCLDIR)" \
	      -D CMAKE_CXX_FLAGS:STRING="$(CXXFLAGS) $(EXTRA_PIO_CPPDEFS) $(INCLDIR)" \
	      -D CMAKE_VERBOSE_MAKEFILE:BOOL=ON \
	      -D GPTL_PATH:STRING=$(INSTALL_SHAREDPATH) \
	      -D PIO_ENABLE_TESTS:BOOL=OFF \
	      -D PIO_USE_MALLOC:BOOL=ON \
	      -D USER_CMAKE_MODULE_PATH:LIST="$(CIMEROOT)/CIME/non_py/src/CMake;$(EXTERN_PATH)/parallelio/cmake"

ifdef ADIOS2_DIR
	CMAKE_OPTS += -D WITH_ADIOS2:BOOL=ON
endif
ifeq ($(DEBUG), TRUE)
  CMAKE_OPTS += -D PIO_ENABLE_LOGGING=ON
endif

# Allow for separate installations of the NetCDF C and Fortran libraries
ifeq ($(NETCDF_SEPARATE), FALSE)
  CMAKE_OPTS += -D NetCDF_PATH:PATH=$(NETCDF_PATH)
else ifeq ($(NETCDF_SEPARATE), TRUE)
  # NETCDF_Fortran_DIR points to the separate
  # installation of Fortran NetCDF for PIO
  CMAKE_OPTS += -D NetCDF_C_PATH:PATH=$(NETCDF_C_PATH) \
		-D NetCDF_Fortran_PATH:PATH=$(NETCDF_FORTRAN_PATH)
endif

ifdef ZLIB_PATH
	CMAKE_OPTS += -D LIBZ_PATH:STRING="$(ZLIB_PATH)"
endif

ifdef SZIP_PATH
	CMAKE_OPTS += -D SZIP_PATH:STRING="$(SZIP_PATH)"
endif

ifdef HDF5_PATH
	CMAKE_OPTS += -D HDF5_PATH:STRING="$(HDF5_PATH)"
endif

ifdef PNETCDF_PATH
	CMAKE_OPTS += -D PnetCDF_PATH:STRING="$(PNETCDF_PATH)"
else
	CMAKE_OPTS += -D WITH_PNETCDF:LOGICAL=FALSE -D PIO_USE_MPIIO:LOGICAL=FALSE
endif
ifdef PIO_FILESYSTEM_HINTS
	CMAKE_OPTS += -D PIO_FILESYSTEM_HINTS:STRING="$(PIO_FILESYSTEM_HINTS)"
endif
ifeq ($(MPILIB),mpi-serial)
	CMAKE_OPTS += -D PIO_USE_MPISERIAL=TRUE -D MPISERIAL_PATH=$(INSTALL_SHAREDPATH)
endif

# This captures the many cism-specific options to cmake
CMAKE_OPTS += $(USER_CMAKE_OPTS)

# CMake doesn't seem to like it when you define compilers via -D
# CMAKE_C_COMPILER, etc., when you rerun cmake with an existing
# cache. So doing this via environment variables instead.
ifndef CMAKE_ENV_VARS
  CMAKE_ENV_VARS :=
endif
CMAKE_ENV_VARS += CC=$(CC) \
		  CXX=$(CXX) \
		  FC=$(FC) \
		  LDFLAGS="$(LDFLAGS)"

F90_LDFLAGS += $(FFLAGS)

# We declare GLCMakefile to be a phony target so that cmake is
# always rerun whenever invoking 'make GLCMakefile'; this is
# desirable to pick up any new source files that may have been added
.PHONY: GLCMakefile
GLCMakefile:
	cd $(GLC_DIR); \
	$(CMAKE_ENV_VARS) cmake $(CMAKE_OPTS) $(GLCROOT)/source_cism

$(PIO_LIBDIR)/Makefile:
	cd $(PIO_LIBDIR); \
	$(CMAKE_ENV_VARS) cmake $(CMAKE_OPTS) $(PIO_SRC_DIR)

FTMakefile:
	cd $(FTORCH_DIR)
	$(CMAKE_ENV_VARS) cmake $(CMAKE_OPTS) $(FTORCH_SRC_DIR)

#-------------------------------------------------------------------------------
# Build & include dependency files
#-------------------------------------------------------------------------------

touch_filepath:
	touch Filepath

# Get list of files and build dependency file for all .o files
#   using perl scripts mkSrcfiles and mkDepends
# if a source is of form .F90.in strip the .in before creating the list of objects
SOURCES := $(shell cat Srcfiles)
BASENAMES := $(basename $(basename $(SOURCES)))
OBJS    := $(addsuffix .o, $(BASENAMES))
INCS    := $(foreach dir,$(cpp_dirs),-I$(dir))

CURDIR := $(shell pwd)

Depends: Srcfiles Deppath
	$(CASETOOLS)/mkDepends $(USER_MKDEPENDS_OPTS) Deppath Srcfiles > $@

Deppath: Filepath
	$(CP) -f Filepath $@
	@echo "$(MINCROOT)" >> $@

Srcfiles: Filepath
	$(CASETOOLS)/mkSrcfiles

Filepath:
	@echo "$(VPATH)" > $@

#-------------------------------------------------------------------------------
# echo file names, paths, compile flags, etc. used during build
#-------------------------------------------------------------------------------

db_files:
	@echo " "
	@echo "* VPATH   := $(VPATH)"
	@echo "* INCS    := $(INCS)"
	@echo "* OBJS    := $(OBJS)"
db_flags:
	@echo " "
	@echo "* cc      := $(CC)  $(CFLAGS) $(INCS) $(INCLDIR)"
	@echo "* .F.o    := $(FC)  $(FFLAGS) $(FIXEDFLAGS) $(INCS) $(INCLDIR)"
	@echo "* .F90.o  := $(FC)  $(FFLAGS) $(FREEFLAGS) $(INCS) $(INCLDIR)"

#-------------------------------------------------------------------------------
# Rules used for the tests run by "configure -test"
#-------------------------------------------------------------------------------

test_fc: test_fc.o
	$(LD) -o $@ test_fc.o $(F90_LDFLAGS)
ifeq ($(NETCDF_SEPARATE), FALSE)
test_nc: test_nc.o
	$(LD) -o $@ test_nc.o -L$(LIB_NETCDF) -lnetcdff -lnetcdf $(F90_LDFLAGS)
else ifeq ($(NETCDF_SEPARATE), TRUE)
test_nc: test_nc.o
	$(LD) -o $@ test_nc.o -L$(LIB_NETCDF_FORTRAN) -L$(LIB_NETCDF_C) -lnetcdff -lnetcdf $(F90_LDFLAGS)
endif
test_mpi: test_mpi.o
	$(LD) -o $@ test_mpi.o $(F90_LDFLAGS)
test_esmf: test_esmf.o
	$(LD) -o $@ test_esmf.o $(F90_LDFLAGS)

#-------------------------------------------------------------------------------
# create list of component libraries - hard-wired for current ccsm components
#-------------------------------------------------------------------------------
ifeq ($(COMP_LND),clm)
  USE_SHARED_CLM=TRUE
else
  USE_SHARED_CLM=FALSE
endif

ifeq ($(USE_SHARED_CLM),FALSE)
  LNDOBJDIR = $(EXEROOT)/lnd/obj
  LNDLIBDIR=$(LIBROOT)
  ifeq ($(COMP_LND),clm)
    LNDLIB := libclm.a
  else
    LNDLIB := liblnd.a
  endif
  INCLDIR += -I$(LNDOBJDIR)
else
  LNDLIB := libclm.a
  LNDOBJDIR = $(SHAREDLIBROOT)/$(SHAREDPATH)/clm/obj
  LNDLIBDIR = $(EXEROOT)/$(SHAREDPATH)/lib
  INCLDIR += -I$(INSTALL_SHAREDPATH)/include
  ifeq ($(COMP_NAME),clm)
    INCLUDE_DIR = $(INSTALL_SHAREDPATH)/include
  endif
endif
ifeq ($(LND_PRESENT),TRUE)
  INCLDIR += -I$(LNDOBJDIR)
endif
ifeq ($(COMP_GLC), cism)
  ULIBDEP += $(CISM_LIBDIR)/libglimmercismfortran.a
endif
ifeq ($(OCN_SUBMODEL),moby)
  ULIBDEP += $(LIBROOT)/libmoby.a
endif

ifdef COSP_LIBDIR
  ifeq ($(CIME_MODEL),cesm)
    ULIBDEP += $(COSP_LIBDIR)/libcosp.a
    # Additionally include RTTOV if asked via CAM_CONFIG_OPTS.
    ifeq ($(findstring -rttov,$(CAM_CONFIG_OPTS)),-rttov)
      ULIBDEP += $(COSP_LIBDIR)/librttov_wrapper.a $(COSP_LIBDIR)/librttov_mw_scatt.a $(COSP_LIBDIR)/librttov_brdf_atlas.a $(COSP_LIBDIR)/librttov_emis_atlas.a  $(COSP_LIBDIR)/librttov_other.a $(COSP_LIBDIR)/librttov_parallel.a $(COSP_LIBDIR)/librttov_coef_io.a $(COSP_LIBDIR)/librttov_hdf.a $(COSP_LIBDIR)/librttov_main.a
    endif
  endif
endif

ifdef FV3CORE_LIBDIR
  ULIBDEP += $(FV3CORE_LIBDIR)/libfv3core.a
  ULIBDEP += $(LIBROOT)/libfms.a
endif

ifdef MPAS_LIBDIR
  ULIBDEP += $(MPAS_LIBDIR)/libmpas.a
endif

ifndef CLIBS
   ifdef ULIBDEP
     # For each occurrence of something like /path/to/foo/libbar.a in ULIBDEP,
     # CLIBS will contain -L/path/to/foo -lbar
     CLIBS := $(foreach LIBDEP,$(strip $(ULIBDEP)), -L$(dir $(LIBDEP)) $(patsubst lib%.a,-l%,$(notdir $(LIBDEP))))
   endif
endif

# libcsm_share.a is in ULIBDEP, but -lcsm_share is in ULIBS rather than CLIBS,
# so this needs to be added after creating CLIBS above

CSMSHARELIB = $(INSTALL_SHAREDPATH)/lib/libcsm_share.a
ULIBDEP += $(CSMSHARELIB)

GENF90 ?= $(CIMEROOT)/CIME/non_py/externals/genf90/genf90.pl

#-------------------------------------------------------------------------------
# build rules:
#-------------------------------------------------------------------------------

.SUFFIXES:
.SUFFIXES: .F90 .F .f90 .f .c .cpp .o .in

ifeq ($(MPILIB),mpi-serial)
  ifdef MPI_SERIAL_PATH
    MPISERIAL = $(MPI_SERIAL_PATH)/lib/libmpi-serial.a
    MLIBS += -L$(MPI_SERIAL_PATH)/lib -lmpi-serial
    CMAKE_OPTS += -DMPI_C_INCLUDE_PATH=$(MPI_SERIAL_PATH)/include \
      -DMPI_Fortran_INCLUDE_PATH=$(MPI_SERIAL_PATH)/include \
      -DMPI_C_LIBRARIES=$(MPI_SERIAL_PATH)/lib/libmpi-serial.a \
      -DMPI_Fortran_LIBRARIES=$(MPI_SERIAL_PATH)/lib/libmpi-serial.a
  else
    MPISERIAL = $(INSTALL_SHAREDPATH)/lib/libmpi-serial.a
    MLIBS += -L$(INSTALL_SHAREDPATH)/lib -lmpi-serial
    CMAKE_OPTS += -DMPI_C_INCLUDE_PATH=$(INSTALL_SHAREDPATH)/include \
      -DMPI_Fortran_INCLUDE_PATH=$(INSTALL_SHAREDPATH)/include \
      -DMPI_C_LIBRARIES=$(INSTALL_SHAREDPATH)/lib/libmpi-serial.a \
      -DMPI_Fortran_LIBRARIES=$(INSTALL_SHAREDPATH)/lib/libmpi-serial.a
  endif
endif

$(PIOLIB) : $(MPISERIAL) $(GPTLLIB)

$(CSMSHARELIB): $(PIOLIB) $(GPTLLIB)

ifneq ($(findstring csm_share,$(COMP_NAME)),csm_share)
  $(OBJS):  $(CSMSHARELIB)
else
  complib: install_lib
endif

install_lib: $(COMPLIB)
	$(CP) -p $(COMPLIB) $(CSMSHARELIB)
	$(CP) -p *.$(MOD_SUFFIX) *.h $(INCLUDE_DIR)

# This rule writes the include flags and the link flags used in the $(EXEC_SE) rule below
# It expects the variable OUTPUT_FILE to be defined
# Set COMP_NAME=driver to get the same flags as are used when building the driver
.PHONY: write_include_and_link_flags
write_include_and_link_flags:
	@$(RM) -f $(OUTPUT_FILE)
	@echo CIME_CSM_SHR_INCLUDE = $(CSM_SHR_INCLUDE) >> $(OUTPUT_FILE)
	@echo CIME_ESMF_F90COMPILEPATHS = $(ESMF_F90COMPILEPATHS) >> $(OUTPUT_FILE)
	@echo CIME_INCLDIR = $(INCLDIR) >> $(OUTPUT_FILE)
	@echo CIME_INCS = $(INCS) >> $(OUTPUT_FILE)
	@echo CIME_CLIBS = $(CLIBS) >> $(OUTPUT_FILE)
	@echo CIME_ULIBS = $(ULIBS) >> $(OUTPUT_FILE)
	@echo CIME_SLIBS = $(SLIBS) >> $(OUTPUT_FILE)
	@echo CIME_MLIBS = $(MLIBS) >> $(OUTPUT_FILE)
	@echo CIME_F90_LDFLAGS = $(F90_LDFLAGS) >> $(OUTPUT_FILE)

# If variables are added to this rule, similar changes should be made in the write_link_flags rule above
$(EXEC_SE): $(OBJS) $(ULIBDEP) $(CSMSHARELIB) $(PIOLIB) $(GPTLLIB)
	$(LD) -o $(EXEC_SE) $(OBJS) $(CLIBS) $(ULIBS) $(SLIBS) $(MLIBS) $(F90_LDFLAGS)

$(COMPLIB): $(OBJS)
	$(AR) $(ARFLAGS) $(COMPLIB) $(OBJS)

.c.o:
	$(CC) -c $(INCLDIR) $(INCS) $(CFLAGS)  $<
.F.o:
	$(FC) -c $(INCLDIR) $(INCS) $(FFLAGS) $(FIXEDFLAGS) $<
.f.o:
	$(FC) -c $(INCLDIR) $(INCS) $(FFLAGS) $(FIXEDFLAGS) $<
.f90.o:
	$(FC) -c $(INCLDIR) $(INCS) $(FFLAGS) $(FREEFLAGS)  $<
.F90.o:
	$(FC) -c $(INCLDIR) $(INCS) $(FFLAGS) $(FREEFLAGS) $(CONTIGUOUS_FLAG) $<
.cpp.o:
	$(CXX) -c $(INCLDIR) $(INCS) $(CXXFLAGS)  $<

%.F90: %.F90.in
	$(GENF90) $< > $@

clean_dependsatm:
	$(RM) -f $(EXEROOT)/atm/obj/Srcfiles

clean_dependscpl:
	$(RM) -f $(EXEROOT)/cpl/obj/Srcfiles

clean_dependsocn:
	$(RM) -f $(EXEROOT)/ocn/obj/Srcfiles

clean_dependswav:
	$(RM) -f $(EXEROOT)/wav/obj/Srcfiles

clean_dependsiac:
	$(RM) -f $(EXEROOT)/iac/obj/Srcfiles

clean_dependsglc:
	$(RM) -f $(EXEROOT)/glc/obj/Srcfiles

clean_dependsice:
	$(RM) -f $(EXEROOT)/ice/obj/Srcfiles

clean_dependsrof:
	$(RM) -f $(EXEROOT)/rof/obj/Srcfiles

clean_dependsesp:
	$(RM) -f $(EXEROOT)/esp/obj/Srcfiles

clean_dependslnd:
	$(RM) -f $(LNDOBJDIR)/Srcfiles

clean_dependscsmshare:
	$(RM) -f $(SHAREDLIBROOT)/$(SHAREDPATH)/csm_share/Srcfiles

clean_depends: clean_dependsatm clean_dependscpl clean_dependswav clean_dependsglc clean_dependsice clean_dependsrof clean_dependslnd clean_dependscsmshare clean_dependsesp clean_dependsiac


cleanatm:
	$(RM) -f $(LIBROOT)/libatm.a
	$(RM) -fr $(EXEROOT)/atm/obj

cleancpl:
	$(RM) -fr $(EXEROOT)/cpl/obj

cleanocn:
	$(RM) -f $(LIBROOT)/libocn.a
	$(RM) -fr $(EXEROOT)/ocn/obj

cleanwav:
	$(RM) -f $(LIBROOT)/libwav.a
	$(RM) -fr $(EXEROOT)/wav/obj

cleaniac:
	$(RM) -f $(LIBROOT)/libiac.a
	$(RM) -fr $(EXEROOT)/iac/obj

cleanesp:
	$(RM) -f $(LIBROOT)/libesp.a
	$(RM) -fr $(EXEROOT)/esp/obj

cleanglc:
	$(RM) -f $(LIBROOT)/libglc.a
	$(RM) -fr $(EXEROOT)/glc

cleanice:
	$(RM) -f $(LIBROOT)/libice.a
	$(RM) -fr $(EXEROOT)/ice/obj

cleanrof:
	$(RM) -f $(LIBROOT)/librof.a
	$(RM) -fr $(EXEROOT)/rof/obj

cleanlnd:
	$(RM) -f $(LIBROOT)/$(LNDLIB)
	$(RM) -fr $(LNDOBJDIR)

cleancsmshare:
	$(RM) -f $(CSMSHARELIB)
	$(RM) -fr $(SHAREDLIBROOT)/$(SHAREDPATH)/csm_share

cleanpio:
	$(RM) -f $(PIO_LIBDIR)/libpio*
	$(RM) -fr $(SHAREDLIBROOT)/$(SHAREDPATH)/pio

cleangptl:
	$(RM) -f $(GPTLLIB)
	$(RM) -fr $(SHAREDLIBROOT)/$(SHAREDPATH)/gptl

clean: cleanatm cleanocn cleanwav cleanglc cleanice cleanrof cleanlnd cleanesp cleaniac

realclean: clean cleancsmshare cleanpio cleangptl

# the if-tests prevent DEPS files from being created when they're not needed
ifneq ($(MAKECMDGOALS), db_files)
ifneq ($(MAKECMDGOALS), db_flags)
ifeq (,$(findstring clean,$(MAKECMDGOALS)))
    -include Depends $(CASEROOT)/Depends.$(COMPILER) $(CASEROOT)/Depends.$(MACH) $(CASEROOT)/Depends.$(MACH).$(COMPILER)
endif
endif
endif
ifeq ($(COMP_NAME),csm_share)
  shr_assert_mod.mod: shr_assert_mod.o
endif

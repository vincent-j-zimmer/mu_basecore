# Override Validation Plugin

Module Level Override Validation Plugin and Override Tag Creation Tool

## About

OverrideValidation is a UEFI Build Plugin and Command Line Tool used to create
linkage between overriding and overridden modules and parse INF files referenced
in platform DSC files during build process and then produce a text report of the
module overriding status.  The text report then allows deeper analysis of the
Overriding Hierarchy, the Override Linkage Validity, the Override Linkage Ages,
and overall breakdown of usage.

### UEFI Build Plugin

When used in the plugin capacity this plugin will do its override linkage
validation work in the do_pre_build function.  This plugin uses the following
variables from the build environment:

 1. ACTIVE_PLATFORM - [REQUIRED] - must be workspace relative or package path
    relative pointing to the target platform dsc file, otherwise this validation
    will not run
 1. BUILD_OUTPUT_BASE - [REQUIRED] - must be an absolute path specified to store
    override log at $(BUILD_OUTPUT_BASE)/OVERRIDELOG.TXT, otherwise no report
    will be generated
 1. BUILDSHA - [OPTIONAL] - should have valid commit sha value for report
    purpose, if not provided, 'None' will be used for the corresponding field
 1. PRODUCT_NAME - [OPTIONAL] - should give friendly product name, if not
    provided, 'None' will be used for the corresponding field
 1. BUILDID_STRING - [OPTIONAL] - should give friendly version string of
    firmware version, if not provided, 'None' will be used for the corresponding
    field

This tool provides two types of validation, determined by the type of **tags** included in the overriding module:

- **Override**: Override validation, as indicated by override tags, intends to enforce the validity of a linkage. Thus if
the target that is overridden is either not found or has an change since the last linkage update, the build will break.
- **Track**: Track validation, indicated by track tags from tracking modules, intends to soft-track updates of certain
module with various flavors across upstream changes. This validation will ignore this tag if the corresponding target
is not found. If a single target is found in multiple track tags, there must be one and only one linkage that matches
the current status of target, otherwise build will break.

  *Note*: If one module contains one or more track tags, at least one tracked target needs to be found, otherwise build
  will break.

### Command Line Tool

When used as a command line tool, this tool takes the absolute path of workspace
(the root directory of Devices repo) as well as the absolute path of overridden
module's inf file and then generate a screen-print line for users to include in
overriding modules in order to create override linkage. Check the required
parameters by using the -h option for command line argument details.

The override can also be used on the Active Platform DSC or the Flash Definition
FDF defined by the DSC.

### Example

Command to generate an override record:

``` cmd
OverrideValidation.py -w C:\Repo -m C:\Repo\SM_UDK\MdePkg\Library\BaseMemoryLib\BaseMemoryLib.inf
```

Override record to be included in overriding module's inf:

``` cmd
#Override : 00000001 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | cc255d9de141fccbdfca9ad02e0daa47 | 2018-05-09T17-54-17
```

Command to generate a track record:

``` cmd
OverrideValidation.py --track -w C:\Repo -m C:\Repo\SM_UDK\MdePkg\Library\BaseMemoryLib\BaseMemoryLib.inf
```

Track record to be included in tracking module's inf:

``` cmd
#Track : 00000002 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | 5bca19892b2e9f4c00a74041fa6b1eab | 2021-12-07T04-25-21 | 5c76ea08864294e11f8d7d1ac2ccf76c72673c8f
```

Track records to be included in tracking multiple flavors of the same module's inf (you should do not need this in a
perfect world):

``` cmd
# Production build
#Track : 00000002 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | 5bca19892b2e9f4c00a74041fa6b1eab | 2021-12-07T04-25-21 | 5c76ea08864294e11f8d7d1ac2ccf76c72673c8f
# Debug build
#Track : 00000002 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | dbfc0ece0cb8fa499ac2141c80107926 | 2022-02-09T00-31-30 | fd114d321703a32c4684d8411ba0fe7dd7012c14
```

Command to generate an override record for a target file or directory:

``` cmd
OverrideValidation.py -w C:\Repo -t C:\Repo\MU_BASECORE\MdeModulePkg\Library\BaseSerialPortLib16550
```

Override record to be included in overriding module's inf:

``` cmd
#Override : 00000002 | MdeModulePkg/Library/BaseSerialPortLib16550 | 140759cf30a73b02f48cc1f226b015d8 | 2021-12-07T05-30-10 | fa99a33fdb7e8bf6063513fddb807105ec2fad81
```

Command to generate a deprecation record:

``` cmd
OverrideValidation.py -w C:\Repo -d C:\Repo\MU_BASECORE\MdeModulePkg\BaseMemoryLib\BaseMemoryLib.inf -dr C:\Repo\MU_BASECORE\MdeModulePkg\BaseMemoryLibV2\BaseMemoryLib.inf -dt 120
```

Deprecation record to be included in the deprecated module's inf:

``` cmd
#Deprecated : 00000001 | MdeModulePkg/BaseMemoryLibV2/BaseMemoryLib.inf | 2024-02-16T04-00-28 | 120
```

Deprecation warning example:

``` cmd
WARNING - Use of Deprecated module: C:\Repo\MU_BASECORE\MdeModulePkg\BaseMemoryLib\BaseMemoryLib.inf, Please switch to:  C:\Repo\MU_BASECORE\MdeModulePkg\BaseMemoryLibV2\BaseMemoryLib.inf.
```

To disable Deprecation warnings for a given module, add it to the deprecation modules skip list.

``` cmd
self.env.SetValue("DEPRECATED_MODULES_SKIPLIST", "MdeModulePkg\BaseMemoryLib\BaseMemoryLib.inf; MdeModulePkg\MemoryAllocationLib\MemoryAllocationLib.inf", "Skip list for platforms")
```

Override log generated during pre-build process:

``` cmd
Platform:     PlatformName
Version:      123.456.7890
Date:         2018-05-11T17-56-27
Commit:       _SHA_2c9def7a4ce84ef26ed6597afcc60cee4e5c92c0
State:        3/4

Overrides
----------------------------------------------------------------

OVERRIDER: MdePkg/Library/BaseMemoryLibOptDxe/BaseMemoryLibOptDxe.inf
ORIGINALS:
  + MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | SUCCESS | 2 days

OVERRIDER: PlatformNamePkg/Library/NvmConfigLib/NvmConfigLib.inf
ORIGINALS:
  + MdeModulePkg/Bus/Pci/NvmExpressDxe/NvmExpressDxe.inf | MISMATCH | 35 days
  | Current State: 62929532257365b261080b7e7b1c4e7a | Last Fingerprint: dc9f5e3af1efbac6cf5485b672291903
  + MdePkg/Library/BaseMemoryLibOptDxe/BaseMemoryLibOptDxe.inf | SUCCESS | 0 days
  + MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | SUCCESS | 2 days
  + MdeModulePkg/Library/BaseSerialPortLib16550 | SUCCESS | 7 days

```

Command to regenerate the override records in a given .inf file:

``` cmd
OverrideValidation.py -w c:\src -r c:\src\FooPkg\OverridingModule.inf
```

an example of the diff produced when using -r:

``` diff
diff --git a/FooPkg/OverridingModule.inf b/FooPkg/OverridingModule.inf
index 2d4ca47299..90da207a39 100644
--- a/FooPkg/OverridingModule.inf
+++ b/FooPkg/OverridingModule.inf
@@ -8,7 +8,7 @@
 #
 #
-#Override : 00000002 | BarPkg/OverridenModule.inf | 4f7eed98e3c084eecdff5fa2e1e57db1 | 2021-11-23T21-41-21 | 44b40c0358489da6c444e7cfb2be26e56b7c16a1
+#Override : 00000002 | BarPkg/OverridenModule.inf | 143b08782a2abc620d1eb57461c6e290 | 2022-03-10T23-09-45 | 6f8c53a3fcd79b202c708e7aa58256cafbf24bc4
 #
```

## Versions

There are two versions of the override format and one version of the deprecation format

### Override Version 1

``` cmd
OVERRIDE_FORMAT_VERSION_1 = (1, 4) # Version 1
#Override : VERSION | PATH_TO_MODULE | HASH | YYYY-MM-DDThh-mm-ss
#Override : 00000001 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | cc255d9de141fccbdfca9ad02e0daa47 | 2018-05-09T17-54-17
```

### Override Version 2

``` cmd
OVERRIDE_FORMAT_VERSION_2 = (2, 5) # Version 2
#Override : VERSION | PATH_TO_MODULE | HASH | YYYY-MM-DDThh-mm-ss | GIT_COMMIT
#Override : 00000002 | MdePkg/Library/BaseMemoryLib/BaseMemoryLib.inf | cc255d9de141fccbdfca9ad02e0daa47 | 2018-05-09T17-54-17 | 575096df6a
```

Version 2 includes a second hash at the end, which is the git commit that the
upstream was last updated. This allows to tools to do a `git diff` between what
you currently have and what is in the tree. It currently only diffs the
overridden file (the INF or DSC) and the overriding file.

### Deprecation Version 1

``` cmd
# DEPRECATION_FORMAT_VERSION_1 = (1, 4) # Version 1
#Deprecated : VERSION | PATH_TO_NEW_MODULE_TO_USE | YYYY-MM-DDThh-mm-ss | DEPRECATION_TIMELINE

example:
#Deprecated : 00000001 | MdeModulePkg/BaseMemoryLibV2/BaseMemoryLib.inf | 2024-02-16T04-00-28 | 90
```

## Copyright & License

Copyright (c) Microsoft Corporation.
SPDX-License-Identifier: BSD-2-Clause-Patent

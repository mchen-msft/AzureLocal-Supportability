# Symptoms

Operations, such as Add-Server or Update, which invoke InstallNetRuntimes will 
fail if a newer version of .NET Core components are already installed with an 
error matching the pattern:

```PlainText
Type 'InstallNetRuntimesLocally' of Role 'ExternalDependenciesRole' raised an exception: Could not find 'Microsoft.NETCore.App 10.0.x'
```

or

```PlainText
Type 'InstallNetRuntimesLocally' of Role 'ExternalDependenciesRole' raised an exception: Could not find 'Microsoft.AspNetCore.App 10.0.x'
```

The error message indicates the version of .NETCore that is expected to be
installed did not install.

# Issue Validation

Run `dotnet --list-runtimes` to confirm that a newer version of the conflicting
package is installed, specifically look at versions for the following runtimes:

* Microsoft.NETCore.App 10.\*
* Microsoft.AspNetCore.App 10.\*

If a newer version of either package exists, then this TSG is applicable to
mitigating the issue.

# Cause

If a newer version of the .NETCore runtime package is installed, older versions
of the installer default to a no-op install and shortcircuits to a success state.

The post install validation check is pinned to the specific
version, and since the installer did nothing, will raise a failure.

This will be addressed in future releases and the version check will be changed to a minimum
version check.

# Mitigation Details

## 1. Identify the right version of the runtime installer

**The error message indicates the version of the .NET Core runtime that needs to get installed.**

Here is a table for known version for releases of Azure Local for reference, but **confirm with the error message
which version needs to be installed**.

|Azure Local Build | Required Runtime Version |
|------------------|--------------------------|
| 2604             | 10.0.6                   |
| 2605             | 10.0.8                   |
| 2606             | 10.0.9                   |
| 2607             | 10.0.10                  |
| 2608             | 10.0.11                  |
| 2609             | 10.0.12                  |

Also run `Get-ChildItem -Path c:\NugetStore\Microsoft.DotNet10.Runtime.Installer*` confirm the installer is present.

## 2. Identify conflicting versions on nodes

Run `dotnet --list-runtimes` to list all installed runtimes on every node. Make note of every newer
version of the runtime found. These newer versions are preventing the required version from being
installed.

## 3. Find the uninstallers for the conflicting runtimes

Run the following to find the cached installer for the conflicting .NET Core runtimes on every node.
The paths should be the same across all nodes.

```Powershell
# Suppose the conflicting version is 10.0.10 - Update this to whatever found in the previous step.
$ConflictVersion = "10.0.10"
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\aspnetcore-runtime-$ConflictVersion*.exe"
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\dotnet-runtime-$ConflictVersion*.exe"
```

## 4. Run the uninstalls

Run the found uninstallers on every node with the following command line parameters:
```Powershell
# Replace <Found Path> with path found in previous step
& "<Found Path>\aspnetcore-runtime-10.0.10-win-x64.exe" /uninstall /quiet
& "<Found Path>\dotnet-runtime-10.0.10-win-x64.exe" /uninstall /quiet
```

The uninstaller paths should be the same across all nodes.

## 5. Confirm the conflicting newer runtimes are no longer installed:

Run `dotnet --list-runtimes` on every node again to make sure that the newer conflicting runtime
has been removed.

## 6. Manually install the runtimes.

To be certain that the required runtime is installed, manually run the installers for the required runtimes.

```Powershell
# Assuming the required version is 10.0.6. The required version will be in the error message.
$RequiredVersion = "10.0.6"

# 1. Confirm the required installer is present on node
Get-ChildItem "c:\nugetstore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\dotnet-runtime-$RequiredVersion*"
Get-ChildItem "c:\nugetstore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\aspnetcore-runtime-$RequiredVersion*"

# If not present on node, search other nodes and copy over as needed

# 2. Run the found installers
& "c:\NugetStore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\dotnet-runtime-$RequiredVersion-win-x64.exe" /install /quiet
& "c:\NugetStore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\aspnetcore-runtime-$RequiredVersion-win-x64.exe" /install /quiet
```

## 7. Confirm the required runtimes are installed (and force a repair if not)

```Powershell
dotnet --list-runtimes

# Make sure the right version of the runtime shows up.  For example, if the right version is 10.0.6, then
# the following strings should appear in the output
# "Microsoft.AspNetCore.App 10.0.6 [C:\Program Files\dotnet\shared\Microsoft.AspNetCore.App]"
# "Microsoft.NETCore.App 10.0.6 [C:\Program Files\dotnet\shared\Microsoft.NETCore.App]"
```

If the runtimes are still not found, force a repair

```Powershell
# If still not installed, run repair
& "c:\NugetStore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\dotnet-runtime-$RequiredVersion-win-x64.exe" /repair /quiet
& "c:\NugetStore\Microsoft.DotNet10.Runtime.Installer.$RequiredVersion\bin\aspnetcore-runtime-$RequiredVersion-win-x64.exe" /repair /quiet
```

and then recheck.

## 8. Resume Operation

Resume the blocked operation and the ExternalDependenciesRole operation should complete successfully this time.

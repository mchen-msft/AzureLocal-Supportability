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

Up to Azure Local 2610, the post install validation check is pinned to the specific
version, and since the installer did nothing, will raise a failure.

Starting from 2610, the version check has been relaxed to accept a minimum
version or greater.

# Mitigation Details

## Make note of the version that must get installed

The error message indicates the version of the .NET Core runtime that needs to get installed. 

## Identify conflicting versions on nodes

Run `dotnet --list-runtimes` to list all installed runtimes on every node. Make note of every newer
version of the runtime found. These newer versions are preventing the required version from being
installed.

## Find the uninstaller commands

Run the following to find the cached installer for the conflicting .NET Core runtimes on every node.
The paths should be the same across all nodes.

```Powershell
# Suppose the conflicting version is 10.0.10
$ConflictVersion = "10.0.10"
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\aspnetcore-runtime-$ConflictVersion*.exe"
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\dotnet-runtime-$ConflictVersion*.exe"
```

## Run the uninstalls

Run the found uninstallers on every node with the following command line parameters:
```
"<Found Path>\aspnetcore-runtime-10.0.10-win-x64.exe" /uninstall /quiet
"<Found Path>\dotnet-runtime-10.0.10-win-x64.exe" /uninstall /quiet
```

## Confirm the conflicting newer runtimes are no longer installed:

Run `dotnet --list-runtimes` on every node again to make sure that the newer conflicting runtime
has been removed.

## Resume Operation

Resume the blocked operation and the ExternalDependenciesRole operation should complete successfully this time.

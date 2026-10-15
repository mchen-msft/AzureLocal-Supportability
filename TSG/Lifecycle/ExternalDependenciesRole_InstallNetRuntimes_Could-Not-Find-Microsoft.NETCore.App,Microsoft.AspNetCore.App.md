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

# Issue Validation

The error message indicates the version of .NETCore that is expected to be
installed.

Run `dotnet --list-runtimes` to confirm that a newer version of the conflicting
package is installed, specifically look at versions for the following runtimes:

* Microsoft.NETCore.App 10.\*
* Microsoft.AspNetCore.App 10.\*

If a newer version of either package exists, then this TSG is applicable to
mitigating the issue.

# Cause

Up to Azure Local 2610, the .NET runtime version check is pinned to a version.
Starting from 2610, the version check has been relaxed to accept a minimum
version or greater.

# Mitigation Details

## Identify conflicting versions on nodes

Run `dotnet --list-runtimes` to list all installed runtimes on every node.

## Find the uninstaller command

Run the following to find the cached installer for the .NET Core runtimes on every node.
The paths should be the same across all nodes.

```Powershell
# Suppose the conflicting version is 10.0.10
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\aspnetcore-runtime-10.0.10*.exe"
Get-ChildItem -Recurse -Path "C:\ProgramData\Package Cache\*\dotnet-runtime-10.0.10*.exe"
```

## Run the uninstall

Run the found uninstallers on every node with the following command line parameters:
```
"<Found Path>\aspnetcore-runtime-10.0.10-win-x64.exe" /uninstall /quiet
"<Found Path>\dotnet-runtime-10.0.10-win-x64.exe" /uninstall /quiet
```

## Confirm the runtimes are no longer installed:

Run `dotnet --list-runtimes` on ever node again to make sure that the newer runtime
has been removed.

## Resume Operation

Resume the blocked operation and the ExternalDependenciesRole operation
should complete successfully this time.

# Using Credential Providers to Access Authenticated Feeds

NuGet 3.3 now supports Credential Providers, which enable NuGet to work seamlessly with authenticated feeds. 
After you install a Credential Provider, NuGet will automatically acquire and refresh credentials for authenticated feeds as necessary.

Note: Credential providers only work in NuGet.exe (not in dotnet restore or visual studio). There is a single credential provider that is built into the Visual Studio NuGet extension to support Visual Studio Team Services.

NuGet Credential Providers can be used in 3 ways:

1. [Globally](#installing-a-credential-provider-globally)
2. [Environment Variable](#using-a-credential-provider-from-an-environment-variable)
3. [Alongside NuGet.exe](#using-a-credential-provider-alongside-nugetexe)

## Installing a Credential Provider Globally

To make a Credential Provider available to all instances of NuGet.exe run under the current user's profile, 
add it to `%LocalAppData%\NuGet\CredentialProviders`. You may need to create the `CredentialProviders` folder.

Credential Providers can be installed at the root of the `CredentialProviders` folder or within a subfolder. If a Credential Provider has 
multiple files/assemblies, using subfolders can help keep the Providers organized.

## Using a Credential Provider from an Environment Variable

Credential Providers can also be stored anywhere and then made accessible to NuGet.exe via an environment variable. To use a Credential Provider this way, set the `%NUGET_CREDENTIALPROVIDERS_PATH%` to the location of your Provider. The variable can be a semicolon-separated list, e.g. 
`path1;path2`, if you have have multiple Credential Providers in different locations.

## Using a Credential Provider Alongside NuGet.exe

Credential Providers can also be placed in the same folder as NuGet.exe.

## Using Multiple Credential Providers

The NuGet.exe client will search the above locations, in order, for Credential Providers.
Specifically, it will search for any file that matches the pattern `CredentialProvider*.exe`
and load each provider in the order it's found. If two Credential Providers are found in the
same directory, they will be loaded in alphabetical order.

## Available Credential Providers

Some of the available Credential Providers are:

* [Visual Studio Team Services Credential Provider](https://www.visualstudio.com/get-started/package/use/auth#vsts-credential-provider)

# Creating a Credential Provider

The NuGet 3.3 client implements an internal CredentialService that is used to acquire credentials.
The CredentialService has a list of built-in and plug-in Credential Providers.
Each provider is tried sequentially until credentials are acquired.

Built-in providers, such as the existing providers to fetch credentials from nuget.config and to
prompt the user on the command line, execute in-process. 

During credential acquisition, the credential service will try plug-in credential providers in the following order, stopping as soon as credentials are acquired:

1. Credentials will be fetched from NuGet configuration files.
2. All 3rd party credential providers will be tried in the order of discovery outlined above.
3. The user will be prompted for credentials on the command line.

## Creating a Credential Provider

To create a Credential Provider, create a command-line executable that takes the inputs
specified below, acquires credentials as appropriate, then returns the appropriate exit status code
and standard output.

### Credential Provider Executable Basics

Credential Providers must follow the naming convention `CredentialProvider*.exe`., and each 
Credential Provider must:

1. Determine whether it can provide credentials for the targeted URI before initiating credential
acquisition. If the plugin cannot supply credentials for the targeted source, then it should return
status code 1 and supply no credentials.
2. Not modify nuget.config (i.e. by setting credentials there).
3. Handle HTTP Proxy configuration on their own.
The CredentialService will not pass HTTP Proxy configuration information to the plugin.
4. Encode stdout responses using UTF-8 encoding.

### Credential Provider Input Parameters

| Input parameter   | Description                                                                          |
|-------------------|--------------------------------------------------------------------------------------|
| `-Uri {value}`    | The package source Uri for which credentials will be filled.                         |
| `-NonInteractive` | If present, providers will not issue interactive prompts.                            |
| `-IsRetry`        | If present, this is a retry and the credentials were rejected on a previous attempt. Providers typically use the `IsRetry` flag to ensure that they bypass any existing cache and prompt for new credentials if possible. |

### Credential Provider Exit Status Codes

| Exit status code | Result                | Description                                                                                                                                                                                                                                                                                 |
|------------------|-----------------------|---------------------------------------------------------                                                                                                                                                                                                                                    |
| `0`              | Success               | Credentials were successfully acquired and have been written to stdout.                                                                                                                                                                                                             |               
| `1`              | ProviderNotApplicable | The current provider does not provide credentials for the given uri.                                                                                                                                                                                                                       |               
| `2`              | Failure               | The Credential Provider is the correct provider for the given Uri, but cannot provide credentials.  In this case, the CredentialService will throw an exception and the current request should fail and not be retried. A typical example would be a user cancelling an interactive login. |

### Credential Provider Standard Output

| Property                               | Notes                                                                                                               |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| `Username`                             | Username for authenticated requests.                                                                                |
| `Password`                             | Password for authenticated requests.                                                                                |
| `Message`                              | Optional details about the response.  Will only be used by NuGet to show additional error details in failure cases. |

### Example stdout

```
{ "Username" : "freddy@example.com",
  "Password" : "bwm3bcx6txhprzmxhl2x63mdsul6grctazoomtdb6kfbof7m3a3z",
  "Message"  : "" }
```

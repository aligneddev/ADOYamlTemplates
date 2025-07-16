# Azure DevOps Pipline templates

[Please visit my blog post](https://aligneddev.net/blog/) for a write up about these templates

These templates can be reused in other repositories by including them in the pipeline and passing in variable values.

These are for OnPremise environments, the Azure pipeline YAML can be found in Microsoft documentation.

I'm using the resources to show how the templates can be in a different repo than the pipelines and source code.

```yaml
resources: 
  repositories: 
  - repository: {repos name}
    name: {project name}/{repos name}
    type: git 
    ref: master #branch name

jobs:
  - template: frameworkWebBuildAndTest.yml@Shared

```

https://elanderson.net/2020/04/azure-devops-pipelines-use-yaml-across-repos/ was the initial inspiration

## Examples

The Examples folder has an example of using these templates.

### Structure

The templates need to have the code in a folder other than the root. `src/` is good. Set the rootFolder parameter = `src`.

The standard is to have:
- YourProject.sln
- .gitignore
- NuGet.config (casing is important)
- README.md
- azure-pipelines.yml
- .editorconfig for C# projects
- src/
- scripts/ (if applicable/needed)

## Settings and Secrets

Some organizations aren't using Azure and KeyVault yet 😨. They may be encrypting the web.config/appsetting.json files on the server. The approach below was from working with such an organization.

Note: **The better way to secure secrets would be to use Azure Key Vault.** The application code would establish a connection via Entra Id and certificates. See the [Microsoft Learn documentation for more details](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/administration/setup-app-key-vault-onprem).

- No secrets stored in the appsettings.json or web.configs!
- Variables are better, encrypted environment variables on the server better and Azure Key Vault would be even better
- use Variable Groups for common secrets, Variables on the pipeline for project specific
- Replace Tokens in the publish step to avoid secrets being stored in the artifacts for the build

## Configuration Transformation

https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/file-transform-v2?view=azure-pipelines&viewFallbackFrom=azure-devops

https://learn.microsoft.com/en-us/previous-versions/azure/devops/pipelines/tasks/transforms-variable-substitution?view=tfs-2018

For Framework, the configuration file is transformed based from the file on the build configuration (Development for .Net, Debug=Development for Framework. Release=Production) during the BuildAndTest Stage. Ex: If release, the values from web.Production.config are placed into web.config.

For .Net, the json file is chosen for the set ASPNETCORE_ENVIRONMENT (see the Appsettings.json section below) which is set in the build stage and added to the artifact for that environment.

Then secrets from the ADO Variable Group (shared) or ADO Variables for the Pipeline are set in the web.config/appsettings.{Environment}.json during the deployment stage (keeping the secrets out of the artifacts stored in the pipeline).

The config file is deployed to the environment. If the application is using the ConfigurationFileEncrypter, AddEncryptedJson("appsetttings.Production.json"), the config file will be encrypted upon startup. The readme has more information. 

When the FileTransform@1 task is ran, it takes the value from the Variable or Variable Group and overwrites the value in the appsettings.json/web.config. The name has to match. ex: EnvironmentTest in the web.config will be overwritten when EnvironmentTest is set in the Variable Group/Variable Pipeline. 
nested example: NestedTest.EnvironmentTest in IIS/Net

Note: Using the Variable Pipeline will only work if the same value is for all environments.

Put the Production values into the Variable Group for the pipeline and lock the secrets.

https://medium.com/@jitendra1400/azure-devops-replace-tokens-key-in-json-using-file-transform-in-yaml-pipeline-d0adcd104054 shows an example.

## Web.config

Used for .Net Framework, these templates assume that there is a web.config, web.Debug.config and web.Production.config. They use the FileTransform to replace values in the web.config based on the rules in the environment config files.

## Appsettings.json

Used for .Net, these templates assume there is an appsettings.json, appsettings.Development.json and appsettings.Production.json.

The "/p:EnvironmentName={Develpment/Release}" is set in the build phase with a FileTransform step, setting this in the web.config, by adding it in the IISPublish yml Deployment stage and exeSqlAgentPublish step.
```xml
<aspNetCore processPath="dotnet" 
            arguments=".\MyApp.dll" 
            stdoutLogEnabled="false" 
            stdoutLogFile=".\logs\stdout" 
            hostingModel="inprocess">
  <environmentVariables>
    <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
  </environmentVariables>
</aspNetCore>
```

Note: It can also be set in the environment variables on the server, but we are using the web.config
More detail: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-9.0

Note: the appsettings.Production.json file is excluded from the Development artifact with a IISWebAppDeploymentOnMachineGroup AdditionalArguments flag.

## Exes - App.config, appsettings.jwon

Framework uses App.config, .Net uses appsettings.json

the FileTransform@2 task says "This task is intended for web packages and requires a web package file. It does not work on standalone JSON files."

### Running from a SQL Server Agent

This isn't very common, but you can use SQL Server Agents to run an exe on a schedule.

ASPNETCORE_ENVIRONMENT must be set in the Sql Server Agent Step
example {networkpath}\NetConsoleApp.exe ASPNETCORE_ENVIRONMENT=Development

https://dotnet.microsoft.com/en-us/download/dotnet/11.0


dotnet tool install -g microsoft.maui.cli --prerelease

maui doctor

maui doctor --fix

https://visualstudio.microsoft.com/downloads

Run the downloaded file to start the install process

Select MAUI Devlopment and Windows Development

dotnet workload install maui --include-previews

https://code.visualstudio.com/download

create vs code profile - maui
activate maui profile

Settings
	C# Dev Kit
		Nuget: allow prerelease versions

add extensions to maui profile
```
	code --install-extension alexcvzz.vscode-sqlite
	code --install-extension humao.rest-client
	code --install-extension jaufrdevosse.litedb-vscode
	code --install-extension microsoft-aspire.aspire-vscode
	code --install-extension ms-azuretools.vscode-containers
	code --install-extension ms-azuretools.vscode-docker
	code --install-extension ms-dotnettools.csdevkit
	code --install-extension ms-dotnettools.csharp
	code --install-extension ms-dotnettools.dotnet-maui
	code --install-extension ms-dotnettools.vscode-dotnet-runtime
	code --install-extension ms-mssql.data-workspace-vscode
	code --install-extension ms-mssql.mssql
	code --install-extension ms-mssql.sql-database-projects-vscode
	code --install-extension ms-vscode-remote.remote-containers
	code --install-extension ms-vscode.remote-explorer
	code --install-extension ms-vscode.remote-server
```

add mcp servers
	@mcp
	enable mcp servers marketplace
	- Microsoft Learn
	- Microsoft Nuget
	
Switch to MAUI agent

Restart VS Code

Create a .net maui project 
	.net 11 SDK


Create global.json file
```json
{
    "sdk":{
        "version":"11.0.100-preview.7.26381.103",
        "allowPrerelease": true,
        "rollForward": "latestPatch"
    }
}
```

# OneDriveClient library for CODESYS

[![](https://img.shields.io/github/v/release/vuhuy/codesys-onedriveclient)](https://github.com/vuhuy/codesys-onedriveclient/releases)
[![](https://img.shields.io/github/release-date/vuhuy/codesys-onedriveclient)](https://github.com/vuhuy/codesys-onedriveclient/releases)
[![](https://img.shields.io/github/license/vuhuy/codesys-onedriveclient)](https://github.com/vuhuy/codesys-onedriveclient/blob/master/LICENSE)

OneDriveClient is a library to access OneDrive and SharePoint from CODESYS projects. The library does not interact directly with OneDrive and SharePoint through the Microsoft Graph API but uses [onedrive-uploader](https://github.com/virtualzone/onedrive-uploader) that needs to be available on the target system.

| Version | Recommended onedrive-uploader version | Compiler version |
| ------- | ------------------------------------- | ---------------- |
| [0.1.0](https://github.com/vuhuy/codesys-onedriveclient/releases/tag/v0.1.0) | [0.8.0](https://github.com/virtualzone/onedrive-uploader/releases/tag/v0.8.0) | [3.5.16.50](https://www.codesys.com/rss-article/release-codesys-351650.html) |
| [Main](https://github.com/vuhuy/codesys-onedriveclient/tree/main) | [0.8.0](https://github.com/virtualzone/onedrive-uploader/releases/tag/v0.8.0) | [3.5.16.50](https://www.codesys.com/rss-article/release-codesys-351650.html) |

## Features

OneDriveClient is developed to access OneDrive and SharePoint resources from CODESYS. This library is not intended for directory synchronization or retention management.

* Tested with CODESYS Control for Linux on ARM64. 
* Access OneDrive and SharePoint from CODESYS.
* Create, delete, and list directories.
* Download, upload, and delete files.

This library should work for every device running CODESYS Control regardless of the system architecture. The onedrive-uploader dependency offers multi-platform libraries. However, using this library with anything other than Linux / ARM64 remains untested. Feel free to submit an [issue report](https://github.com/vuhuy/codesys-onedriveclient/issues) if needed.

## Limitations

OneDriveClient has a few limitations by design. The directory listing function can truncate data if either the amount of direct child items surpasses the allocated limit of 1000 items, or when the captured standard output is larger than 30000 bytes.

## Getting started

This library requires onedrive-uploader to be available on the target device. To access OneDrive and SharePoint, your application needs to be registered in Azure. A configuration file for onedrive-uploader needs to be generated.

### Azure app registration

The following instructions are just an example. Always keep security in mind. Proceed with caution or consult your system administrator.

1. Log in to the [Microsoft Azure Portal](https://portal.azure.com).
2. Navigate to your Azure Active Directory.
3. Navigate to app registrations from the menu.
4. Start a new registration for your application.
   1. Enter a name for your application.
   2. Select *Accounts in any organizational directory (Any  Azure AD directory - Multitenant) and personal Microsoft accounts (e.g.  Skype, Xbox)*.
   3. Configure a Web Redirect URI to *http://localhost:53682/*.
5. Save the application (client) ID. You need this later.
6. Navigate to certificates & secrets from the menu.
7. Create a new client secret.
   1. Enter a description for your client secret.
   2. Set the desired expiration date for your client secret.
8. Save the secret value. You need this later. The secret won't be displayed again after leaving the page.
9. Navigate to API permissions from the menu.
10. Add permission for your application.
    1. Request API permissions for Microsoft Graph.
    2. Select *delegated permissions*.
    3. Select the required permissions.
       1. Application only:  `Files.ReadWrite.AppFolder`, `User.Read`, `offline_access`.
       2. Access the entire drive: `Files.Read`, `Files.ReadWrite`, `Files.Read.All`, `Files.ReadWrite.All`, `User.Read`, `offline_access`.

### Onedrive-uploader configuration

Onedrive-uploader needs to be configured before it can be used. Configuration can be done from any supported system with a web browser installed. This section assumes the system used for configuration is different from the target device running CODESYS Control.

1. Download the correct [onedrive-uploader precompiled binary file](https://github.com/virtualzone/onedrive-uploader/releases) compatible with the system used for configuration.
2. Rename the downloaded binary to `onedrive-uploader`.
3. Linux users need to change the mode with `chmod u+x onedrive-uploader` to grand execution permissions.
4. Run `onedrive-uploader config` from the command line.
   1. Enter your application client ID.
   2. Enter your application secret.
   3. Select the default scope if you want access to the entire drive, or App Root if you want to access the application only.
   4. Select the default drive root if you want access to the entire drive, or App Root if you want to access the application only.
   5. Use the redirect URL. Hit enter to use the default value.
   6. Save your configuration. Enter a location or hit enter to use the default value. 
5. Run `onedrive-uploader login` from the command line. Open the returned link in your web browser.
6. Save the path to the generated `config.json` file. You need this later.

### Prepare CODESYS project

The OneDriveClient library for CODESYS needs onedrive-uploader to be available on the target system. You can manually install it on the target system or include onedrive-uploader in your CODESYS project.

1. Create a folder named `ExternalFiles` in your CODESYS project.
2. Add the correct [onedrive-uploader precompiled binary file](https://github.com/virtualzone/onedrive-uploader/releases) compatible with the target system running CODESYS Control as an embedded external file.
3. Add the generated `config.json` as an embedded external file.
4. Add the OneDriveClient library to your Library Repository and include it in your project using the Library Manager. Check out the Library Manager for additional documentation of the available POUs and DUTs.

## CODESYS examples

### Configure using embedded external files

```
VAR
	sAppPath : STRING(80) := '';
	stConfig : OneDriveClient.ONEDRIVE_CONFIG;
END_VAR

(* Set the OneDriveClient configuration parameters. *)
SysFile.SysFileGetPath('$$PlcLogic$$/Application/', sAppPath, 255);

stConfig.sExecutableFile := CONCAT(STR1 := sAppPath, STR2 := 'onedrive-uploader_linux_arm64_v0.8.0');
stConfig.sConfigFile := CONCAT(STR1 := sAppPath, STR2 := 'config.json');
stConfig.xIsWindows := FALSE;

(* On Linux systems, make sure onedrive-uploader has execute permissions. *)
OneDriveClient.ChangeMode(stConfig := stConfig);
```

### List remote directory

```
VAR
	stConfig		: OneDriveClient.ONEDRIVE_CONFIG;
	sRemoteDirectoryPath	: STRING(400) := '/';
	astItems		: ARRAY [1..1000] OF OneDriveClient.ONEDRIVE_ITEM_LIST;
	uiItemCount		: UINT := 0;
END_VAR

OneDriveClient.OneDriveList(
	sRemoteDirectoryPath := sRemoteDirectoryPath,
	astItems := astItems,
	stConfig := stConfig,
	uiItemCount => uiItemCount
);
```

### Create remote directory

```
VAR
	stConfig		: OneDriveClient.ONEDRIVE_CONFIG;
	sRemoteDirectoryPath	: STRING(400) := '/onedrive-test';
END_VAR

OneDriveClient.OneDriveMakeDir(
	sRemoteDirectoryPath := sRemoteDirectoryPath,
	stConfig := stConfig
);
```

### Upload local file to remote directory

```
VAR
	stConfig		: OneDriveClient.ONEDRIVE_CONFIG;
	sRemoteDirectoryPath	: STRING(400) := '/onedrive-test';
	sLocalFilePath		: STRING(255) := '/home/root/closetoyou.wav';
END_VAR

OneDriveClient.OneDriveUpload(
	sRemoteDirectoryPath := sRemoteDirectoryPath,
	sLocalFilePath := sLocalFilePath,
	stConfig := stConfig
);
```

### Get info of remote file or directory

```
VAR
	stConfig	: OneDriveClient.ONEDRIVE_CONFIG;
	sRemotePath	: STRING(400) := '/onedrive-test/closetoyou.wav';
	stInfo		: OneDriveCLient.ONEDRIVE_ITEM_INFO;
END_VAR

OneDriveClient.OneDriveInfo(
	sRemotePath := sRemotePath,
	stConfig := stConfig,
	stInfo := stInfo
);
```

### Download remote file to local directory

```
VAR
	stConfig		: OneDriveClient.ONEDRIVE_CONFIG;
	sRemoteFilePath		: STRING(400) := '/onedrive-test/closetoyou.wav';
	sLocalDirectoryPath	: STRING(255) := '/tmp';
END_VAR

OneDriveClient.OneDriveDownload(
	sRemoteFilePath := sRemoteFilePath,
	sLocalDirectoryPath := sLocalDirectoryPath,
	stConfig := stConfig
);
```

### Remove remote file or directory

```
VAR
	stConfig	: OneDriveClient.ONEDRIVE_CONFIG;
	sRemotePath	: STRING(400) := '/onedrive-test/';
END_VAR

OneDriveClient.OneDriveRemove(
	sRemotePath := sRemotePath,
	stConfig := stConfig
);
```

## UTF-8 encoding and WSTRING

CODESYS does not have proper support for UTF-8 encoded characters in STRING types prior to CODESYS 3.5 SP18. If available, enabling this option should support UTF-8 encoded paths with STRING (untested). Otherwise use WSTRING and interact with the library through the helper functions `NameToWstring` and `PathFromWstring`.

```
VAR
	stConfig		: OneDriveClient.ONEDRIVE_CONFIG;
	wsCreateRemotePath	: WSTRING(400) := "/onedrive-test/unicode/úniçôdè";
	sCreateRemotePath	: STRING(400) := '';
	sListRemotePath		: STRING(400) := '/onedrive-test/unicode';
	astItems		: ARRAY [1..1000] OF OneDriveClient.ONEDRIVE_ITEM_LIST;
	wsDirectoryName		: WSTRING(255) := "";
END_VAR

(* Convert WSTRING to STRING. *)
OneDriveClient.PathFromWstring(wsPath := wsCreateRemotePath, sPath := sCreateRemotePath);

(* Create remote directory. *)
OneDriveClient.OneDriveMakeDir(
	sRemoteDirectoryPath := sCreateRemotePath,
	stConfig := stConfig
);

(* List remote directory. *)
OneDriveClient.OneDriveList(
	sRemoteDirectoryPath := sListRemotePath,
	stConfig := stConfig,
	astItems := astItems
);

(* Get name of only item in remote directory. *)
OneDriveClient.NameToWstring(stItem := astItems[1], wsItemName := wsDirectoryName);
```

## Credits

- ☕[Imotik](https://www.imotik.nl/en/)
- ⭐[virtualzone](https://github.com/virtualzone)/[onedrive-uploader](https://github.com/virtualzone/onedrive-uploader)
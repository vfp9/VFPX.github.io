1. Create a subdirectory of your project folder named ThorUpdater.

2. Copy CreateThorUpdate.ps1, Project.txt, and Version.txt into ThorUpdater.

3. Edit Project.txt to specify your project information:

    - appName is the display name for the project.
    
    - appID is similar to appName but must be URL-friendly (no spaces or other illegal URL characters).

    - majorVersion is the major version number, such as 1.0.

    - excludeFiles is a comma-delimited list of file names to exclude from the zip file. Wildcards are allowed, so use *.bak to exclude BAK files. Don't put any spaces before or after commas or it won't work.

    - excludeFolders is a comma-delimited list of folders to exclude from the zip file. Put an * before and after each folder name since they're treated as wildcards (for example, *Docs* to exclude any folder containing "Docs" in the name). Don't put any spaces before or after commas or it won't work.

    Note: CreateThorUpdate.ps1 doesn't look for "appName =", "appID =", etc.; it just picks up the content after the "=" and expects that line 1 contains the display name, line 2 contains the application ID, and so on.

4. CreateThorUpdate.ps1 generates a Thor version file named <appID>Version.txt in the ThorUpdater folder, where <appID> is the appID value specified in Project.txt. CreateThorUpdate.ps1 uses Version.txt as a template when generating this file. Thor version files contain code that specify the version number for the current release. Version.txt uses MajorVersion.JulianDate for the version number of a project, where MajorVersion is the majorVersion value specified in Project.txt and JulianDate is the current day number starting from January 1, 2001. For example, for a new version released on October 12, 2017, the JulianDate value is 6494. The nice thing about using JulianDate is that you can work backwards from the value to get the release date if necessary. Version.txt is generic; it uses APPNAME as a placeholder for the project name and MAJORVERSION as a placeholder for the major version number in the AvailableVersion property of the update object that Thor passes into the code. For example, if Project.txt contains this:

    ```
    appName = Project Explorer  
    appID = ProjectExplorer  
    majorVersion = 1.0
    ```

    then if run on October 12, 2017, CreateThorUpdate.ps1 generates ProjectExplorerVersion.txt in the ThorUpdater folder with this content:

    ```
    lparameters toUpdateObject
    local ldDate, ;
    	lnJulian, ;
    	lcJulian
    ldDate   = date(2017,10,12)
    lnJulian = val(sys(11, ldDate)) - val(sys(11, {^2000-01-01}))
    lcJulian = padl(transform(lnJulian), 4, '0')
    toUpdateObject.AvailableVersion = 'Project Explorer-1.0.' + lcJulian + ;
    	'-update-' + dtoc(ldDate, 1)
    return toUpdateObject
    ```

    The format of AvailableVersion in Version.txt -- the project name, a dash, the version number, a dash, some text (Thor doesn't appear to use it), a dash, and the release date formatted as YYYYMMDD -- is required by Thor.

5. CreateThorUpdate.ps1 creates a zip file named <appID>.zip in the ThorUpdater folder, where <appID> is the appID value specified in Project.txt. This zip file contains all files in all subdirectories of the project folder with the exception of any files specified in the excludeFiles setting and any folders specified in the excludeFolders setting of Project.txt. You'll likely want the following in Project.txt as a minimum:

    ```
    excludeFiles = *.bak,*.fxp
    excludeFolders = *ThorUpdater*
    ```
    
6. After editing Project.txt, you're ready to create the files Thor needs to download and install or update your project. Right-click CreateThorUpdate.ps1 in the ThorUpdater folder and choose Run with PowerShell from the shortcut menu. That creates <appID>Version.txt and <appID>.zip. Add these two files to your repository (if you haven't done so previously), commit, and push to the remote repository. Every time you release a new version, repeat this step.

7. In order for Thor to know about your project, you have to create a Thor updater program named Thor_Update_<appID>.prg. Here is a template you can use for such a program; see the comments about what to edit to make it specific for your project:

    ```
    lparameters toUpdateObject
    local lcAppName, ;
    	lcAppID, ;
    	lcRepositoryURL, ;
    	lcDownloadsURL, ;
    	lcVersionFileURL, ;
    	lcZIPFileURL, ;
    	lcRegisterWithThor
    
    * Set the project settings; edit these for your specific project.
    
    lcAppName       = 'Project Explorer'
    	&& the name of the project
    lcAppID         = 'ProjectExplorer'
    	&& similar to lcAppName but must be URL-friendly (no spaces or other
    	&& illegal URL characters)
    lcRepositoryURL = 'https://github.com/DougHennig/ProjectExplorer'
    	&& the URL for the project's repository
    
    * If the version file and zip file are in the ThorUpdater folder of the
    * master branch of a GitHub repository, these don't need to be edited.
    * Otherwise, set lcVersionFileURL and lcZIPFileURL to the correct URLs.
    
    lcDownloadsURL   = strtran(lcRepositoryURL, 'github.com', ;
    	'raw.githubusercontent.com') + '/master/ThorUpdater/'
    lcVersionFileURL = lcDownloadsURL + lcAppID + 'Version.txt'
    	&& the URL for the file containing code to get the available version number
    lcZIPFileURL     = lcDownloadsURL + lcAppID + '.zip'
    	&& the URL for the zip file containing the project
    
    * This is code to execute after the project has been installed by Thor for the
    * first time. Edit this if you want do something different (such as running
    * the installed code) or display a different message. You can use code like
    * this if you want to execute the installed code; Thor replaces
    * ##InstallFolder## with the installation path for the project:
    *
    * 'do "##InstallFolder##MyProject.prg"'
    
    text to lcRegisterWithThor noshow textmerge
    	messagebox('From the Thor menu, choose "More -> Open Folder -> Components" to see the folder where <<lcAppName>> was installed', 0, '<<lcAppName>> Installed', 5000)
    endtext
    
    * Set the properties of the passed updater object. You likely won't need to edit this code.
    
    with toUpdateObject
    	.ApplicationName      = lcAppName
    	.Component            = 'Yes'
    	.VersionLocalFilename = lcAppID + 'VersionFile.txt'
    	.RegisterWithThor     = lcRegisterWithThor
    	.VersionFileURL       = lcVersionFileURL
    	.SourceFileUrl        = lcZIPFileURL
    	.Link                 = lcRepositoryURL
    	.LinkPrompt           = lcAppName + ' Home Page'
    	.Notes                = GetNotes()
    endwith
    return toUpdateObject
    
    * Get the notes for the project. Edit this code as necessary.
    
    procedure GetNotes
    local lcNotes
    text to lcNotes noshow
    Project Explorer
    
    Project Manager: Doug Hennig
    
    Project Explorer is a VFPX project that replaces the Project Manager with a modern interface and modern capabilities. It has most of the features of the Project Manager but adds integration with DVCS (including built-in support for FoxBin2PRG and optional auto-commit after changes), support for multiple projects within a "solution," allows you to organize your items by keyword or other criteria, and has support for easy "auto-registering" addins that can customize the appearance and behavior of the tool.
    endtext
    return lcNotes
    ```

7. After creating this program, zip it and email it to the VFPX project managers (projects@vfpx.org). They'll add it to the Thor repository so Thor knows about your project. You only have to do this one time.

8. Now that Thor knows about your project, the next time a user chooses Check for Updates from the Thor menu in VFP, Thor will download <appID>Version.txt from your repository, see that the project is available to be installed or a new version is ready for download, and display it to the user in the update dialog. If the user chooses to install the project or the update, Thor downloads <appID>.zip from your repository and unzips it in the project folder (the Thor\Tools\Components\<appName> subdirectory of the main Thor folder) and creates <appID>VersionFile.txt in that folder so it know what version the user has so they aren't prompted about an update until the next release.

9. Every time you release a new version, right-click CreateThorUpdate.ps1 in the ThorUpdater folder and choose Run with PowerShell from the shortcut menu. That creates <appID>Version.txt and <appID>.zip. Commit and push to the remote repository.
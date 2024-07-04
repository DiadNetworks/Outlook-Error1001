# Outlook-Error1001
A guide to fixing: *Error “Something Went Wrong [1001]” signing in to Microsoft 365 Desktop Applications* (when caused by FSLogix profile mangement issues)

![image](https://github.com/DiadNetworks/Outlook-Error1001/assets/143122318/71381185-09e3-47df-9960-04122321106e)

## What we are fixing:
The error “Something Went Wrong [1001]” when signing into Microsoft 365 desktop applications. More specifically, here we are focusing on Outlook desktop.  

There are 2 known causes for this error:
1. Security software impacting the WAM plug-in (AAD.BrokerPlugin).
2. FSLogix user profile management issues.

If your issue is being caused by scenario 1, here are some resources to put you on the right path:  
https://support.microsoft.com/en-us/office/error-something-went-wrong-1001-signing-in-to-microsoft-365-desktop-applications-6f63238d-d83c-437c-a929-de72fe819793  
https://learn.microsoft.com/microsoft-365/troubleshoot/authentication/cannot-sign-in-microsoft-365-desktop-apps  

If your issue is being caused by scenario 2, continue reading.

## Understanding the problem:
Here is an article from Microsoft regarding best practices for user profile management:  
https://learn.microsoft.com/entra/identity/devices/howto-device-identity-virtual-desktop-infrastructure#non-persistent-vdi  

We'll be focusing on this part of that article, showing how to exclude the listed files and registry keys from being roamed:  
```
Roaming any data under the path %localappdata% is not supported. If you choose to move content under %localappdata%, make sure that the content of the following folders and registry keys never leaves the device under any condition. For example: Profile migration tools must skip the following folders and keys:

%localappdata%\Packages\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy 

%localappdata%\Packages\Microsoft.Windows.CloudExperienceHost_cw5n1h2txyewy 

%localappdata%\Packages\<any app package>\AC\TokenBroker 

%localappdata%\Microsoft\TokenBroker 

HKEY_CURRENT_USER\SOFTWARE\Microsoft\IdentityCRL 

HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\AAD 

HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WorkplaceJoin

Roaming of the work account's device certificate is not supported. The certificate, issued by "MS-Organization-Access", is stored in the Personal (MY) certificate store of the current user and on the local machine.
```

## Fixing the problem:
1. We need a `redirections.xml` file to tell FSLogix what to exclude from roaming. Download the `redirections.xml` file from the repository or copy this into your own `redirections.xml` file:
    ```
    <?xml version="1.0"  encoding="UTF-8"?>
    <FrxProfileFolderRedirection ExcludeCommonFolders="0">
    <Excludes>
    <Exclude>%localappdata%\Packages\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy</Exclude>
    <Exclude>%localappdata%\Packages\Microsoft.Windows.CloudExperienceHost_cw5n1h2txyewy</Exclude>
    <Exclude>%localappdata%\Packages</Exclude>
    <Exclude>%localappdata%\Microsoft\TokenBroker</Exclude>
    <Exclude>HKEY_CURRENT_USER\SOFTWARE\Microsoft\IdentityCRL</Exclude>
    <Exclude>HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\AAD</Exclude>
    <Exclude>HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WorkplaceJoin</Exclude>
    </Excludes>
    <Includes>
    </Includes>
    </FrxProfileFolderRedirection>
    ```  

2.  We now need to tell FSLogix to look at this `redirections.xml` file.  
    - Create a folder (name it what you like, `FSLogix-Redirections` seems clean) somewhere accessible from the servers that are using FSLogix. This can be a shared file location on your network. **Important:** Make sure all users have permission to access this folder.  
    - Now place the `redirections.xml` file in that folder.  
    - Now find and edit your group policy for the servers that are using FSLogix. This can be a group policy used for multiple servers in group policy management on a domain controller, or just the local group policy on a single server - depends on your setup.  
    - Navigate to the `Redirection XML Source Folder` policy located at `Computer Configuration > Policies > Administrative Templates > FSLogix > Profile Containers`, see image:  
    - Double click the policy to edit it. By default this policy is set to "Not configured", change that to "Enabled". For the "Redirection XML Source Folder" value, enter the path to the folder that your `redirections.xml` file is in. **Note:** The path to the folder that it's located in, not the path to the actual file.  
    - Apply the policy and you're done.  

3. Now we will confirm that the policy is working and that FSLogix is adding the exclusion rules upon user login.  
Choose a user from your domain to use for testing and continue.  
    - Make sure the user is completely logged out from the server (not just disconnected), and then login as the user.  
    - Open this file path in file explorer: `C:\ProgramData\FSLogix\Logs\Profile`
    - Open the latest log in this folder (date modified should be the same as when you logged in with the user).
    - Scroll to the bottom of the file and search (Ctrl+F usually) for: `Configuration Read (REG_SZ): SOFTWARE\FSLogix\Profiles\RedirXMLSourceFolder`
    - If the `redirections.xml` was successful, you should see logs similar to this:
        ```
        [23:36:31.364][tid:00000f10.00003bdc][INFO]             Configuration Read (REG_SZ): SOFTWARE\FSLogix\Profiles\RedirXMLSourceFolder.  Data: \\<stg-acct>.file.core.windows.net\containers
        [23:36:31.364][tid:00000f10.00003bdc][INFO]             Attempting to copy: "\\<stg-acct>.file.core.windows.net\containers\Redirections.xml" to: "C:\Users\%username%\AppData\Local\FSLogix\Redirections.xml"
        [23:36:31.396][tid:00000f10.00003bdc][INFO]             Redirections.xml copy success
        [23:36:31.396][tid:00000f10.00003bdc][INFO]             Reading profile folder redirections
        [23:36:31.411][tid:00000f10.00003bdc][INFO]             Creating base folders for profile folder redirections
        [23:36:31.411][tid:00000f10.00003bdc][INFO]             Creating base folder 'AppData\Roaming\Microsoft\Teams\media-stack\'
        [23:36:31.427][tid:00000f10.00003bdc][INFO]             Creating base folder 'AppData\Local\Microsoft\Teams\meeting-addin\Cache\'
        [23:36:31.427][tid:00000f10.00003bdc][INFO]             Creating base folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\Logs\'
        [23:36:31.427][tid:00000f10.00003bdc][INFO]             Creating base folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\PerfLogs'
        [23:36:31.427][tid:00000f10.00003bdc][INFO]             Creating base folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\WV2Profile_tfw\WebStorage'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Adding exclude rule for folder 'AppData\Roaming\Microsoft\Teams\media-stack\'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Added redirection C:\Users\%username%\AppData\Roaming\Microsoft\Teams\media-stack -> C:\Users\local_%username%\AppData\Roaming\Microsoft\Teams\media-stack
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Adding exclude rule for folder 'AppData\Local\Microsoft\Teams\meeting-addin\Cache\'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Added redirection C:\Users\%username%\AppData\Local\Microsoft\Teams\meeting-addin\Cache -> C:\Users\local_%username%\AppData\Local\Microsoft\Teams\meeting-addin\Cache
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Adding exclude rule for folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\Logs\'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Added redirection C:\Users\%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\Logs -> C:\Users\local_%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\Logs
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Adding exclude rule for folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\PerfLogs\'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Added redirection C:\Users\%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\PerfLogs -> C:\Users\local_%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\PerfLogs
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Adding exclude rule for folder 'AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\WV2Profile_tfw\WebStorage\'
        [23:36:32.099][tid:00000f10.00003bdc][INFO]             Added redirection C:\Users\%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\WV2Profile_tfw\WebStorage -> C:\Users\local_%username%\AppData\Local\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams\EBWebView\WV2Profile_tfw\WebStorage
        ```

4. If the logs check out, the issue is now fixed. If the log has issues, double check these two important notes:
    - The value in the group policy should be the path to the parent folder of `redirections.xml`, not the path to the actual `redirections.xml` file.  
    - All users need to have at minimum "read" access to the parent folder of `redirections.xml` and at minimum "read" access to the `redirections.xml` file.

## What to do now?
The credentials for Outlook need to be cleared before you can sign in again.  

An easy way to do this is to open Excel (or Word), go to the account settings and sign out.  

Now, when you sign into Outlook it should ask for an email address and password (whereas previously it would maybe have the password or email saved).
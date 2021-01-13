---
layout: single
title: Installing uTorrent on WHS “Vail” as Service
tags: [utorrent, vail, whs, windows home server]
category: [blog]
comments: false
#OCTOBER 10, 2010 BY PRAVESH SONI

---

![Works on my machine](/assets/images/worksonmymachine.jpg "Works on my machine"){: .align-right}  First of all, let me say that the credit for this post will goes to [Philip Churchill](https://mswhs.com/). He initially posted the original article on how to [install uTorrent on WHS as windows service](http://www.mswhs.com/2007/07/how-to-install-utorrent-on-windows-home-server/). The idea behind to install [uTorrent](https://www.utorrent.com/) on the [WHS](http://www.microsoft.com/windows/products/winfamily/windowshomeserver/default.mspx) is to have unattended torrent download without having user to log in to the server/computer. This guide is working perfectly for WHS V1 until WHS V2 code name “Vail”. Vail is based on latest Windows Server 2008 R2 and only available in 64bit edition. There are lot of security enhancements in the base OS prevents uTorrent to work properly as per the steps mentioned in the original article. Due to the security enhancements in Windows Server 2008 R2, we can not use **Local Service** OR **Network Service** account for uTorrent Service. Here is a workaround for making uTorrent working under WHS “Vail”.


### Create User


First of all, we need to create a user account for uTorrent service account using server dashboard.

Open server dashboard and navigate to **Users** tab. Invoke **Add a User Account** wizard by clicking **Add a user account** link in **Users Tasks** pane.

![WHS Vail - Dashboard](/assets/images/DashboardView.png)



Once the wizard opens, fill the appropriate information in the first step and proceed to next by clicking on **Next** button.

![WHS Vail - Create user wizard 1](/assets/images/CreateUser1.png)



In second step, don’t assign any permissions to any shared folder to the user account.


![WHS Vail - Create user wizard 2](/assets/images/CreateUser2.png)



Also do not allow remote access and finish the wizard by clicking **Create account** button.



![WHS Vail - Create user wizard 3](/assets/images/CreateUser3.png)


### Create Shared folder

in next step, we need to create a shared folder for uTorrent download data. To do so, invoke **Add a Folder** wizard using the task pane in **Server Folders**.

![WHS Vail – Create shared folder wizard 1](/assets/images/CreateFolder1.png)


In first step of wizard, give share name and description and proceed to next step by clicking **Next** button.

![WHS Vail – Create shared folder wizard 2](/assets/images/CreateFolder2.png)


In second step, click on **Specific people** to assign permissions to our uTorrent service account.

![WHS Vail – Create shared folder wizard 3](/assets/images/CreateFolder3.png)


Assign permission as per above screen shot and finish the wizard by clicking **Add folder** button to complete the folder creation process.

### Install uTorrent

[Download uTorrent](https://www.utorrent.com/downloads/) and start installation wizard accept all default options but but don’t install shortcuts to the desktop, start menu, and quick launch bar as they will not be necessary. After completion of installation, uTorrent should launch automatically. Configure uTorrent as per the settings which we used for WHS V1. Don’t forget to enable WebUI in uTorrent options.

### Create Windows Service

To create windows service, login to Vail console using Remote Desktop Connection. Copy srvany.exe from Windows Server2003 Resource Kit to uTorrent install folder. Now open command prompt and run sc.exe with following parameters to create service.

```
sc create uTorrent binPath= “C:Program Files (x86)uTorrentsrvany.exe” displayName= “uTorrent”
```

![WHS Vail – Create windows service](/assets/images/CreateService.png)

**NOTE:** Please note that there is a space after equal sign.
{: .notice--info}

If all goes well then you will get **[SC] CreateService SUCCESS** feedback in command prompt.

Now we need to create a .Reg file using notepad and paste in the 3 lines of code below.

```
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\uTorrent\Parameters]
“Application”=”C:\\Program Files (x86)\\utorrent\\utorrent.exe”
```

Save file as service.reg and merge this in to registry by double clicking the newly created .reg file.

### Configure uTorrent Service

Now click the Start button and open Services console from **Administrative Tools > Services**. Find uTorrent right-click and select Properties.

Select the Log On tab. Click the This account button and enter WHS as the This account and enter the Password you setup earlier for this user account and confirm the Password.

![uTorrent - Service account properties](/assets/images/ServiceAccountProperties.png)


OK out and close the Services dialog.

For user profile to be created, we need to start the service and stop so we get the profile folder for the user we created earlier. Next, we need to copy the **uTorrent** settings from the Administrator profile to the uTorrent User profile. Go to **C:\\Users\\Administrator\\AppData**. Copy the **uTorrent** directory to **C:\\Users\\\<Torrent_User\>\\AppData**. When the service starts, it will have all settings ready and waiting to download.


Now its time for downloading torrent content using web GUI. You can access the web interface by navigating http://\<your_server:port\>/gui/.

For further details and know about how to install uTorrent on WHS V1 refer Scott Duff – Autumn Walker, Philip Churchill – mswhs.com, Drashna – wegotserved.com.

Happy downloading from Vail…

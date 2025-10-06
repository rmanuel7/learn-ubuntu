# Basic operations on Linux

> [!NOTE]
> Run commands by using the `sudo` prefix

### Package managers
Package managers are used to install, upgrade, and remove applications in Linux. 

> [!NOTE]
> APT works on a database of available packages. We recommend that you update the package managers, and then upgrade the packages after a fresh installation.

- To update the package database on Ubuntu, run `sudo apt update`. Notice that the `sudo` prefix is entered before the `apt` command.
> [!IMPORTANT]
> The update command **doesn't actually upgrade** any of the installed software packages. Instead, it updates the package database. The actual upgrade is done by the `sudo apt upgrade` command.

### Install the package
- `sudo apt install apache2`
- `sudo apt install wget`
- `sudo apt install net-tools` -> netstat

<br/><br/>
**References**
1. [Microsoft: Basic operations on Linux](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnetcore/practice-troubleshoot-linux/1-2-linux-special-directories-users-package-managers)

<br/>

---

# Install .NET Core

- The first command is a `wget` command. `wget` is a non-interactive network downloader.
  ```bash
  wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  ```

- In the second command, `dpkg` is the package manager for Debian and Ubuntu. This command adds the Microsoft package signing key to the list of trusted keys, and then adds the package repository.
  ```bash
  sudo dpkg -i packages-microsoft-prod.deb
  ```

- The **ASP.NET Core Runtime** allows you to run apps that were made with .NET that didn't provide the runtime.
  ```bash
  sudo apt-get update && \
    sudo apt-get install -y aspnetcore-runtime-8.0
  ```
> [!TIP]
> .NET Core packages are named in the format of {**product**}-{**type**}-{**version**}

### Create service file for your ASP.NET Core application
- Create the service definition file:
  ```bash
  sudo nano /etc/systemd/system/kestrel-helloapp.service
  ```

- The following example is an .ini service file for the app:
  ```txt
  [Unit]
  Description=Example .NET Web API App running on Linux
  
  [Service]
  WorkingDirectory=/var/www/helloapp
  ExecStart=/usr/bin/dotnet /var/www/helloapp/helloapp.dll
  Restart=always
  # Restart service after 10 seconds if the dotnet service crashes:
  RestartSec=10
  KillSignal=SIGINT
  SyslogIdentifier=dotnet-example
  User=www-data
  Environment=ASPNETCORE_ENVIRONMENT=Production
  Environment=DOTNET_NOLOGO=true
  
  [Install]
  WantedBy=multi-user.target
  ```

- Save the file and enable the service. Start the service and verify that it's running.
  ```bash
  sudo systemctl enable kestrel-helloapp.service
  sudo systemctl start kestrel-helloapp.service
  sudo systemctl status kestrel-helloapp.service
  ```

### Test the web site from another terminal.

- Run `netstat` together with the `-tlp` switch
  ```bash
  netstat -tlp
  ```

<br/><br/>
**References**
1. [Microsoft: Install .NET Core in Linux](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnetcore/practice-troubleshoot-linux/1-3-install-dotnet-core-linux)
2. [Microsoft: Install .NET SDK or .NET Runtime on Ubuntu](https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu-install?tabs=dotnet8&pivots=os-linux-ubuntu-2204)
3. [Microsoft: Create and configure ASP.NET Core applications in Linux](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnetcore/practice-troubleshoot-linux/2-3-configure-aspnet-core-application-start-automatically#create-service-file-for-your-aspnet-core-application)
4. [Microsoft: Monitor the app](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-8.0&tabs=linux-ubuntu#monitor-the-app)

<br/>

---

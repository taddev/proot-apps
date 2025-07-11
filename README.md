# PRoot Apps

PRoot Apps is a simple platform to install and use applications without any privileges in Linux userspace using [PRoot](https://proot-me.github.io/). Applications are bundled with their entire toolchain much like [snap](https://snapcraft.io/) or [Flatpak](https://flatpak.org/), unlike these platforms:

* No system requirements outside of curl and bash
* Can be run in very restrictive environments even inside userspace in a Docker container
* Performs no space saving optimizations, every PRoot app lives in its own individual file folder without a base or any overlay style filesystem
* Are ingested from [Docker](https://www.docker.com/) registries and can be published by anyone while being consumed by anyone

# For Users

## Install or update

```
rm -f $HOME/.local/bin/{ncat,proot-apps,proot,jq}
mkdir -p $HOME/.local/bin
curl -L https://github.com/taddev/proot-apps/releases/download/$(curl -sX GET "https://api.github.com/repos/taddev/proot-apps/releases/latest" | awk '/tag_name/{print $4;exit}' FS='[""]')/proot-apps-$(uname -m).tar.gz | tar -xzf - -C $HOME/.local/bin/
export PATH="$HOME/.local/bin:$PATH"
```

**Optional add path to env**

```
echo 'export PATH="$HOME/.local/bin:$PATH"' >> $HOME/.bashrc
```

**PRoot Apps is currently only supported on amd64 and arm64 systems**

## Uninstall

```
proot-apps remove all
rm -f $HOME/.local/bin/{ncat,proot-apps,proot,jq}
rm -Rf $HOME/proot-apps/
```

## Hello World

### Graphical Installer

Supported applications can be installed via a graphical installer by adding it:

```
proot-apps install gui
```

### Command Line

To install your first app [Firefox](https://www.mozilla.org/firefox/) simply execute:

```
proot-apps install firefox
```

The files for Firefox will be installed to a folder in `$HOME/proot-apps/`, Desktop and start menu shortcuts will also be created these names should not conflict with system installed packages.

These short named apps are available from the supported list below, but any app can be consumed from a Docker endpoint IE:

```
proot-apps install ghcr.io/taddev/proot-apps:firefox
```

To remove the application:

```
proot-apps remove firefox
```

To update the application:

```
proot-apps update firefox
```

## Supported Apps

<details>
  <summary>Click to expand</summary>

| Name | Full Endpoint | Arch | Description |
| :----: | :----: | :----: |--- |
| atlauncher | ghcr.io/taddev/proot-apps:atlauncher | linux/amd64 | ATLauncher is a launcher for Minecraft which integrates multiple different modpacks to allow you to download and install modpacks easily and quickly.|


</details>

# For Developers

This repository was made to be cloned and forked with all of the build logic being templated to the repository owner and name. Simply forking this repository and enabling GitHub actions should be enough to get building using GitHub's builders. Also included in this repository is a nightly package check action, in order to use that you will need to set a [GitHub Secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) for `PAT` for that bot to work as it needs basic write access to the repo to trigger downstream build actions. The build logic is currently configured to detect changes to the specific folders in `apps` to determine if they need to be built. To build a backlog of the images in this repo when forked removing the package_versions.txt from the apps you want to build and commiting that is likely the easiest approach. `find . -name package_versions.txt -exec git rm {} \;`

When forking all readme updates should be made to the `ci-scripts/README.template` as this is the source file used to write out the readme linked to your forked repo. Also keep in mind all the logic in your forked proot-apps binary and gui installer will be linked to your repostiry and packages.

To generate release tarballs of proot-apps to be ingested with your install command a tag needs to be generated in your fork. This can be achieved with `git tag 0.1.0 && git push origin --tags`. This again will be linked back to your repo in all references for app ingestion.

## Building local apps

Prerequisites:

* [Docker](https://www.docker.com/) installed
* PRoot apps installed as your user

In this repository is a helper script `build-install-local` that can be used to generate a PRoot app from the build logic in this repo. To build and extract the firefox image simply: 

```
./build-install-local firefox
```

Then follow the instructions to install and test it out.

## Basic application structure

We are using Docker to do the heavy lifting here, leveraging it for building and it's registry system for hosting the applications. In the end what makes up a PRoot app is the entire operating system needed for the application to run. There are 4 files required for an application:

* Dockerfile - This contains all of the build logic for the application, installation and customization for the application should be performed here including:
  - Desktop file launching proot-apps instead of the default command from OS install
  - Setting a bin name for the application so it can be launched from the users CLI
  - Modifying the Name of the application in it's desktop entry as to not conflict with the users OS level applications
* entrypoint - This file is executed every time the application is launched it should contain any logic for the application to run properly, this might include custom flags to drop sandboxing requirements as they will not work in PRoot
* install - This file will be called when the user installs the application with proot-apps it should:
  - Add a desktop shortcut if applicable
  - Add a start menu entry if applicable
  - Add an icon for the application if applicable
* remove - This file will be called when the user uninstalls the application with proot-apps it should:
  - Remove a desktop shortcut if applicable
  - Remove a start menu entry if applicable
  - Remove an icon for the application if applicable

It is important to understand that nothing about this platform is security focused PRoot in general is running in userspace and can easily be escaped by the application by simply killing its Tracer PID. The point of this platform is not application isolation it is easing the burden of installation to not require any form of sudo access or unconventional platform binaries. To that end on init PRoot Apps will also start a system socket for sending commands to the underlying host to escape it's jail. This leverages a UNIX socket with ncat to relay commands to the host. This is required to open things like file explorers on the host that would otherwise not be available. PRoot already mounts in the system dbus socket and many applications will leverage this to call external programs in Desktop userspace, for applications that do not use this they will conventionally pass URLs and file paths to `xdg-open`. The `xdg-open` binary in the PRoot jail can be replaced with a simple wrapper: 

```
#!/bin/bash
echo "xdg-open "$@"" | nc -N -U /system.socket
```

This same wrapper concept can be used to wrap any other binary inside the jail that you would want to open up on the host and not inside of the PRoot runspace itself.

One other concept to keep in mind is if you want it to work in the application you will need to install it in the build logic. The only storage space PRoot really has access to at runtime is the user's home directory. If the program needs GPU acceleration, you will need to install the drivers (amdgpu, dri gallium, etc). If the program needs language packs or fonts again you will need to install them. In general this approach is incredibly wasteful when it comes to disk space but without using more complex systems like OverlayFS or bubblewrap (user namespacing) it is necessary.

## Adding a new app

Three are three things needed for an app to work with all build and ingestion logic: 

* A folder with a `Dockerfile` must be created in the `/apps/` directory named after the app name (this is what the user types to install the application)
* Metadata for the application must be added to `/metadata/metadata.yml` file including:
  - name - the app name also tied to the folder used previously, should be a single string all lowercase
  - full_name - the display name for the application
  - arch - what arches to build the application for
  - icon - the icon name for the application
  - description - a blurb explaining the application to the user
  - (optional) disabled - this will only build and push the tag for the application, it will not be advertised in the readme or desktop application
* A logo for the application placed in the `/metadata/img` folder of the repository, svg is preffered here, but can also use 192x192 pngs

When adding new applications we highly encourage copying an existing application folder as a start to understand what files are needed. Most apps can simply be installed from a distribution's repository and then it is just housekeeping to ensure the desktop file and icon conform to PRoot apps standards.

# For Administrators

PRoot Apps can use a local folder for app management and updates. This essentially replaces the remote repository with a folder of tar files. This is setup by using the environment variable `PA_REPO_FOLDER`, when set the user will pull their apps from a local folder of tar files at that path instead of a remote repo including updates.

## Setup Local Repo

In this example we will use the path `/mnt/apps` to act as our local repository. First create the directory and set your env:

```
mkdir /mnt/apps
export PA_REPO_FOLDER=/mnt/apps
```

We can use the `localrepo` action to perform get, update, or remove. Update and remove support passing `all` as a string to perform the action on all locally stored apps.

Get some apps: 

```
proot-apps localrepo get firefox chrome
```

Inside this folder will be:

```
└── /mnt/apps/
    ├── ghcr.io_YOURNAMESPACE_proot-apps_chrome/
    │   ├── app.tar.gz
    │   └── SHALAYER
    └── ghcr.io_YOURNAMESPACE_proot-apps_firefox/
        ├── app.tar.gz
        └── SHALAYER
```

`localrepo` can be used to update this repo as well:

```
proot-apps localrepo update all
```

This will sync down any updates from remote. 

To have users leverage the gui app in your namespace to install and remove applications you will also need to place the metadata from your repository (metadata folder) in this directory IE: 

```
└── /mnt/apps/
    └── metadata/
        ├── metadata.yml
        └── img/
            ├── logo1.svg
            └── logo2.svg
```

The metadata can be customized to what you want to present to the user in the gui application.

## Userspace ingesting the repo

The most likely scenario would be mounting your repo into the users session read only at a specific mount point like `/mnt/apps`.

To achieve this if it is a Docker container just mount in with a bind and set the env: 

```
-e PA_REPO_FOLDER=/mnt/apps
-v /mnt/apps:/mnt/apps:ro
```

When the user uses proot-apps in this session it will all be connected into this folders contents instead of a remote repository.

On the administration side the apps can be updated in the folder and the users with this mount will be able to ingest them with the normal command:

```
proot-apps update all
```

### try

The `try` command creates a new, temporary container which is removed once it is closed.

```
Usage: turbo try <options> <image> [<parameters>...]

<options> available:
  -a, --attach               Attach to stdin, stdout, and stderr of the container
      --admin                Run the container as admin user
  -d, --detach               Run the container in the background
      --disable=VALUE        Disable the specified Spoon VM setting
  -e, --env=VALUE            Set environment variables inside the container
      --enable=VALUE         Enable the specified Spoon VM setting
      --enable-log-stream    Enable web streaming of logs
      --env-file=VALUE       Read in a line delimited file of ENV variables
      --format=VALUE         Use json format for output
      --hosts=VALUE          Add an en to the virtual /etc/hosts file (<redirect>:<name>)
      --link=VALUE           Add link to another container (<container>:<alias>)
      --mount VALUE          Mount a host folder into the container, format: [other-container:]<source>=<target>
  -n, --name=VALUE           Name of created container
      --network=VALUE        Run container in specified virtual network
      --no-stream            Force no streaming even when stream is available
      --pull                 Pulls base images from hub before running, if they exist
      --route-add=VALUE      Add a TCP or UDP mapping, format: [<hostPort>]:<containerPort>[/tcp|udp]
      --route-block=VALUE    Isolate all ports of specified protocol (TCP or UDP) by default
      --startup-file=VALUE   Override the default startup file
      --trigger=VALUE        Execute named group of startup files
      --using=VALUE          Use specified images as a temporary dependency
      --vm=VALUE             The Spoon VM version to run the container with
  -w, --working-dir=VALUE    Set the initial working directory inside the container
      --wait-after-error     Leave program open after error
      --wait-after-exit      Leave program open after it exits
      --with-root=VALUE      Set the containers root directory
```

Turbo `try` can be used to specify multiple images by separating each image with a comma. If the same file, registry entry, or environment variable exists in multiple images, then the one from whichever image was specified last will win the conflict and be used in the virtual environment. Virtual machine settings are taken from the last specified image. Due to this "layering" approach, it is a good practice to specify images with newer versions of applications or libraries after images with older versions.

```
# Create a container with the apache/apache image
> turbo try apache/apache

# Create a container with apache and mysql
> turbo try apache/apache,mysql/mysql

# Create a container with .NET 3 and 4
> turbo try microsoft/dotnet:4.0.3,microsoft/dotnet:3.5.1
```

Containers are started with the startup file specified in the last passed image. If a startup file is not set in the base image then `cmd.exe /k` is used.
	
```
# Default startup file is used to start container
> turbo try oracle/jdk

# Override the startup file to use the command prompt
> turbo try --startup-file=cmd.exe oracle/jdk
```

The initial working directory for the container can be set with the `workdir` instruction or the `-w` flag. The current directory will be used if `workdir` was not specified and no `--startup-file` parameter was provided when building the image. 

```
# By default, a container's working directory matches the host's working directory
C:\Users> turbo try git/git

(0x3842xd) C:\Users>

# This sets the working directory to the root of the C drive
C:\Users> turbo try -w="C:\" git/git

(0x3842xd) C:\> 

```

Spoon VM settings can be enabled or disabled with the `--enable` and `--disable` flags, respectively. For a list of Spoon VM settings, see **VM Settings** section of the documentation.

Please note that `turbo.exe` always runs outside of the container on the host even if executed from within the container.

#### Adding Environment Variables

Environment variables can be added to a container with the `-e` or `--env-file` flags. 

```
# Add environment variable 'foo' with value 'bar'
> turbo try -e=foo=bar <image>

# Specify multiple env vars with multiple flags
> turbo try -e=foo=bar -e=x=2 <image>
```

If your container requires several environment variables then we recommend creating an **env-file**. An **env-file** is a line-delimited text file that lists all the environment variables to add to the container. The example file below lists 3 environment variables: 

```
foo=bar
utensil=turbo
my-var=10
```

Environment variables are always expanded on the host system before they are added to the container. 

```
> echo %PATH%

C:\Windows\system32;C:\Windows;

> turbo try -e=%PATH%;C:\Users <image>

(2fedfja3) > echo %PATH%
C:\Windows\system32;C:\Windows;C:\Users	
```

#### Virtual Networks

By default, containers run in the host network, meaning that any services exposed by a container can be accessible to the outside world just as if the application was running natively on the host. However, it is possible to run containers in virtualized network environments by specifying a network name other than "host" with the `--network` flag. Running a container in virtualized network environment prevents it from exposing services to the outside world (unless `--route-add` flag is used appropriately) while allowing for easy communication between containers running in the same virtualized network environment. In a virtual network, containers can connect to each other using their names as specified with the `--name` flag if there was any or the first 8 characters of the container ID otherwise.

```
# Launch a new container in the host network context (the default)
> turbo try --network=host <image>

# Launch two containers in a "mynet" virtual network
> turbo try -d --network=mynet --name=web <image>
web

> turbo try -d --network=mynet myself/webbrowser http://web
dd73e48aec024a7b9e15d2cf6599394f

# The former will accessible by its name "web" within the network,
# and the latter by its short ID: "dd73e48a"
```

**Note:** When connecting, always use the container name and not the network name. After all, what should your application connect to if there were two separate containers exposing the same services on the same virtual network if you connected by network name instead of container name?

#### Port Mapping

All network operations (opening/closing ports, for example) are passed through to the local machine when running in the host network context. To remap container ports to other ports on the local machine, use the `--route-add` flag. This flag also works when running in a virtualized network environment (by specifying the `--network` flag). Specific protocols (tcp or udp) can be mapped by specifying a `/[protocol]` after the mapping. If no protocol is specified, tcp is assumed.

```
# Map container tcp port 8080 to local port 80
> turbo try --route-add=80:8080 <image>

# Map udp traffic on container port 8080 to local port 80
> turbo try --route-add=80:8080/udp <image>

# Map container tcp port 80 to random port on local machine
# The random port can be later queried using the netstat command
> turbo try --route-add=:80 <image>
```

The default policy of allowing containers to bind to any port on the local machine can be changed with the `--route-block` flag. It isolates all services bound to container ports on specified protocols (tcp or udp). They can only be opened using the `--route-add` flag.

```
# Isolate all tcp services of a container
> turbo try --route-block=tcp <image>

# Isolate all tcp and udp services, but allow container tcp port 3486
# be bound to port 80 on local machine
> turbo try --route-block=tcp,udp --route-add=80:3486 <image>
```

#### Adding Custom Name Resolution Entries

All containers use name resolution provided by the host operating system. You can add specific name resolution overrides using the `--hosts` flag. The syntax is similar to that of the `hosts` file of the operating system.

```
# Make name my-test-service resolve to whatever the name
# test-service-43 resolves
> turbo try --hosts=my-test-service:test-service-43 <image>

# Make name mysite.net resolve to IPv4 address 127.0.0.1 and
# name ipv6.mysite.net resolve to IPv6 address ::1
> turbo try --hosts=127.0.0.1:mysite.net --hosts=::1:ipv6.mysite.net <image>
```

#### Container-to-Container Links

If you decided to not expose any services running in a container to the public by specifying the `--route-block` flag and not `--route-add`, you may still want to be able to connect to the services in your container from another container on the same machine. Although this is best achieved by running the containers in the same virtual network using the `--network` flag, container linking can be used for this purpose as well.

When creating a container with the `turbo try` command, you can use the `--link` flag to link it to any existing containers and the new container will be able to connect to any services exposed by the linked containers. Such connection creates a parent-child relationship where the newly created container is the parent.

With each link, an alias name must be specified. Name resolution overrides are added to the parent container so it can refer to its children by these names. Note how with container links the name that a container will use to refer to another container is defined by the former (the parent) using a parameter, instead of by the name of the container as is the case with virtual networks (the `--network` flag).

Container links also work between containers running in different virtual networks.


#### Using Startup Triggers

Images can be created with SpoonScript that have multiple startup files. Collections of startup files can be linked together by a trigger name and executed together.

```
# in turbo.me file to create "test-trigger" image...
startup file ["c:\windows\system32\notepad.exe", "c:\windows\regedit.exe"]
startup file doc=[("c:\windows\system32\notepad.exe", "c:\doc\welcome.txt"), ("c:\windows\system32\notepad.exe", "c:\doc\howto.txt")]


# from command-prompt...

# launch both notepad and regedit are launched
> turbo try test-trigger

# launch welcome.txt and howto.txt in notepad
> turbo try test-trigger --trigger=doc
```

#### Using Mount
The `mount` option provides a way to mount folders from the host into the container, giving access to resources from the host system.
If the source folder doesn't exist, the `mount` option is ignored. If the target folder doesn't exist, it is created.

Example for mounting a folder.

```
turbo try --mount "C:\FolderOnHostSystem=C:\FolderInContainer" clean
```

Mounts are useful to share a cache folder, like a local Maven repository:
```
turbo try --mount "%USERPROFILE%\.m2=%USERPROFILE%\.m2" jdk,maven
```

Mounting multiple folder is done by repeating the mount parameter:

```
turbo try --mount "C:\Mount1=C:\InContainer1" --mount "C:\Mount2=C:\InContainer2" clean
```

It is also possible to mount a folder from another container:

```
turbo try --mount <containerid>:"C:\FolderInSourceContainer=C:\FolderInTargetContainer" clean
```

#### Exit code

When the container stops, the exit code of startup file is displayed in decimal form.

#### Selecting VM version

A specific VM version can be selected by using the `--vm=version` flag. If the selected version is lower than the minimum version that is required by turbo.exe, then the minimum version will be used instead.

#### JSON output

When `--format=json` option was passed this command will provide output in JSON format. It will contain either a `container` array with information about created container or an `error` object if command failed.
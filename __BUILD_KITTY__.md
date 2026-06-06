
# MacOS

### build
```
% ./dev.sh build
  OR
% ./dev.sh build --full
```

### deploy

`./kitty/launcher/kitty.app` can be copied to Applications


# LinuxOS

### build

Inside the oraclelinux9-dev container

```
# boptionally
% ./dev.sh deps

% ./dev.sh linux-dist --full
```

### deploy
manually copy
```
jinhyu@ol9:/opt/kitty% sudo mv linux-package ___v0.47.0_linux-package
[sudo] password for jinhyu:
jinhyu@ol9:/opt/kitty% sudo mv dependencies ___v0.47.0_dependencies
jinhyu@ol9:/opt/kitty%
jinhyu@ol9:/opt/kitty%
jinhyu@ol9:/opt/kitty% sudo cp -r ~/github/jasonyukr/kitty/dependencies .
jinhyu@ol9:/opt/kitty% sudo cp -r ~/github/jasonyukr/kitty/linux-package .
jinhyu@ol9:/opt/kitty% ls -l
total 0
drwxr-xr-x. 3 root root 151 Jun  6 22:57 dependencies
drwxr-xr-x. 5 root root  41 Jun  6 22:57 linux-package
drwxr-xr-x. 3 root root 151 May 12 23:41 ___v0.46.2_dependencies
drwxr-xr-x. 5 root root  41 May 12 23:41 ___v0.46.2_linux-package
drwxr-xr-x. 3 root root 151 May 20 13:46 ___v0.47.0_dependencies
drwxr-xr-x. 5 root root  41 May 20 13:45 ___v0.47.0_linux-package
drwxr-xr-x. 3 root root 151 Apr 12 20:47 ____v45_dependencies
drwxr-xr-x. 5 root root  41 Apr 12 20:47 ___v45_linux-package
jinhyu@ol9:/opt/kitty%
```

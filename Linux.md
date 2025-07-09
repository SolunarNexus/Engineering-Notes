### Make AppImage an executable application (+sidebar assignment)
- source: https://davidmatthew.ie/how-to-launch-an-appimage-ubuntu/

- make it executable: 
```
chmod -x AppName-xxx.AppImage
```

- extract it: 
```
./AppName-xxx.AppImage --appimage-extract
```

- move the resulting directory squashfs-root:
```
sudo mv squashfs-root /opt/appname
```

- create a desktop app entry:
```
cd ~ && touch appname.desktop
```

- modify contents of the entry (example):
```
[Desktop Entry]
Name=Inkscape
Type=Application
Categories=Graphics;
MimeType=image/svg+xml;
Exec=/opt/inkscape/AppRun %F
Icon=/opt/inkscape/inkscape.svg
Terminal=false
StartupNotify=true
```

- install and add to application favorites: 
```
sudo desktop-file-install appname.desktop
```

- AppImageLauncher that supposedly does all the heavy lifting: https://github.com/TheAssassin/AppImageLauncher
### Load .profile into ZSH terminal
Add this command into `~/.zshrc` file
```shell
# Load common aliases, functions, env variables etc.
[[ -e ~/.profile ]] && emulate sh -c 'source ~/.profile'
```
### Launch custom scripts from anywhere
Add your directory containing scripts (e.g. `~/scripts`) to your `$PATH` environment variable. Then you can run your `script-you-want-to-run` from anywhere as a command (but not like `./script-you-want-to-run` because `./` denotes the current directory).

Read [[#Set `$PATH` environment variable for desktop session|this]] to understand how to change `$PATH` variable.
### Set `$PATH` environment variable for desktop session
In the file `~/.profile` add to the script:
```shell
if [ -d "$HOME/scripts" ] ; then
  PATH="$PATH:$HOME/scripts"
fi
```

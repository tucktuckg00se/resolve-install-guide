# DaVinci Resolve Installation Guide - Distrobox (Rocky Linux)

This guide covers installing DaVinci Resolve Studio inside a distrobox container using Rocky Linux. This is useful for running Resolve in an isolated environment while maintaining access to your system's GPU and display.

> **Note:** DaVinci Resolve does not support Wayland natively and requires X11 to run. If you're using Wayland as your display server, Resolve will run through XWayland.

> **Note:** **I have not tested this install guide with AMD or Intel GPUs**

---

## Prerequisites

- Distrobox installed on your system
- GPU with proper drivers installed on your host system:
  - **NVIDIA:** Proprietary NVIDIA drivers
  - **AMD:** Mesa drivers (usually pre-installed)
- DaVinci Resolve Studio installer (`.run` file) downloaded

---

## Initial Setup

### Create and Enter Distrobox Container

**For NVIDIA GPUs:**

```bash
distrobox create --nvidia resolve -i rockylinux/rockylinux
```

**For AMD GPUs:**

```bash
distrobox create resolve -i rockylinux/rockylinux
```

**What this does:**
- Creates a new container named "resolve"
- `--nvidia` (NVIDIA only) passes through GPU access to the container
- AMD GPUs automatically get `/dev/dri` access, so no special flags are needed

Enter the container:

```bash
distrobox enter resolve
```

> You're now inside the container. All following commands will be run inside this container unless otherwise noted.

---

### Update System and Enable Repositories

Update all installed packages:

```bash
sudo dnf update
```

Install EPEL repository and plugin support:

```bash
sudo dnf install epel-release dnf-plugins-core
```

Enable the CRB (CodeReady Builder) repository:

```bash
sudo crb enable
```

> **What this does:** CRB contains additional development libraries and dependencies needed by DaVinci Resolve.

---

### Install Required Dependencies

Install all the libraries and tools that DaVinci Resolve needs:

```bash
sudo dnf install ocl-icd libxcrypt pciutils libnsl fuse fuse-libs alsa-lib apr apr-util fontconfig freetype libglvnd libglvnd-egl libglvnd-glx libglvnd-opengl libgomp librsvg2 libXcursor libXfixes libXi libXinerama libxkbcommon libxkbcommon-x11 libXrandr libXrender libXtst libXxf86vm mesa-libGLU mtdev pulseaudio-libs xcb-util xcb-util-cursor xcb-util-image xcb-util-keysyms xcb-util-renderutil xcb-util-wm
```

**What these packages do:**
- OpenCL libraries for GPU acceleration
- Audio libraries for audio playback
- Graphics libraries for rendering
- X11 libraries for displaying the GUI
- Font and text rendering libraries

---

### Install X11 and Qt Dependencies

Install the Qt framework and X11 components:

```bash
sudo dnf install qt5-qtbase qt5-qtbase-gui libxcb libX11-xcb mesa-libGL mesa-libEGL
```

**What these packages do:**
- Qt framework that Resolve's interface is built with
- X11 communication libraries
- OpenGL graphics support

---

## Installation

### Prepare the Installer

Navigate to where you downloaded the DaVinci Resolve installer:

```bash
cd ~/Downloads
```

Make the installer executable (replace with your actual version):

```bash
chmod +x DaVinci_Resolve_Studio_20.3.1_Linux.run
```

---

### Run the Installer

```bash
sudo -E QT_QPA_PLATFORM=xcb ./DaVinci_Resolve_Studio_20.3.1_Linux.run -i
```

**What this does:**
- `sudo` runs the installer as root (needed to install to `/opt/resolve`)
- `-E` preserves your environment variables (like `DISPLAY`)
- `QT_QPA_PLATFORM=xcb` tells Qt to use X11 for the installer
- `-i` runs the installer in interactive mode

> **Note:** The installer may run in text mode - this is normal and works fine.

---

## Create a Desktop Launcher

### For Wayland Users

Create a wrapper script:

```bash
mkdir -p ~/.local/bin
```

```bash
cat > ~/.local/bin/resolve-wrapper << 'EOF'
#!/bin/bash
cd /opt/resolve/bin/
QT_QPA_PLATFORM=xcb ./resolve
EOF
```

```bash
chmod +x ~/.local/bin/resolve-wrapper
```

Create the desktop file:

```bash
mkdir -p ~/.local/share/applications
```

```bash
cat > ~/.local/share/applications/davinci-resolve.desktop << 'EOF'
[Desktop Entry]
Name=DaVinci Resolve
Exec=distrobox-enter -n resolve -- ~/.local/bin/resolve-wrapper
Icon=DV_Resolve
Type=Application
Categories=AudioVideo;VideoEditing;
Comment=Professional video editing, color correction, visual effects and audio post-production
Terminal=false
EOF
```

---

### For X11 Users

Create the desktop file:

```bash
mkdir -p ~/.local/share/applications
```

```bash
cat > ~/.local/share/applications/davinci-resolve.desktop << 'EOF'
[Desktop Entry]
Name=DaVinci Resolve
Exec=distrobox-enter -n resolve -- /opt/resolve/bin/resolve
Icon=DV_Resolve
Type=Application
Categories=AudioVideo;VideoEditing;
Comment=Professional video editing, color correction, visual effects and audio post-production
Terminal=false
EOF
```

> DaVinci Resolve should now appear in your application menu and can be launched like any other application.

---

## Importing Media Files

### From External Drives

External drives and USB devices are accessible at `/run/media/[your-username]/[drive-name]`.

**To import media:**

1. In DaVinci Resolve, use **File → Import → Import Media** (or `Ctrl + I`)
2. In the file browser sidebar, click **Computer**
3. Navigate to `/run/media/[your-username]/[drive-name]`
4. Select your files and import

> **Important:** Drag and drop does not work when running Resolve in distrobox. You must use the Import Media function.

---

### From Custom Mount Points

If you have drives mounted elsewhere (like `/mnt`), you need to recreate the container with those mounts.

Exit the container and remove it:

```bash
exit
```

```bash
distrobox rm resolve
```

Recreate with additional mounts:

```bash
distrobox create --nvidia resolve -i rockylinux/rockylinux --volume /mnt:/mnt:rslave
```

> You'll need to re-run all the installation steps after recreating the container.

---

## Troubleshooting

### Application Starts But No Window Appears

**For Wayland users:**

Ensure you're using the wrapper script with `QT_QPA_PLATFORM=xcb`, or run manually:

```bash
cd /opt/resolve/bin/
QT_QPA_PLATFORM=xcb ./resolve
```

**For X11 users:**

Try explicitly setting the platform:

```bash
cd /opt/resolve/bin/
QT_QPA_PLATFORM=xcb ./resolve
```

---

### Debug Qt Plugin Loading

Run with debug output to see what's failing:

```bash
cd /opt/resolve/bin/
QT_DEBUG_PLUGINS=1 ./resolve
```

---

### Verify GPU Access

**For NVIDIA:**

```bash
nvidia-smi
```

You should see your GPU listed.

**For AMD:**

```bash
ls -la /dev/dri
```

You should see device files like `card0`, `renderD128`, etc.

---

### Permission Issues

If you get permission errors:

```bash
sudo chown -R root:users /opt/resolve
```

```bash
sudo chmod -R g+rw /opt/resolve
```

---

## HiDPI Display Scaling

If Resolve appears too small on a HiDPI display:

1. Launch DaVinci Resolve
2. Go to **DaVinci Resolve → Preferences** (or `Ctrl + ,`)
3. Navigate to **User → UI Settings**
4. Adjust the **UI Scaling** slider (try 150% or 200%)
5. Restart Resolve

> This is the recommended way to handle HiDPI displays rather than relying on system-level scaling.

---

## Optional: Install FFmpeg Encoder Plugin

The [FFmpeg Encoder Plugin](https://github.com/EdvinNilsson/ffmpeg_encoder_plugin) enables you to export H.264, H.265, and AV1 video from DaVinci Resolve Studio using FFmpeg encoders. This is especially useful since the free codecs available in Resolve on Linux are limited.

**Supported encoders:**
- **H.264:** X264, VAAPI, NVENC
- **H.265:** X265, VAAPI, NVENC
- **AV1:** SVT-AV1, VAAPI, NVENC

### Enable RPM Fusion Repository

FFmpeg is not available in Rocky Linux's default repositories, so we need to enable RPM Fusion:

```bash
distrobox enter resolve
```

```bash
sudo dnf install --nogpgcheck https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

```bash
sudo dnf install --nogpgcheck https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm
```

```bash
sudo dnf install --nogpgcheck https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-8.noarch.rpm
```

### Install FFmpeg

Now install FFmpeg:

```bash
sudo dnf install ffmpeg
```

### Install VAAPI Drivers (For Hardware Acceleration)

VAAPI provides hardware-accelerated encoding. Support varies by GPU type.

**For Intel GPUs:**

```bash
sudo dnf install libva-intel-driver intel-media-driver libva-utils
```

**For AMD GPUs:**

```bash
sudo dnf install mesa-va-drivers libva-utils
```

**For NVIDIA GPUs:**

```bash
sudo dnf install libva-utils
```

> **Note for NVIDIA users:** NVIDIA GPUs have limited VAAPI support on Linux. The FFmpeg plugin will still work using:
> - **NVENC encoders** (H.264, H.265, AV1) - Hardware-accelerated through CUDA, doesn't require VAAPI
> - **CPU encoders** (X264, X265, SVT-AV1) - Software encoding, no GPU required
>
> NVENC is the recommended option for NVIDIA users and should work automatically through the `--nvidia` flag.

Verify VAAPI is working (Intel/AMD only):

```bash
vainfo
```

You should see a list of supported profiles and entrypoints. If you get errors on NVIDIA, this is expected - use NVENC or CPU encoders instead.

### Install the Plugin

Download and install the plugin:

```bash
cd /tmp
```

```bash
wget https://github.com/EdvinNilsson/ffmpeg_encoder_plugin/releases/latest/download/ffmpeg_encoder_plugin.dvcp.bundle.zip
```

```bash
sudo unzip ffmpeg_encoder_plugin.dvcp.bundle.zip -d /opt/resolve/IOPlugins/
```

```bash
rm ffmpeg_encoder_plugin.dvcp.bundle.zip
```

### Restart Resolve

Exit and relaunch DaVinci Resolve. The new export options should now be available in the Deliver page under the Format dropdown.

---

## Uninstalling

### Remove the Desktop Launcher

```bash
rm ~/.local/share/applications/davinci-resolve.desktop
```

**For Wayland users**, also remove the wrapper script:

```bash
rm ~/.local/bin/resolve-wrapper
```

---

### Remove the Container

```bash
distrobox rm resolve
```

This removes the entire container including Resolve and all dependencies.

---

## Additional Notes

- **Studio vs Free:** DaVinci Resolve Studio (paid version) is required for full GPU acceleration and some advanced features on Linux
- **Codecs:** Some video codecs may require additional plugins or libraries
- **GPU drivers:** Ensure your GPU drivers are properly installed on your host system
- **Updates:** To update Resolve, download the new installer and run it the same way

---

## Why Use Distrobox?

- **Isolation:** Keeps Resolve's dependencies separate from your main system
- **Easy cleanup:** Completely remove Resolve by deleting the container
- **Compatibility:** Rocky Linux provides a stable enterprise environment
- **GPU access:** Proper GPU passthrough for rendering performance
- **No system pollution:** All dependencies stay in the container
# Notes on Displaylink and EVDI on Ubuntu 22.04 with Kernel 6.2

So I did a thing....

Yep, I in-place upgraded Ubuntu 20.04 to 22.04 and it made my life a mess from
a Displaylink point of view.  Laptop screen and screens attached to dock work,
but screen plugged into USB with Displaylink adapter would not come up.

This is one of potentially many unknown issues with this upgrade (e.g. none of
my Ansible venvs work at the moment!), but I'm going to document what I did to
get that Displaylink thing done.

## Get your environment right to build

Get your bearings straight first.  If you have a python venv activated, go
ahead and deactivate it.

List your pythons...

```bash
ls -la /usr/bin/python*

# you'll likely see entries for python2.7 and python3.10
```

Let's use `update-alternatives` to ensure we're using python3 by default.

```bash
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2.7 1
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.10 2

# now we select the default one
sudo update-alternatives --config python

# pick the one you want from the list, and confirm...

python --version
```

Make sure we have some packages

```bash
sudo apt install python3-dev
sudo apt install pybind11-dev

# yes, this part hurts my soul a little bit, but yolo
sudo pip install pybind11
```

## Resources

I'm going to lay down some links that helped me get through this.  You need to
read and heed.  Some of this probably depends on how jacked your system is due
to in-place upgrades, but...

1. https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu-5.6.1?filetype=exe
2. https://code.berrydejager.com/Fix-DisplayLink_drivers-linux-kernel-6/
3. https://github.com/DisplayLink/evdi/issues/402
4. https://github.com/DisplayLink/evdi/pull/401

We kind of need to combine stuff from all of the above.

## Get source, make some modifications, and build

OK, before you go in, understand that the Synaptics driver currently (as of
2023-04-07) wants to target evdi-1.12.0.  But you'll be building evdi version
1.13.1.  So there is some above and beyond hackery that needs to happen here.

> I'm assuming you have `build-essentials` and other packages installed to
> support `make` via `Makefile`, compling with `gcc` and friends...

```bash
mkdir -p ~/workspace/github/
cd ~/workspace/github/

git clone https://github.com/DisplayLink/evdi.git
cd evdi
```

> I'm going to identify the files that need changes below, but I'm probably
> going to go ahead and fork the evdi repo to my personal GitHub account and
> push the files there so you can reference/diff them to see the changes.

We need to modify `library/evdi_lib.h` to say this will be version 1.12.0. Yes,
we're technically lying here as we're building something that the community has
agreed to label 1.13.1, but Synaptics and their installers aren't patched/ready
for that yet.  Set LIBEVDI_VERSION_MINOR to 12.

We need to modify `module/Makefile` to put the same version lie in place. Set
MODVER=1.12.0.

We need to modify `module/dkms.conf` to tell the same lie.  Set PACKAGE_VERSION
to 1.12.0.

> Please notice that this is all being done against the `devel` branch I cloned
> on 2023-04-07.  So YMMV depending on _when_ you do this and what version of
> the Synaptics Displaylink drivers you download/install.

Now you can `make` and install...

```bash
make
sudo make install
```

This didn't make any noticeable changes, even after restarting displaylink
service:

```bash
sudo systemctl restart displaylink-driver.service
```

## Hack the Synaptics installer and get this working

So we've got our locally built evdi, and looking at instructions from resource
#2 above, we see the steps to obtain the installer, extract it, extract again,
and use a `curl` command to get some `evdi.tar.gz` file to feed the extracted
installer script.  BUT, that `evdi.tar.gz` has all of the version info tied
to 1.13.1.  So we really need to make our own `*.tar.gz` with our edits.

```bash
mkdir -p ~/Downloads/displaylink/5.6.1/
cd ~/Downloads/displaylink/5.6.1/
```

Download 'DisplayLink USB Graphics Software for Ubuntu5.6.1-EXE.zip' from
Synaptics (see resource #1) and place zip file into above folder

```bash
unzip DisplayLink\ USB\ Graphics\ Software\ for\ Ubuntu5.6.1-EXE.zip

# if you had run the installer previously..
sudo ./displaylink-driver-5.6.1-59.184.run uninstall

./displaylink-driver-5.6.1-59.184.run --noexec --keep
cd displaylink-driver-5.6.1-59.184/

# hang out here for a minute
```

In another terminal window

```bash
cd ~/workspace/github/
mkdir evdi-devel
cp -R evdi/* evdi-devel/
tar -cvf evdi.tar.gz evdi-devel
```

Now go back to your displaylink folder...

```bash
cp ~/workspace/github/evdi.tar.gz .

# take a not of what dkms knows about
dkms status

# we need to nuke em all
sudo dkms remove evdi/1.13.1 --all
sudo dkms remove evdi/1.7.0 --all

# check if anything is still loaded in kernel memory and nuke it
lsmod | grep evdi
sudo modprobe -r evdi

# run our patched installer and hold onto your butts
sudo ./displaylink-installer.sh
```

If `displaylink-driver.service` is still running, I think this will "just work"
and you should see the display come to life.  Otherwise you can try using
`systemctl` to restart it and/or reboot your box.

Alas, this is all the stuff I went through to get that monitor working.

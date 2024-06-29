# Bootc Example
## Description:
This project is to go over how to use a bootc image using Centos Stream as well as going over a few issues I had found while trying to use podman to build the images on both Windows Podman Desktop and WSL. It seems like lots of the issues I ran across could be due to me operating on Windows vs Linux, but figured I'd create this to help someone along the way.    

I also am going to be using the anaconda iso installer vs the qcow2 method of image provisioning for installing on bare matel or VMs without an image. ISO images take longer to create, but they seem to be pretty versitile in their application.

**PLEASE USE WSL ON WINDOWS IF USING A WINDOWS BOX FOR THIS EXAMPLE**  
I have had much more success using WSL on Windows when following the Centos Bootc examples vs Podman Desktop. To get to WSL on Windows (Assuming you already have it installed), bring up a terminal and type the following: 
   
Windows Only (Install Podman Desktop or CLI if you haven't yet):  
```
wsl
```

## Bootc Example Steps:
### Step 1: Setup the Environment:
Install qemu to be able to build ISO (You may need a differnt qemu version installed if you are not working with an x86_64 system):
```
sudo dnf install qemu-img qemu
```  
Create the podman machine if not already setup: 
```
podman machine init
```   

Ensure podman is running it's machine in a rootful state
```
podman machine stop
podman machine set --rootful
podman machine start
```
cd into the directory where you have this repo cloned\copied to.
```
cd <path_to_repo>
```
Configure a config.toml file to create your first user. In the provided template, you can rename that from "config-template.toml" to "config.toml".

config.toml:
``` 
[[customizations.user]]
name = "admin"
password = "admin"
key = "ssh-rsa AAA ... user@email.com"
groups = ["wheel"] 
```

Pull down the Centos image locally:
```
podman pull quay.io/centos-bootc/centos-bootc:stream9
```  
### Step 2: Create a custom image:

Have podman take the Centos Stream image and apply our ContainerFile to the image:
```
podman build --tag centos-bootc:1.0 -f ContainerFile
```
View your new image with the following:
```
podman images
```
You should see a localhost/centos-bootc image with a "1.0" tag.
### Step 3: Push Image to quay.io
The following assumes you already have a quay.io account. If not, go ahead and create an account at https://quay.io.

Login to quay.io:
```
podman login quay.io
```
Push your local image to quay.io:
```
podman push localhost/centos-bootc:1.0 quay.io/<username>/centos-bootc:1.0
```

### Step 4: Create your ISO:

Ensure your root user can see the local container image.  
``` 
sudo podman pull quay.io/<username>/centos-bootc:1.0
```
Run the following command to create your ISO
``` 
sudo podman run --rm  -it --privileged --pull=newer \
--security-opt label=type:unconfined_t \
-v /var/lib/containers/storage:/var/lib/containers/storage \
-v ./config/config.toml:/config.toml \
-v $(pwd)/output:/output \
quay.io/centos-bootc/bootc-image-builder:latest \
--type anaconda-iso --rootfs xfs \
--local quay.io/<username>/centos-bootc:1.0
```

### Step 5: Boot your machine:
After the last step, you should have an install.iso file in your output/bootiso folder. Take that iso and load it on any bare metal VM or physical machine and it should load Centos Stream on your box.  

If you left the config.toml file unchanged, your login should be "admin" with the password "admin"  

browse to "http://<server_ip>" in a browser to verify your custom image is running a website. 
### Step 6: Create a second custom image:
Now we want to update our image to change the website some. Assume Containerfile2 has the changes you want to apply (just some new wording in index2.html).

Have podman take the Centos Stream image and apply our ContainerFile to the image:
```
podman build --tag centos-bootc:1.1 -f ContainerFile2
```
View your new image with the following:
```
podman images
```
You should see a localhost/centos-bootc image with a "1.1" tag.

### Step 7: Push New Image to quay.io
The following assumes you already have a quay.io account. If not, go ahead and create an account at https://quay.io.

Login to quay.io:
```
podman login quay.io
```
Push your local image to quay.io:
```
podman push localhost/centos-bootc:1.1 quay.io/<username>/centos-bootc:1.1
```
### Step 8: Swap your bootc Image:
Login to the machine you imaged with your ISO back on step 3 and double verify what image your machine is currently :
```
bootc status
```
You should see something similar to: "image: quay.io/centos-bootc/centos-bootc:stream9" somewhere in that blob of text.

Swap your bootc image to your custom image:
```
bootc switch quay.io/<username>/centos-bootc:1.1
```
If your run your bootc status again, you should see your image has changed and it's "staging" image to your new image.

Reboot your server and browse to "http://<server_ip>" in a browser to verify your custom image is running your updated website. 

Congratulations! you swapped an image!

## Troubleshooting:
### Issue 1: Compression:
Sometimes I ran into an issue where ```bootc switch``` returned ```ERROR Switching: Pulling: Importing: Unhandled layer type: application/vnd.oci.image.layer.v1.tar+zstd ```. 

If you ran into this, that means podman pushed your image as a zstd compressed image and bootc cannot recognize it. To fix it redo step 3 and 7's "podman push" and add the ```--compression-format=gzip ``` flag right after the "push" command. 

The zstd format may be supported in future versions of bootc, but for now, I'm hoping this saves someone some headaches that it caused me.


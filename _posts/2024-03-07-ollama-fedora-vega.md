## Running Ollama on Fedora Workstation with AMD Vega Frontier Edition GPU

> **_NOTE:_** I kept all my notes in the process of getting this working as there were some tools and techniques that I want to remember related to hardware and driver troubleshooting. The solution though ended up making much of the early work unnecessary. I'd recommend anyone looking to use Ollama with AMD cards focus on using the container method as it includes the necessary drivers that may not be easy to find and configure for your OS/distribution.

There is a great guide by Anvesh G. Jhuboo at [Setting up ROCm and PyTorch on Fedora: A Step-By-Step Guide](https://medium.com/@anvesh.jhuboo/rocm-pytorch-on-fedora-51224563e5be) that walks through the steps and commands. The Vega Frontier Edition is a different beast of a video card though. Its intended as a workstation card and thus always has its own "isms" whenever I try to use it for things.

ROCm v6 is not available in Fedora 39 and is expected in Fedora 40. The directions for enabling / testing this can be found at [Fedora SIG HC](https://fedoraproject.org/wiki/SIGs/HC#Installation)

### Problem - No GPU detected

As of v0.1.27 the Vega Frontier Edition is not detected automatically when just downloading and running ollama on my Fedora 39 workstation. 

```
$ ollama serve
Couldn't find '/home/jeff/.ollama/id_ed25519'. Generating new private key.
Your new public key is: 

ssh-ed25519 <cut>

time=2024-02-25T13:47:27.161-05:00 level=INFO source=images.go:710 msg="total blobs: 0"
time=2024-02-25T13:47:27.161-05:00 level=INFO source=images.go:717 msg="total unused blobs removed: 0"
time=2024-02-25T13:47:27.161-05:00 level=INFO source=routes.go:1019 msg="Listening on 127.0.0.1:11434 (version 0.1.27)"
time=2024-02-25T13:47:27.161-05:00 level=INFO source=payload_common.go:107 msg="Extracting dynamic libraries..."
time=2024-02-25T13:47:29.783-05:00 level=INFO source=payload_common.go:146 msg="Dynamic LLM libraries [cpu_avx cpu cuda_v11 cpu_avx2 rocm_v5 rocm_v6]"
time=2024-02-25T13:47:29.783-05:00 level=INFO source=gpu.go:94 msg="Detecting GPU type"
time=2024-02-25T13:47:29.783-05:00 level=INFO source=gpu.go:265 msg="Searching for GPU management library libnvidia-ml.so"
time=2024-02-25T13:47:29.789-05:00 level=INFO source=gpu.go:311 msg="Discovered GPU libraries: []"
time=2024-02-25T13:47:29.789-05:00 level=INFO source=gpu.go:265 msg="Searching for GPU management library librocm_smi64.so"
time=2024-02-25T13:47:29.790-05:00 level=INFO source=gpu.go:311 msg="Discovered GPU libraries: []"
time=2024-02-25T13:47:29.790-05:00 level=INFO source=cpu_common.go:11 msg="CPU has AVX2"
time=2024-02-25T13:47:29.790-05:00 level=INFO source=routes.go:1042 msg="no GPU detected"
```


### Troubleshooting

### Add your user to the video group

NOTE: All commands are run as your non-privileged user unless specified otherwise
```
$ sudo usermod -a -G render,video $LOGNAME
$ id $LOGNAME
uid=1000(jeff) gid=1000(jeff) groups=1000(jeff),10(wheel),39(video),986(libvirt),1001(docker),969(render)
```

Notice the groups now include video

### Verify that the GPU was discovered and using the amdgpu drivers

```
$ lsmod | grep amdgpu
amdgpu              13348864  37
amdxcp                 12288  1 amdgpu
i2c_algo_bit           20480  1 amdgpu
drm_ttm_helper         12288  1 amdgpu
ttm                   110592  2 amdgpu,drm_ttm_helper
drm_exec               12288  1 amdgpu
gpu_sched              65536  1 amdgpu
drm_suballoc_helper    12288  1 amdgpu
drm_buddy              20480  1 amdgpu
drm_display_helper    229376  1 amdgpu
video                  77824  1 amdgpu
```

### Installing rocminfo to see the ROCM support for this card

The rocminfo command provides a lot of output and should list all available agents. We are verifying that the video card is listed as one of those agents. 

```
# Install needed packages
$ dnf install rocminfo inxi

# List all ROCm info
$ rocminfo

# Search for our video card in the output
$ rocminfo | grep Vega

# List all system information
$ inxi -Fzxx

# List only graphics system information
$ inxi -Gzxx
```

dnf install rocm-opencl

### Install rocm-hip (needed?)

As of Fedora 39 this is now available from the included updates repository

```
$ dnf install rocm-hip rocm-hip-devel
```

### Install ROCm System Management Interface rocm-smi 

We are actually installing the development package for rocm-smi to provide the librocm_smi64.so library


```
$ dnf install rocm-smi-devel
```


Didn't work :-/

### The Solution (finally getting it to work)

Not sure why I put off using the container version, but it ultimately made this a lot simpler. I wanted to run this as a non-privileged user so I went with Podman as a systemd service running in user mode.

#### Rootless Podman

As amazing as rootless podman is, it does require a bit more setup than just using Docker. I think its worth it though.

First you need to make sure you have the prerequisite packages installed.
```
$ sudo dnf install podman skopeo
```
I know I'll be passing hardware through to the containers so I enabled the selinux boolean to allow that.
```
$ sudo setsebool container_use_devices=true
```
I needed to create a non-privileged user that I can run the ollama service from.
```
$ useradd ollama
```
I thought I was going to need to add ollama to the /etc/subgid and /etc/subuid files as I'd done in the past to enable rootless podman, but it was already there. Perhaps this is a newer feature of podman... or maybe I blacked out and did it without realizing it. Either way, its important that you have entries in these files like what is shown below to allow the user to create additional namespaces.
```
$ cat /etc/subuid | grep ollama
ollama:524288:65536
$ cat /etc/subgid | grep ollama
ollama:524288:65536
```

#### Creating the service

For some reason, I can't remember how I came to the conclusion, I needed to add the ollama user to the render and video groups.
```
$ sudo usermod -a -G render,video ollama
```

Then I pulled the latest ollama image that contains the packages/drivers necessary for using rocm. As you can see its a chunky container so this may be a good time for a coffee break if you've got a slow connection. When I pulled it I don't recall seeing a generic 'rocm' tag available. There is one now though, as well as version specific tags. You can find the latest images available on [ollama/ollama Dockerhub](https://hub.docker.com/r/ollama/ollama/tags)
```
$ podman pull ollama/ollama:0.1.27-rocm
$ podman images | grep ollama
docker.io/ollama/ollama        0.1.27-rocm  f50147ec1d2a  12 days ago  21 GB

```

Since I know I'll want to have a container run even when I'm not signed in as the ollama user, I had to enable-linger for that account.
```
$ id ollama
uid=1002(ollama) gid=1003(ollama) groups=1003(ollama),39(video),969(render)
$ sudo loginctl enable-linger 1002
```
The process I used here could be optimized. Currently its version locked to a specific image. When I next upgrade it, I may switch it.

As the user ollama, I ran podman create mapping my local devices to the container, and a local folder for configurations.

```
$ /usr/bin/podman --runtime /usr/bin/crun create --device=/dev/kfd:/dev/kfd --device=/dev/dri:/dev/dri -v /home/ollama/.ollama:/root/.ollama:Z -p 1
1434:11434 --name ollama 'ollama/ollama:0.1.27-rocm'
```

I then created a systemd service to launch that container with a restart policy of always.

```
$ podman generate systemd --files --restart-policy=always --name ollama
$ cat container-ollama.service
# container-ollama.service
# autogenerated by Podman 4.9.3
# Sun Feb 25 19:40:20 EST 2024

[Unit]
Description=Podman container-ollama.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/tmp/containers-user-1002/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=always
TimeoutStopSec=70
ExecStart=/usr/bin/podman start ollama
ExecStop=/usr/bin/podman stop  \
	-t 10 ollama
ExecStopPost=/usr/bin/podman stop  \
	-t 10 ollama
PIDFile=/tmp/containers-user-1002/containers/overlay-containers/0b89e0c6a77583e6925ecfcc214620ff74b1d59d069bb41d066812e5c2b0d2cc/userdata/conmon.pid
Type=forking

[Install]
WantedBy=default.target
```
I then copied the container-ollama.service file it generated to ~/.config/systemd/user/container-ollama.service.
```
$ cp container-ollama.service .config/systemd/user/
```
Now it was time to start that service. I ran into an error when attempting to start a service using the --user flag that required that I define the XDG_RUNTIME_DIR for the ollama user.
```
$ echo "export XDG_RUNTIME_DIR=/run/user/$(id -u)" > ~/.bashrc.d/systemd 
$ source ~/.bashrc.d/systemd
$ systemctl --user enable container-ollama
$ systemctl --user start container-ollama
```
Over a week later it is still running strong.
```
$ systemctl status --user container-ollama
● container-ollama.service - Podman container-ollama.service
     Loaded: loaded (/home/ollama/.config/systemd/user/container-ollama.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/user/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Sun 2024-02-25 23:01:13 EST; 1 week 3 days ago
```
### Making it usable for others
As great as the terminal access was, I have some kids that were excited to "train" this AI to play some games. So I needed a better interface. Thankfully most of what we need has already been done setting up ollama to work.

I looked through the massive list of options provided in the [Ollama Github - Community Integrations](https://github.com/ollama/ollama?tab=readme-ov-file#community-integrations) write up and settled on [Open WebUI (Formerly Ollama WebUI)](https://github.com/open-webui/open-webui).

They provide a handy container that works great in a rootless podman setup. I created a directory to store configurations and created a container for it.
```
$ mkdir open-webui
$ podman create -v open-webui:/app/backend/data:Z -e OLLAMA_API_BASE_URL=http://127.0.0.1:11434/api --name open-webui -p 3000:8080 --name open-webui ghcr.io/open-webui/open-webui:main
```
I think I must have been getting tired at this point because rather than created a systemd unit for it like I did with ollama, I just started it in daemon mode.
```
$ podman run -d open-webui
$ podman ps
CONTAINER ID  IMAGE                                COMMAND        CREATED      STATUS      PORTS                     NAMES
0b89e0c6a775  docker.io/ollama/ollama:0.1.27-rocm  serve          10 days ago  Up 10 days  0.0.0.0:11434->11434/tcp  ollama
0a604c801e13  ghcr.io/open-webui/open-webui:main   bash start.sh  10 days ago  Up 10 days  0.0.0.0:3000->8080/tcp    open-webui

```
Unlike ollama that I was using from the local terminal, this I wanted to expose to users on my local network. That means that I needed to open the port necessary to connect.
```
$ firewall-cmd --add-port 11434/tcp --permanent
$ firewall-cmd --reload
```

With that I was now able to access Ollama from anywhere in the house. Open-WebUI is really a fantastic interface. The ability to quickly switch models and upload documents directly through the interface makes it a must have.

### Resources that really helped me figure this out
The work that so many people have put into making all this work is humbling. Here are some of the resources that really helped me fumble though getting all this working correctly.
1. [Podman Github - Rootless tutorial](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)
1. [Podman.io - Podman generate systemd](https://docs.podman.io/en/stable/markdown/podman-generate-systemd.1.html)
1. [prawilny/ollama-rocm-docker](https://github.com/prawilny/ollama-rocm-docker/blob/master/README.md)
1. [Ollama Github Issue #2718- Doc permission requirements for Rocm Docker Image to access /dev/dri and /dev/kfd](https://github.com/ollama/ollama/issues/2718)
1. [Ollama Github Issue #2749- AMD GPU ROCm library search path hardcoded to wrong path](https://github.com/ollama/ollama/issues/2749)
1. [Ollama Github Issue #2165 - ROCm v5 crash - free(): invalid pointer](https://github.com/ollama/ollama/issues/2165)
1. [Ollama Github Issue #738 - AMD GPU & ROCm support](https://github.com/ollama/ollama/issues/738)
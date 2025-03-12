# Raspberry Pi 4 Model B - Camera Streaming Setup
This guide walks you through setting up **camera-streamer** on a **Raspberry Pi 4 Model B** with **libcamera** support, **RTSP streaming**, and **firewall configurations**. 

**GitHub Source for camera-streamer:** [ayufan/camera-streamer](https://github.com/ayufan/camera-streamer/tree/main)
---

## **System Requirements**
- **Raspberry Pi 4 Model B**
- **Raspberry Pi OS (Bookworm)**
- **Camera Module (Arducam 5MP OV5647 Miniature Camera Module with Long Flex Cable)**  
- **Connected via Wi-Fi or Ethernet**
- **Tailscale (Optional)**

### **1️⃣ Install Required Dependencies**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt update
sudo apt install -y libavformat-dev libavutil-dev libavcodec-dev \
libcamera-dev liblivemedia-dev v4l-utils pkg-config xxd \
build-essential cmake libssl-dev libcamera0.4 libcamera-apps \
libcamera-tools python3-libcamera -y
```

### **2️⃣ Validate your system**

This streamer requires hardware encoding/decoding. Ensure the following devices exist:

1. ISP (`/dev/video13`, `/dev/video14`, `/dev/video15`)
2. JPEG encoder (`/dev/video31`)
3. H264 encoder (`/dev/video11`)
4. JPEG/H264 decoder (for UVC cameras, `/dev/video10`)
5. At least LTS kernel (5.15, 6.1) for Raspberry PIs

Run:
```bash
uname -a
v4l2-ctl --list-devices
```

If the devices are missing, upgrade your Raspberry Pi OS kernel:
```bash
sudo apt update
sudo apt dist-upgrade
sudo reboot
```

Modify the boot configuration file:
```bash
sudo nano /boot/firmware/config.txt
```

Add the following lines:
```bash
# Increase GPU memory for camera streaming
gpu_mem=256
dtoverlay=vc4-kms-v3d,cma-128

# Enable support for OV5647 camera (Arducam 5MP Miniature)
dtoverlay=ov5647
start_x=1
```

Save and exit (CTRL+X, then Y, then ENTER), then reboot:
```bash
sudo reboot
```

After reboot, verify the camera is detected:
```bash
libcamera-hello
```

### **3️⃣ Clone and Compile Camera-Streamer**
Since the main camera-streamer repository has known libcamera compatibility issues, we need to checkout a fixed branch:
```bash
git clone https://github.com/ayufan-research/camera-streamer.git --recursive
cd ~/camera-streamer

# Fetch the libcamera fix
git fetch origin pull/160/head:fix-libcamera
git checkout fix-libcamera

# Clean and rebuild
make clean
make
sudo make install
```

### **4️⃣ Allow Required Ports in Firewall**
```bash
sudo ufw allow 8080/tcp
sudo ufw allow 8554/tcp
sudo ufw enable
sudo ufw reload
```

### **5️⃣ Start Camera-Streamer Manually**
```bash
camera-streamer \
  --camera-path=/base/soc/i2c0mux/i2c@1/ov5647@36 \
  --camera-type=libcamera \
  --camera-width=1280 \
  --camera-height=720 \
  --camera-fps=30 \
  --camera-video.disabled=0 \
  --camera-video.height=720 \
  --http-listen=0.0.0.0 \
  --http-port=8080 \
  --rtsp-port=8554
```
### **6️⃣ Viewing the Stream**

If it starts successfully, you can access:
- http://(RPI-IP):8080/
- Replace (RPI-IP) with either your Raspberry Pi's local or Tailscale IP (e.g., 192.168.1.100).
*if using the Rapsberry Pi's local IP then the viewing device has to be connected to the same network as the pi*

Error messages can be read:
```bash
journalctl -u camera-streamer -xe
```

Or restart the service:
```bash
sudo systemctl restart camera-streamer
```



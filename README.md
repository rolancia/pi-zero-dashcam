# Pi-Dashcam-SSD

Turn my Raspberry Pi Zero 2 W into a ssd emulator for Dashcams (Tesla, BlackVue, Thinkware, etc.).

---

### 1. Initialization & Cleanup

```bash
sudo modprobe -r g_mass_storage
sudo losetup -D
rm -f /home/rolancia/dashcam.img
sudo rm -f /etc/modprobe.d/g_mass_storage.conf
```

### 2. Disk Image Allocation (Pre-allocation)
fallocate 는 대시캠에서 에러

```bash
# screen 세션 시작
screen -S dashcam_build

# 200GB 이미지 생성 (bs=1M, count=200000)
dd if=/dev/zero of=/home/rolancia/dashcam.img bs=1M count=200000 status=progress
```

### 3. Partitioning & Formatting

```bash
sudo losetup -fP --show /home/rolancia/dashcam.img
# 순서: o (MBR) -> n (New) -> p (Primary) -> 1 (Default) -> Enter -> Enter -> t (Type) -> 7 (exFAT) -> w (Write)
sudo fdisk /dev/loop1 <<EOF
o
n
p
1


t
7
w
EOF

# 3. exFAT 포맷 (Label: Samsung_T7)
sudo mkfs.exfat -L "Samsung_T7" /dev/loop1p1

# 4. 루프 장치 해제
sudo losetup -D
```

### 4. Gadget Configuration (The Spoofing)
라즈베리파이를 Samsung Portable SSD T7으로 위장

`sudo vi /etc/modprobe.d/g_mass_storage.conf`

```bash
# removable=0 : 고정 디스크로 인식
# nofua=1     : 쓰기 성능 최적화
# idVendor/idProduct : Samsung T7 식별자

options g_mass_storage file=/home/rolancia/dashcam.img stall=0 removable=0 nofua=1 idVendor=0x04e8 idProduct=0x61f5 iManufacturer="Samsung" iProduct="Portable SSD T7" iSerialNumber="SAMSUNG_T7_1234"
```

### 5. Boot Configuration

**1) `/boot/firmware/config.txt`**
맨 아래 라인에 추가:
```ini
dtoverlay=dwc2,dr_mode=peripheral
```

**2) `/boot/firmware/cmdline.txt`**
모듈 로드 옵션 추가:
```text
modules-load=dwc2,g_mass_storage
```

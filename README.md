# วิธีติดตั้ง ZeroTier และตั้งค่า LAN Bridge แบบถาวรบน Proxmox LXC (Debian/Ubuntu)

คู่มือนี้จะแนะนำวิธีการติดตั้ง ZeroTier และตั้งค่า LAN Bridge แบบถาวรบน Proxmox LXC Container ที่ใช้ระบบปฏิบัติการ Debian หรือ Ubuntu เพื่อเชื่อมต่อเครือข่าย ZeroTier กับ LAN จริง โดยใช้การตั้งค่า NAT และ static route

---

## 🧱 สิ่งที่ต้องมี
- **เครื่อง Linux**: เช่น Ubuntu 22.04/24.04 หรือ Debian 12 ที่รันใน Proxmox LXC
- **LAN Interface**: เช่น `ens18` (ตรวจสอบด้วย `ip a`)
- **ZeroTier Network**: สร้างและอนุญาต (authorized) เครื่องใน ZeroTier Central
- **Network LAN จริง**: เช่น `192.168.0.0/24`
- **ZeroTier IP**: เครื่อง bridge ได้รับ IP เช่น `192.168.192.x`

---

## 🪜 ขั้นตอน

### 0. ติดตั้ง ZeroTier
1. **ติดตั้ง ZeroTier One**  
   - ดาวน์โหลดและติดตั้ง ZeroTier ด้วยคำสั่ง:
     ```bash
     curl -s https://install.zerotier.com | sudo bash
     ```
     หรือหากต้องการติดตั้งอย่างปลอดภัยด้วย GPG:
     ```bash
     curl -s 'https://pgp.mit.edu/pks/lookup?op=get&search=0x1657198823E52A61' | sudo gpg --import && \
     if z=$(curl -s 'https://install.zerotier.com/' | gpg); then echo "$z" | sudo bash; fi
     ```

2. **ตรวจสอบการติดตั้ง**  
   - ตรวจสอบว่า ZeroTier ทำงาน:
     ```bash
     sudo zerotier-cli info
     ```
     คำสั่งนี้จะแสดงข้อมูลเช่น Address และสถานะ ONLINE

---

### 1. เตรียม LXC Container
1. **แก้ไข Config สำหรับ Unprivileged Container**  
   - แก้ไขไฟล์ `/etc/pve/lxc/<CTID>.conf` (แทน `<CTID>` ด้วย ID ของ Container):
     ```bash
     nano /etc/pve/lxc/<CTID>.conf
     ```
   - เพิ่มบรรทัดต่อไปนี้เพื่อรองรับ ZeroTier และ Docker:
     ```plaintext
     unprivileged: 1
     features: nesting=1,keyctl=1
     lxc.apparmor.profile: unconfined
     lxc.cgroup.devices.allow: a
     lxc.cap.drop:
     lxc.cgroup2.devices.allow: c 10:200 rwm
     lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
     ```

2. **รีสตาร์ท LXC Container**  
   - รีสตาร์ท Container เพื่อให้การตั้งค่ามีผล:
     ```bash
     pct restart <CTID>
     ```

---

### 2. Join ZeroTier และตรวจสอบ IP
1. **Join เครือข่าย ZeroTier**  
   - รันคำสั่งเพื่อเข้าร่วมเครือข่าย ZeroTier:
     ```bash
     sudo zerotier-cli join <network-id>
     ```
     แทน `<network-id>` ด้วย ID จาก ZeroTier Central (เช่น `d5e04297a16fa690`)

2. **ตรวจสอบสถานะและ IP**  
   - ตรวจสอบว่าเข้าร่วมเครือข่ายสำเร็จและได้รับ IP:
     ```bash
     sudo zerotier-cli listnetworks
     ```
     ผลลัพธ์ควรแสดงสถานะ `OK` และ interface เช่น `ztxxxxxxx` พร้อม IP เช่น `192.168.192.109`

---

### 3. เปิด IP Forwarding
- เปิดใช้งาน IP forwarding เพื่อให้เครื่องทำหน้าที่เป็น bridge:
  ```bash
  echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
  sudo sysctl -p
  ```

---

### 4. ตั้งค่า NAT (ZeroTier → LAN)
- ตั้งค่า iptables เพื่อทำ NAT จาก ZeroTier ไปยัง LAN:
  ```bash
     # เปิด IP forwarding
   sudo sysctl -w net.ipv4.ip_forward=1
   sudo sed -i 's/^#\?net.ipv4.ip_forward.*/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   
   # NAT จาก ZeroTier ไป LAN
   sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   
   # อนุญาต Forward
   sudo iptables -A FORWARD -i eth0 -o ztklhwkfmx -m state --state RELATED,ESTABLISHED -j ACCEPT
   sudo iptables -A FORWARD -i ztklhwkfmx -o eth0 -j ACCEPT
  ```
---

### 5. ทำให้ NAT ถาวร
1. **ติดตั้ง iptables-persistent**  
   - ติดตั้งแพ็คเกจเพื่อบันทึกกฎ iptables:
     ```bash
     sudo apt install -y iptables-persistent
     ```
     ตอบ `yes` เมื่อถูกถามให้บันทึกกฎ iptables ปัจจุบัน

2. **เปิดใช้งาน netfilter-persistent**  
   - ตรวจสอบว่าเซอร์วิสทำงาน:
     ```bash
     sudo systemctl enable netfilter-persistent
     sudo systemctl start netfilter-persistent
     ```

---

### 6. ตั้งค่า Static Route บน ZeroTier Central
1. **ไปที่ ZeroTier Central**  
   - เปิดเบราว์เซอร์และไปที่ [https://my.zerotier.com](https://my.zerotier.com)
   - เลือกเครือข่ายของคุณ → ไปที่ **Advanced** → **Managed Routes**

2. **เพิ่ม Route**  
   - เพิ่ม route ดังนี้:
     ```
     Destination: 192.168.0.0/24
     Via: 192.168.192.109
     ```
     - `192.168.0.0/24` คือเครือข่าย LAN จริง
     - `192.168.192.109` คือ IP ของเครื่อง bridge ใน ZeroTier

---

### 7. ทดสอบการเชื่อมต่อ
- จากเครื่องใน ZeroTier network (เช่นเครื่อง client):
  ```bash
  ping 192.168.0.1   # ping router
  ping 192.168.0.184 # ping อุปกรณ์ใน LAN จริง
  ```

---

### 8. ทดสอบหลังรีสตาร์ท
1. **รีบูตเครื่อง**  
   - รีสตาร์ทเครื่องเพื่อยืนยันการตั้งค่าถาวร:
     ```bash
     sudo reboot
     ```

2. **ตรวจสอบหลังบูต**  
   - ตรวจสอบสถานะ ZeroTier:
     ```bash
     sudo zerotier-cli listnetworks  # ต้องแสดง Status = OK
     ```
   - ตรวจสอบ IP และ interface:
     ```bash
     ip a  # ต้องมี interface ztxxxxxxx และ IP เช่น 192.168.192.109
     ```
   - ตรวจสอบกฎ iptables:
     ```bash
     sudo iptables -t nat -L -n -v  # ต้องมี MASQUERADE
     ```

---

## 💬 หมายเหตุ
- **ความง่ายและปลอดภัย**: การตั้งค่าแบบ NAT นี้ใช้งานง่ายและปลอดภัยกว่า Layer 2 bridging เพราะไม่ต้องยุ่งกับ DHCP หรือ bridge จริง
- **Reverse Route**: หากต้องการให้ LAN เข้าถึง ZeroTier ได้ ให้เพิ่ม route และกฎ iptables บน router
- **ZeroTier Central**: ต้องอนุญาต (authorize) อุปกรณ์ทุกตัวใน ZeroTier Central เพื่อให้เชื่อมต่อได้
- **LAN Interface**: ตรวจสอบชื่อ interface ให้ถูกต้อง (เช่น `ens18`, `eth0`) โดยใช้ `ip a`
- **Unprivileged LXC**: การตั้งค่า `lxc.cgroup2.devices.allow` และ `lxc.mount.entry` จำเป็นสำหรับ unprivileged container เพื่อให้ ZeroTier ทำงานได้

---

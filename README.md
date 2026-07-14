# Secure Hybrid Infrastructure IaC

ระบบโครงสร้างพื้นฐานเครือข่ายความปลอดภัยสูงและตั้งค่าอัตโนมัติ (Automated Provisioning & Configuration) จำลองผ่าน Vagrant, Ansible และ WireGuard VPN พร้อมระบบเฝ้าระวังเซิร์ฟเวอร์ (Server Monitoring) ด้วย Prometheus และ Grafana

## การออกแบบสถาปัตยกรรมเครือข่าย (Architecture Design)

```text
+-------------------------------------------------------------+
|                     [ Host OS / Developer ]                 |
|                                |                            |
|                     WireGuard Tunnel (10.8.0.2)             |
|                                v                            |
|             [ Gateway / VPN Server (10.8.0.1) ]             |
|             (Prometheus, Grafana, Node Exporter)            |
|                  Public IP: 192.168.56.10                   |
|                  Internal LAN IP: 10.0.0.1                  |
|                                |                            |
|                  +-------------+-------------+              |
|                  |                           |              |
|                  v                           v              |
|         [ Web Server (web) ]        [ Database Server (db) ]|
|           (Node Exporter)             (Node Exporter)       |
|             IP: 10.0.0.11               IP: 10.0.0.12       |
|             Port: 80 (HTTP)             Port: 5432 (Postgres)|
+-------------------------------------------------------------+
```

โปรเจกต์นี้จำลองรูปแบบโครงสร้างระบบเครือข่ายระดับองค์กร (Production-grade Infrastructure Pattern):
- **การแยกเครือข่ายภายใน (Private Subnet Isolation)**: Web Server และ Database Server จะทำงานอยู่ในวงแลนจำลองปิด (`secure_lan`) โดยไม่มี IP สาธารณะต่อเข้าอินเทอร์เน็ตโดยตรง เพื่อลดความเสี่ยงจากการโจมตีภายนอก
- **ทางเข้าส่วนกลาง (Bastion/VPN Gateway)**: เครื่อง Gateway จะเป็นตัวกลางจุดเดียวในการเชื่อมต่อ คอยทำหน้าที่ควบคุม NAT Routing สำหรับเครื่องภายในในการออกอินเทอร์เน็ต และติดตั้งระบบ WireGuard VPN Server ไว้ดูแลการเชื่อมต่อเข้ามาจัดการระบบ
- **การเข้าถึงระบบที่ปลอดภัย (Secure Access)**: การทำงานและเข้าถึงพอร์ตบริการภายใน เช่น SSH ของทุกเครื่อง หรือหน้าเว็บไซต์ของ Web Server จำเป็นต้องเชื่อมผ่านเครือข่ายที่เข้ารหัสลับ (Encrypted WireGuard VPN) ก่อนเท่านั้น
- **ความปลอดภัยของฐานข้อมูล (Database Hardening)**: PostgreSQL Database บนเครื่อง db ถูกกำหนด Firewall (UFW) อย่างเข้มงวด โดยจำกัดสิทธิ์ให้รับเฉพาะคำขอจากเครื่อง Web Server (10.0.0.11) เท่านั้น
- **ระบบเฝ้าระวังเซิร์ฟเวอร์ (Centralized Server Monitoring)**: ทุกเซิร์ฟเวอร์จำลองจะติดตั้ง Node Exporter เพื่อดึงค่าสถิติการใช้งานฮาร์ดแวร์ โดยมี Prometheus Server บน Gateway คอยดึงข้อมูล (Scrape) ไปเก็บ และใช้ Grafana วาดแผนภูมิแดชบอร์ดแสดงผลเรียลไทม์

---

##  เทคโนโลยีและซอฟต์แวร์ที่เลือกใช้ (Tech Stack & Tools)

- **Virtualization**: Oracle VirtualBox (จำลองระบบปฏิบัติการ)
- **Infrastructure as Code (IaC)**: Vagrant (จัดการโครงสร้าง VMs ด้วยไฟล์โค้ด)
- **Configuration Management**: Ansible (ติดตั้งและจัดการระบบแบบอัตโนมัติ)
- **VPN / Secure Tunnel**: WireGuard (สร้างทราฟฟิก VPN เข้ารหัสเสถียรภาพสูง)
- **Server Monitoring**: Prometheus (เครื่องมือจัดเก็บข้อมูลเมทริกซ์) & Grafana (แสดงผลข้อมูลบอร์ดสถิติ)
- **Services**: Nginx (Web Server) และ PostgreSQL (Database Server)
- **Security & Networking**: Linux `iptables` (สำหรับทำ NAT/IP Masquerading) และ `ufw` (จัดการกฎ Firewall ขาเข้าและออก)

---

##  ขั้นตอนการติดตั้งและรันระบบ (Deployment Guide)

### 1. สิ่งที่ต้องเตรียมบนคอมพิวเตอร์หลัก (Prerequisites)
กรุณาตรวจสอบว่าคอมพิวเตอร์หลักของคุณได้ลงโปรแกรมเหล่านี้เรียบร้อยแล้ว:
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (เวอร์ชัน 6.1 ขึ้นไป)
- [Vagrant](https://developer.hashicorp.com/vagrant/install) (เวอร์ชัน 2.2 ขึ้นไป)
- [WireGuard Client สำหรับ Windows](https://www.wireguard.com/install/)

### 2. ทำการเปิดใช้งานเครื่องจำลองทั้งหมด (Deploy Infrastructure)
เปิด **Git Bash** ในโฟลเดอร์ของโปรเจกต์นี้ จากนั้นรันคำสั่ง:
```bash
# สั่งสร้างและตั้งค่า VMs ทั้ง 3 เครื่อง (gateway, web, db) แบบอัตโนมัติ
vagrant up
```
*หมายเหตุ: ในระหว่างกระบวนการบูตเครื่องจำลอง Vagrant จะติดตั้ง Ansible ลงในเครื่อง `gateway` โดยอัตโนมัติเพื่อนำเพลย์บุ๊กไปตั้งค่าเครื่องปลายทางในวงแลน*

### 3. ตั้งค่าระบบ VPN
1. หลังจากรันคำสั่งเสร็จเรียบร้อย จะเกิดไฟล์ `wg0-client.conf` ขึ้นมาในโฟลเดอร์ราก (Project Root) ของโปรเจกต์นี้
2. เปิดโปรแกรม **WireGuard Client** บนเครื่องคอมพิวเตอร์หลัก (Windows)
3. กดปุ่ม **Add Tunnel** ➔ เลือก **Import tunnel(s) from file...** จากนั้นเลือกไฟล์ `wg0-client.conf` 
4. กดปุ่ม **Activate** เพื่อเชื่อมเข้าหาเครือข่ายของเซิร์ฟเวอร์อย่างปลอดภัย

---

## 🧪 การทดสอบและตรวจสอบระบบ (Verification & Testing)

หลังจากเชื่อม VPN เรียบร้อยแล้ว สามารถทดสอบระบบได้ดังนี้:

### 1. การเข้าดูเว็บไซต์หน้าแรก
เปิดเว็บเบราว์เซอร์แล้วไปที่ไอพีภายในของเครื่อง Web Server:
```text
http://10.0.0.11
```
จะสามารถดูหน้า Resume Portfolio ที่รันผ่าน Nginx บนระบบหลังบ้านได้ทันที

### 2. เข้าตรวจสอบข้อมูลสถิติของเครื่องเซิร์ฟเวอร์ (Grafana Dashboard)
เปิดเว็บเบราว์เซอร์และเข้าไปที่หน้า Dashboard ของ Grafana ผ่านพอร์ต 3000:
```text
http://10.8.0.1:3000 หรือ http://192.168.56.10:3000
```
- ล็อกอินด้วยบัญชีเริ่มต้น: Username: `admin` / Password: `admin`
- เชื่อมต่อ Prometheus Data Source ที่ URL: `http://localhost:9090`
- ทำการ Import Dashboard ID: `1860` เพื่อรับชมสถิติการใช้งานเครื่องเซิร์ฟเวอร์จำลองทั้งหมดแบบเรียลไทม์

### 3. ตรวจสอบระบบความปลอดภัยและสิทธิ์ Firewall ของฐานข้อมูล
ทดสอบระบบ Firewall (UFW) ของฐานข้อมูลเพื่อยืนยันว่าเข้าถึงได้เฉพาะจาก Web Server:
- จาก **คอมพิวเตอร์หลัก (Windows Host)** แม้จะต่อ VPN อยู่ ให้ทดสอบสแกนพอร์ต 5432 ของเครื่อง db:
  ```bash
  nc -zv 10.0.0.12 5432
  ```
  *(การเชื่อมต่อนี้จะ Timeout/ถูกบล็อก เพราะ UFW ไม่อนุญาต)*
- จาก **เครื่อง Web Server** ลองเชื่อมต่อพอร์ต 5432 ของเครื่อง db:
  ```bash
  vagrant ssh web
  nc -zv 10.0.0.12 5432
  ```
  *(การเชื่อมต่อจะประสบความสำเร็จสำเร็จ / Connection succeeded!)*

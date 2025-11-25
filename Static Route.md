
# 🌐 สรุป Network Setup Static Route

**Network ของคุณ**

Device / Interface		IP / Subnet		Notes

OPNsense WAN	192.168.0.50/24		ต่อ vmbr0,เชื่อมWAN 192.168.0.0/24

OPNsense LAN		192.168.10.1/24		LAN network

ZeroTier (OPNsense)		172.30.105.114/16		Zerotier network

LXC container		192.168.0.3/24		ต่อ vmbr0 (WAN network)

PC (WAN)		192.168.0.62/24		ต่อ WAN network

----------

# 1️⃣ LXC container (192.168.0.3)

**ปัญหา:** LXC ไม่มี systemd / networking.service → `ifupdown`/interfaces.d route ไม่ถาวร

**วิธีแก้ถาวร:** ใช้ **rc.local**

1.  สร้างหรือแก้ไฟล์ `/etc/rc.local`
    

`#!/bin/sh -e  
ip route add 192.168.10.0/24 via 192.168.0.50
ip route add 172.30.0.0/16 via 192.168.0.50 
exit 0` 

2.  ทำให้รันได้:
    

`chmod +x /etc/rc.local` 

3.  ทดสอบ:
    

`ip route
ping 192.168.10.1
ping 172.30.105.114` 

✅ หลัง reboot route จะอยู่ถาวร

> **ทางเลือก:** ถ้าใช้ Proxmox LXC hookscript ก็สามารถเพิ่ม route ตอน start container ได้
ใน OPNsense ถ้า WAN Rule มี Reply-to เปิดอยู่ (มักตั้งเป็น “reply-to WAN gateway”) เวลา packet เข้ามา firewall จะบังคับให้ตอบกลับไปยัง gateway ที่ระบุ ทำให้ ICMP ping กลับไม่ถูกต้องหรือถูก drop

✅ วิธีแก้:

ไปที่ Firewall → Rules → WAN

เลือก Rule ที่อนุญาต ICMP / Ping

แก้ไข และ ติ๊กออก Reply-to (ปล่อยว่าง)

บันทึก & Apply

ทดสอบ ping อีกครั้งจาก WAN → LAN

หลังจากนี้ ping จะผ่านทันที เพราะ packet จะเดินทางตาม routing table ปกติ ไม่ถูกบังคับไป gateway เดิม

https://www.reddit.com/r/OPNsenseFirewall/comments/15o6doq/pinging_from_wan_interface/
----------

# 2️⃣ PC ฝั่ง WAN (192.168.0.62)

**Windows PC**

1.  เปิด Command Prompt **Admin**
    
2.  เพิ่ม route ถาวร:
    

`route -p add 192.168.10.0 mask 255.255.255.0 192.168.0.50
route -p add 172.30.0.0 mask 255.255.0.0 192.168.0.50` 

3.  ตรวจสอบ:
    

`route print
ping 192.168.10.1
ping 172.30.105.114` 

✅ Route จะอยู่ถาวรหลัง reboot

----------

# 3️⃣ Firewall / OPNsense

**ต้องเปิดให้ WAN → LAN / ZeroTier ผ่านได้**

1.  Firewall → Rules → WAN
    

-   Protocol: ICMP (หรือ Any)
    
-   Source: 192.168.0.0/24 (WAN net)
    
-   Destination: 192.168.10.0/24 (LAN) + 172.30.0.0/16 (ZeroTier)
    
-   Action: Pass
    

2.  Firewall → Rules → ZeroTier interface (ZT)
    

-   Source: 192.168.0.0/24
    
-   Destination: 172.30.0.0/16
    
-   Action: Pass
    

> ตรวจสอบใน Zerotier: Allow Managed Routes / Allow Global / Allow Default ต้องเปิด

----------

# 4️⃣ Flow การทำงาน

`[PC WAN / LXC] --> 192.168.0.50 (OPNsense WAN) |--> LAN 192.168.10.0/24 |--> ZeroTier 172.30.0.0/16` 

-   Route บน PC/LXC → ชี้ไป OPNsense WAN
    
-   OPNsense → ส่งต่อไป LAN หรือ ZeroTier
    
-   Firewall เปิดให้ผ่าน → ping / traffic ผ่านได้
    

----------

# 5️⃣ สรุปข้อสำคัญ

1.  **LXC:** ต้องใช้ rc.local หรือ hookscript → route ถาวร
    
2.  **Windows PC:** `route -p add ...`
    
3.  **Firewall:** WAN → LAN / ZT ต้องเปิด
    
4.  **Zerotier:** เปิด Allow Managed Routes / Allow Default / Allow Global
    

หลังทำครบ → **PC และ LXC สามารถ ping LAN และ ZeroTier ผ่าน OPNsense ได้แบบถาวร** แม้ reboot ทุกเครื่อง

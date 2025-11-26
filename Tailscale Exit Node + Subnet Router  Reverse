
**เป้าหมาย** ให้เครื่อง Linux ที่อยู่นอกวง LAN (192.168.0.x) สามารถ ping / SSH / RDP เข้าได้ทุก device ใน Tailscale network (100.x.x.x) + ทุก subnet หลัง OPNsense โดยไม่ต้องติดตั้ง Tailscale บนเครื่องนั้นเลย

**สภาพแวดล้อมของผม**

-   Proxmox 9.x
-   OPNsense VM → WAN = 192.168.0.x
-   Linux ที่อยู่นอกวง → 192.168.0.150 (ต่อ router หลัก 192.168.0.1 ปกติ)
-   Tailscale บน OPNsense (ใช้ os-tailscale plugin)

**วิธีทำ (แค่ 4 ขั้นตอน ครอบทุกอย่าง ไม่ต้องเพิ่ม rule ใหม่แม้เพิ่ม subnet ใหม่ในอนาคต)**

1.  Linux ด้านนอก (192.168.0.150) – Static route แบบโหด
    
    
    ```
    sudo ip route add 0.0.0.0/0 via 192.168.0.66        # โหดสุด ทุกอย่างโยนเข้า OPNsense
    # หรือแบบปลอดภัยกว่า
    sudo ip route add 10.0.0.0/8      via 192.168.0.66
    sudo ip route add 172.16.0.0/12   via 192.168.0.66
    sudo ip route add 192.168.0.0/16  via 192.168.0.66
    sudo ip route add 100.64.0.0/10   via 192.168.0.66 #sub tailsacle
    ```
    
2.  OPNsense – Firewall Rule บน WAN (แค่ 1 rule)
    
    text
    
    ```
    Action      : Pass
    Interface   : WAN
    Source      : 192.168.0.150 (หรือ 192.168.0.0/24)
    Destination : any
    Description : Allow external Linux → everything
    ```
    
3.  OPNsense – Outbound NAT บน TAILSCALE interface (สำคัญที่สุด! ขาดอันนี้ reply ไม่กลับ) ไป NAT → Outbound → เปลี่ยนเป็น Manual mode เพิ่ม rule ด้านบนสุด
    

    
    ```
    Interface   : TAILSCALE
    Source      : 192.168.0.150 (หรือ 192.168.0.0/24)
    Destination : any
    Translation : Interface address
    Static Port : ✓ (ติ๊ก)
    Description : NAT return traffic for external Linux
    ```
    
4.  Tailscale Admin Console คลิกเครื่อง OPNsense → Edit route settings → ติ๊กทุก subnet ที่ต้องการ (192.168.50.0/24, 10.x.x.x ฯลฯ) → Save & Approve routes

**ผลลัพธ์** จาก 192.168.0.150 (ไม่มี Tailscale ติดตั้ง)


```
ping 100.85.22.107      → ตอบ
ping 192.168.50.20      → ตอบ
ssh root@100.85.22.107  → เข้าได้เลย
```

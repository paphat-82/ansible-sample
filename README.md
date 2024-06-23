# ansible sample

## ใช้ ansible สำหรับ Update DNS Server ด้วย netplan

1. ทำกการติดตั้ง ansible ลงบนเครื่องต้นทางที่ต้องการใช้งาน ansible
2. สร้าง ssh key สำหรับ remote ไปที่เครื่องปลายทาง
```sh
ssh-keygen -f ~/.ssh/ansible -t ed25519
```
3. แลกคีย์ระหว่างเครื่องตันทางกลับปลายทาง โดยให้ใช้คำสั่ง ssh-copy-id ที่เครื่องที่ติดตั้ง ansible
```sh
ssh-copy-id -i ~/.ssh/ansible.pub username@192.x.x.x
```
4. ทดสอบ remote ssh เพื่อดูว่าสามารถ remote ไปได้ โดยที่ไม่ถาม password หรือเปล่า
```sh
ansible app -m ping -i hosts
```
5. สร้างไฟล์ ansible.cfg

**ansible.cfg :** เป็นไฟล์ที่ใช้สำหรับ configure ของ ansible

**inventory :** ชี้ไฟล์ที่เก็บข้อมูลของเครื่องปลายทาง

**remote_user :** user ที่เครื่องปลายทาง

**private_key_file :** Path ที่เก็บไฟล์ PrivateKey

```sh
[defaults]
inventory = hosts
remote_user = adminja
private_key_file = /root/.ssh/ansible
host_key_checking = False
```
6. สร้างไฟล์ hosts

ใช้สำหรับเก็บ IP Address เครื่องที่ต้องการ remote ซึ่งสามารถแบ่งเป็น group ได้ โดยการ ใช้เป็น [group_name] จากนั้นใส่ IP Address เครื่องด้านล่าง

```sh
[app]
192.x.x.x
[db]
192.x.x.x
```
7. ทดสอบ connect ไปยังเครื่องปลายทาง ด้วยคำสั่ง ansible
- ทดสอบ ping ทั้งหมด
```sh
ansible all -m ping -i hosts
```

- ทดสอบ ping group ที่เป็น db
```sh
ansible db -m ping -i hosts
```

- ทดสอบ ping group ที่เป็น app
```sh
ansible app -m ping -i hosts
```

![image](https://github.com/mothkim/ansible-sample/assets/105619969/4c4693ec-223b-4ed8-8e6b-083ad64f5be7)


8. สร้างไฟล์ netplan สำหรับใช้ในการ apply ที่เครื่องที่ติดตั้ง ansible
```sh
mkdir files
vi files/netplan-test.yml
```
จากนั้นใส่ข้อมูลด้านล่างนี้ เข้าไปไฟล์ files/netplan-test.yml (*ในไฟล์ จะเพิ่ม nameservers 1.1.1.1 เข้าไป เพื่อทดสอบ)
```sh
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.0.0.2/24
      nameservers:
        addresses: [192.0.0.1, 8.8.8.8, 1.1.1.1]
      gateway4: 192.0.0.1
  version: 2
```

9. สร้างไฟลฺ playbook สำหรับเพิ่ม config nameserver ที่ netplan
```sh
vi playbook-netplan-apply.yml
```

ใส่ค่าด้านล่างเข้าไปที่ไฟล์ playbook-netplan-apply.yml
```sh
---
- hosts: app
  tasks:
    - name: Copy Configure netplan to destination
      template:
        src: files/netplan-test.yml
        dest: /tmp/netplan-test.yml
    - name: netplan apply
      ansible.builtin.shell: |
        sudo mv /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.$(date +'%d%b%Y_%H%M').bak
        sudo mv /tmp/netplan-test.yml /etc/netplan/00-installer-config.yaml
        sudo netplan apply
    - name: checking
      ansible.builtin.shell: |
        ip a
```

## อธิบาย script : playbook-netplan-apply.yml
```
---
- hosts: app <-- เลือก remote ไปที่เครื่องที่อยู่ใน group app
  tasks: <-- task งานที่ต้องการทำ เป็น Synctax default ที่ต้องมีอยู่แล้ว
    - name: Copy Configure netplan to destination <-- ชื่อ task
      template: <-- เลือกใช้ template ที่ของ ansible เพื่อ copy ไฟล์
        src: files/netplan-test.yml <-- ไฟล์ต้นทางที่อยู่ที่เครื่อง ansible
        dest: /tmp/netplan-test.yml <-- path ของเครื่องปลายทาง ที่ต้องการวางไฟล์
    - name: netplan apply <-- ชื่อ task
      ansible.builtin.shell: |  <-- ใช้ shell script ที่เป็น builtin ของ ansible
        sudo mv /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.$(date +'%d%b%Y_%H%M').bak <-- ทำการ backup ไฟล์เดิม โดยใช้คำสั่ง mv
        sudo mv /tmp/netplan-test.yml /etc/netplan/00-installer-config.yaml <-- move ไฟล์ที่ copy จากเครื่อง ansible ไปไว้ที่ /etc/netplan/00-installer-config.yaml
        sudo netplan apply <-- รันคำสั่งเพื่อ apply config
    - name: checking
      ansible.builtin.shell: |  <-- ใช้ shell script ที่เป็น builtin ของ ansible
        netplan get <-- check DNS Server ที่เพิ่มไป
```

**ก่อนรัน ควรเช็คให้ดีว่า user ที่เราต้องรันที่เครื่องปลายทาง สามารถรันคำสั่งได้ โดยที่ไม่ติด permission กรณีที่ รันคำสั่งที่เกี่ยวกับ system อาจจะต้องไปเพิ่ม sudo ให้กับ user ที่จะใช้รันที่เครื่องปลายทางก่อน**

10. รันคำสั่งเพื่อ apply configure
```sh
ansible-playbook playbook-netplan-apply.yml
```

กรณีที่ต้องการเห็นรายละเอียดการทำงานของ ansible สามารถใส่ option -v เข้าไปในคำสั่งได้
```sh
ansible-playbook -v playbook-netplan-apply.yml
```

![image](https://github.com/mothkim/ansible-sample/assets/105619969/521d74cc-d334-4663-ba5d-ebd59aa9259d)

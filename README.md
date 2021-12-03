# Ansible Window Splunk Universal Forwarder Integration
### 1. เงื่อนไขพื้นฐานในการใช้ Ansible กับ Window Server
ในการ Setup Ansible ให้สามารถทำงานกับ Window นั้นมีความจำเป็นมากกว่า Linux เพิ่มเติมนิดหน่อยคือปกติแล้ว Window Server นั้นไม่ได้ใช้งาน SSH Services แต่ใช้เป็นการ Remote ผ่าน WinRM Protocol แทนซึ่ง WinRM นั้นจำเป็นที่จะต้องทำงานผ่าน Port 5985 หรือ 5986
[อ้างอิง WinRM Port](https://docs.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management#windows-firewall-and-winrm-20-ports)
![](assets/winrm.png)

ซึ่ง Ansible มีความจำเป็นที่ใช้เวอร์ชั่นอัพเดทล่าสุดของ Powershell เพิ่มเติมดังนั้นแล้วหากเพื่อนคนไหนลองใช้ Ansible ต่อเข้าไปแล้วเกิด Error ตลอดก็ให้นำ Script Powershell นี้ไปรันไว้ก่อนที่เครื่องนั้น โดยใน git proejct นี้เราได้นำไฟล์ที่จำเป็นในการรันไว้ให้แล้วชื่อไฟล์ว่า `enable-ansible-to-window.ps1` ให้ Copy ไปวางที่เครื่อง Window Server ปลายทางและสั่งรัน Powershell script นี้ผ่าน Powershell Terminal

[อ้างอิง Script ที่จำเป็นในการอัพเดท Powershell Module ให้รองรับ Ansible](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#winrm-setup)
![](assets/update-shell.png)

แต่ Config และ Playbook ตัวนี้ยังไม่ได้มีการใช้เทคนิคจัดการเรื่องของ Credentials อย่างเต็มรูปแบบถ้าหากนำไปใช้เพื่อนๆอย่างลืมแก้ไข Config ให้เหมาะสมกับ Environment ในงานของตัวเองด้วยนะ ~

---

### 2. เตรียม Config และ Variable ที่จำเป็นสำหรับคนรัน Ansible Engine ผ่านคอมพิวเตอร์ตัวเอง

Ansible Engine ที่เป็นตัวสั่งควบคุมเครื่องปลายทางให้ทำงานตาม Playbook ที่เขียนไว้นั้นจำเป็นที่จะต้องใช้ OS ประเภท Linux, Unix, Solaris ที่รองรับดังนี้ ซึ่งถึงแม้เราจะไม่สามารถลงตัว Ansible Engine โดยตรงที่ Window ได้แต่เราสามารถลงใน Window WSL2 แล้วใช้งานได้เหมือนกันเลย ~ 

![](assets/support.png)

เมื่อเราลง Ansible Engine เรียบร้อยแล้วให้ทดสอบด้วยการพิมพ์คำสั่ง `ansible --version ` เราควรที่จะเห็นผลลัพธ์คล้ายๆดั่งนี้
```
ansible [core 2.11.6] 
  config file = /Users/supakorn.t/.ansible.cfg
  configured module search path = ['/Users/supakorn.t/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/Cellar/ansible/4.8.0/libexec/lib/python3.10/site-packages/ansible
  ansible collection location = /Users/supakorn.t/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.10.0 (default, Oct 22 2021, 13:39:57) [Clang 13.0.0 (clang-1300.0.29.3)]
  jinja version = 3.0.2
  libyaml = True
```
#### 2.1 เพิ่ม Host และ Credentials ที่ใช้ในการต่อไป Window Server ปลายทาง

เนื่องจาก Ansible นั้นมีความยืดหยุ่นในเรื่องของการใช้ตัวแปรได้หลายระดับมากทั้งตั้งแต่การ Hardcode ลงไปใน Playbook จนถึงสร้างตัวแปร High Level ขึ้นมาเพื่อให้เกิดความยืดหยุ่นเช่นตัวแปรใช้ตาม Group ของ Server ที่เราตั้งชื่อกลุ่มไว้ก็ทำได้เช่นกัน[อ้างอิงลำดับการตัวแปรของ Ansible](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)

ดังนั้นตัวอย่างในการ Setup นี้เราจะค่อยๆเริ่มจากการตั้งตัวแปรเพื่อต่อไปยัง Host ของ Window Server ที่ถูกจัดกลุ่มรวมกันแล้วตั้งชื่อกลุ่มของเครื่องเหล่านี้ว่า `Window`

##### ไฟล์และโฟล์เดอร์ที่จำเป็นในเครื่องคนรัน Ansible Engine

1) `~/.ansible.cfg` สำหรับการระบุตัวแปร Ansible Runtime ต่างๆโดย Inventory คือการระบุตำแหน่งของ Host ไฟล์ที่จะต้องใช้ก็ให้เพื่อนๆแก้ไปตำแหน่งที่ถูกต้องของคอมพิวเตอร์ตัวเองนะ
```
[defaults]
inventory = /Users/supakorn.t/ansible/hosts
```

2) `~/ansible/hosts` ระบุเครื่องปลายทางเช่น IP หรือ Domain โดยสัญลักษณ์ปีกกาเหลี่ยมคือชื่อกลุ่มนั่นเอง `[ชื่อกลุ่ม]`
```
[window]
119.xx.xx.xx
119.xx.xx.xx
```
3) `~/ansible/group_vars/window.yaml`  ชื่อไฟล์ใน directory นี้จะต้องตรงกับชื่อ `[ชื่อกลุ่ม]`กลุ่มในไฟล์ `~/ansible/hosts` 
```
ansible_user: <<User เครื่องปลายทาง ตัวอย่างนี้ใช้ Administrator>>

ansible_password: <<รหัสผ่านเครื่องแอดมินหรือผู้ใช้คนนั้น>>

ansible_connection: winrm

ansible_winrm_server_cert_validation: ignore

ansible_winrm_transport: basic
```
ระบุตัวแปรที่ใช้ในกลุ่มของเครื่อง Host ที่ถูกจัดกลุ่มแล้ว
เวลาที่ Ansible ทำงานนั้นจะทำการมาอ่านไฟล์ `ansible.cfg` โดยสำหรับโปรเจคนี้เราจะมีการ Setup ดั่งนี้แต่เพื่อนๆก็สามารถทดลองไปปรับเปลี่ยนได้นะโดยดูได้จากลิ้งค์ที่แปะไว้
[อ้างอิงการตั้งค่า ansible.cfg](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings)

โดยปกติแล้วเราอาจจะไม่มี File หรือ Directory ที่จำเป็นต้องใช้ทั้งสามตัว ดังนั้นให้เพื่อนๆสร้างขึ้นมาตามลำดับโดยข้างในจะมี Config ดั่งนี้ (อย่าลืมว่าตอนนี้เรากำลังทำใน WSL2 ของ Window นะแต่ถ้าใครใช้ Linux อยู่แล้วก็ไม่มีปัญหาทำต่อไปได้เบย)


#### 2.2 การทำตัวแปรสำหรับ Playbook เพื่อให้เกิดความหยืดหยุ่น
ตัวอย่างนี้มีการใช้เทคนิคส่ง Variable ลงไปเพื่อให้เราไม่จำเป็นต้องไปแก้ Source Code ของ Playbook ทุกครั้งซึ่งจะมี Structure ของไฟล์ที่จำเป็นต้องแก้ไขดั่งนี้

1. `install-file/splunkclouduf.spl` คือไฟล์ License ของ Splunk ให้เราไปขอทดลองโหลด License มาวางไว้ที่ตำแหน่งนี้ **วิธีนี้ไม่ได้ปลอดภัยแต่อย่างใด** เพราะไฟล์ License ถ้าสำคัญมากๆเรอาจจะควรที่จะแยกไปเก็บไว้ใน Storage พิเศษที่ต้องใช้ token หรือการ Authentication ในการ Download ไฟล์มาติดตั้งเพื่อไม่ให้ทุกคนสามารถเอา License ไปใช้ตามชอบได้นะ 
2. `automation/roles/install_va_tool/defaults/main.yml`
```
splunk_license_location: <ตำแหน่งกับชื่อไฟล์ License | install-file/splunkclouduf.spl>
splunk_server: "yyyyy.splunkcloud.com:8089"
splunk_username: admin
splunk_password: <รหัสผ่าน>
application_installer_target_dest: "<ตำแหน่งของ License ไฟล์ที่จะไปวางในเครื่องปลายทาง | C:/Users/Administrator>"
```
ในการทดสอบการรันโปรเจคถ้าหากใครใช้ WSL2 หรือ Linux อยู่แล้วให้สั่งรันได้เลยแต่ถ้าเกิด Crash ขึ้นให้ทำการ export ตัวแปรนี้ลงไปใน Terminal โดยเฉพาะ MacOS ตัวอย่าง Error ที่จะเกิดขึ้น
```
redirecting (type: modules) ansible.builtin.setup to ansible.windows.setup
objc[20651]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called.
objc[20651]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
```
[อ้างอิง multiple_parallel_winrm_connections_crashes_winrm](https://www.reddit.com/r/ansible/comments/7768c5/multiple_parallel_winrm_connections_crashes_winrm/)

```
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
````
ทดสอบการเชื่อมต่อไปยังเครื่องปลายทางด้วยคำสั่ง
```
ansible-playbook -vv send-connection.yml
```
เราจะพบ Error นี้ก็เพราะว่าใน file Tasks `automation/roles/install_va_tool/tasks/debug-connection.yml` ที่ถูกเรียกจาก Playbook `send-connection.yml` นั้นต้องการตัวแปรทดสอบสามตัวคือ `email_address`  `splunk_password` `is_generate_ssl` ตามลำดับ
```
 FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'email_address' is undefined\n\nThe error appears to be in '/Users/supakorn.t/ProjectCode/ansible-window-splunk/automation/roles/install_va_tool/tasks/debug-connection.yml': line 10, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: print Survey Variable\n  ^ here\n"}
```
![](assets/variable.png)

แก้ไขให้ถูกต้องโดยการเติม argument `-e` เพื่อแนบ Variables ระหว่าง Ansible Engine กำลังทำงาน
```
ansible-playbook -vv send-connection.yml  -e "email_address=supakorn.t@ibm.com" -e "splunk_password=justtest" -e "is_generate_ssl=true"
```
โดยถ้าเราลองไปสังเกตที่ไฟล์ `send-connection.yml` เราจะพบว่ามี section key ของ yaml ที่ชื่อว่า `hosts` คือการบอกว่า Playbook ตัวนี้จะรันไปใส่ Host หรือ Group ใดบ้างวึ่งถ้าใช้ all ก็คือรันไปใส่เครื่อง Host ปลายทุกๆตัวจาก Inventory ทีเราได้ตั้งค่าไปในขั้นตอนแรก
ซึ่งเพื่อนๆสามารถไปดูอ้างอิงเพิ่มเติมได้จากลิ้งค์
[อ้างอิง Playbook การเลือก Host](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)
และภายใน tasks เองก็ยัง keys อย่างเช่น `win_shell` `debug` `win_whoami` ซึ่งทั้งหมดนี้จะเรียกว่า Ansible Module ซึ่งเป็น ํTask สำเร็จรูปที่ทำให้เราสามารถใช้คำสั่งแบบ High Level ได้โดยไม่ต้องไปพิมพ์ raw command Powershell จริงๆนั่นเอง แต่ในบางกรณีอย่างเช่นการที่เราต้องสามารถใช้คำสั่งโดยตรงจาก Powershell ที่ไม่มีมาจาก Module default หรือเป็น Binary พิเศษอย่างที่เราเซ็ทผ่าน PATH Environment การนำ module อย่าง `win_shell` มาใช้ก็จะทำให้เราสามารถออกคำสั่งได้โดยตรงนั่นเอง
แต่ถ้าเป็นงานที่มีพร้อมอยู่แล้วการใช้ Module ที่มาพร้อมก็จะช่วยให้เราลดปัญหาเรื่องการเช็คสถานะต่างๆไปได้อย่างเช่นการ Copy ไฟล์ที่ตัดสินใจว่าเราควรจะเลือกไฟล์ทำซ้ำหรือทำเป็น Backup หลายๆเวอร์ชันนั่นเองซึ่งเราสามารถไปดู Module Window ทั้งหมดได้ที่
https://docs.ansible.com/ansible/2.9/modules/list_of_windows_modules.html

### 3. ติดตั้ง Splunk แบบ Automation ผ่าน Choco Package
```
ansible-playbook -vv install-splunk.yml
```
Playbook `install-splunk.yml` จะไปเรียก tasks ที่อยู่ใน `automation/roles/install_va_tool/tasks/main.yml`
ซึ่งทุกๆตัวแปรสามารถใช้สัยลักษณ์ Double Curly Braclet `{{ชื่อตัวแปร}}` เพื่อแก้ไขเพิ่มเติมให้ยืดหยุ่นกว่านี้ได้เลยนะถ้าเพื่อนรันแล้วติดปัญหาต่างกันไป
```
- name: Copy Splunk License # การตั้งชื่อ Task
  win_copy: # เรียกใช้ Module จาก Ansible
   src: "{{ splunk_license_location }}"
   dest: "{{ application_installer_target_dest }}/splunkclouduf.spl"
   force: Yes

- name: Uninstall splunk-universalforwarder
  win_chocolatey: # ลบ Splunk ออกหากเคยติดตั้งอยู่แล้ว
    name: splunk-universalforwarder
    state: absent
    allow_multiple: false

- name: Install splunk-universalforwarder
  win_chocolatey: # ลง Splunk ใหม่
    name: splunk-universalforwarder
    version: '8.2.3'
    state: present # ส่งตัวแปรไปยัง install_args แทนการกด GUI
    install_args: "DEPLOYMENT_SERVER={{splunk_server}} SPLUNKUSERNAME={{splunk_username}} SPLUNKPASSWORD={{splunk_password}} WINEVENTLOG_APP_ENABLE=1 WINEVENTLOG_SEC_ENABLE=1 WINEVENTLOG_SYS_ENABLE=1 WINEVENTLOG_FWD_ENABLE=1 WINEVENTLOG_SET_ENABLE=1"
    force: No
    allow_multiple: false

- name: Ensure that C:\Program Files\SplunkUniversalForwarder\bin is not on the current user's CLASSPATH
  win_path: # เพิ่ม Environment Variable ไปยัง path เพื่อให้ Powershell รันคำสั่ง Splunk ผ่าน CommandLine ได้
    name: PATH 
    elements: C:\Program Files\SplunkUniversalForwarder\bin
    scope: machine
    state: present

- name: Login Splunk License 
  win_shell: splunk login -auth "{{splunk_username}}:{{splunk_password}}"
  register: result

# ติดตั้ง Splunk License ซึ่งต้องให้มั่นใจนะว่าที่ Window Server ปลายทางเรามีไฟล์ licrnse เรียบร้อยแล้ว 
# เพราะคำสั่งนี้ถูกรันภายในเครื่อง Window Server ปลายทางนะซึ่งตอนนี้ไฟล์ควรจะอยู่ที่  C:/Users/Administrator
- name: Install Splunk License 
  win_shell: splunk install app .\splunkclouduf.spl 
  register: result

# restart จึงจะทำให้ได้ผลการติดตั้ง License
- name: Restart Splunk Service 
  win_shell: splunk restart
  register: result
```
ผลลัพธ์ที่ได้ถ้าหากติดตั้งได้ถูกต้อง

![](assets/result.png)

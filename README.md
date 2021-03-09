# Install and configure IIS and mysql in windows ec2 instance
### Step-1
Setting up the windows host
- Log-in to your instance
- open power shell as admin
- copy paste the commands
```
$url = "https://raw.githubusercontent.com/jborean93/ansible-windows/master/scripts/Install-WMF3Hotfix.ps1"

$file = "$env:temp\Install-WMF3Hotfix.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)

powershell.exe -ExecutionPolicy ByPass -File $file -Verbose
```
[reference-link](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html)
### Step-2

Create inventory file

```
[windows1]
<windows_slave_ipaddress>
[windows1:vars]
ansible_ssh_user = Administrator
ansible_ssh_pass = <your_password>
ansible_connection=winrm
ansible_winrm_server_cert_validation= ignore
ansible_winrm_transport = basic
ansible_become=false
ansible_become_method=runas
```
### Step-3

Create Roles

- iis
- mysql
  
iis role is to install iis server and opening port 8080. make sure you have opened 8080 port in ec2 sercurity group.
```
    - name: Install IIS
      win_feature:
        name: "Web-Server"
        state: present
        restart: yes
        include_sub_features: yes
        include_management_tools: yes
    - name: Open site's port on firewall
      win_firewall_rule:
        name: iis port 8080 open
        enable: yes
        state: present
        localport: 8080
        action: Allow
        direction: In
        protocol: Tcp
        force: true
```

mysql is used to install and configure mysql server in the slave machine.
```
  - name: Adding required dll files
    win_copy:
      src: vcruntime140_1.dll
      dest: C:\Windows\System32\
    ignore_errors: true
  - name: Adding required dll files
    win_copy:
      src: d3dx10_35.dll
      dest: C:\Windows\System32\
    ignore_errors: true
  - name: Adding required dll files
    win_copy:
      src: msvcp140.dll
      dest: C:\Windows\System32\
    ignore_errors: true
  - name: Install Visual C +++
    win_package:
      path: https://download.microsoft.com/download/0/6/4/064F84EA-D1DB-4EAA-9A5C-CC2F0FF6A638/vc_redist.x64.exe
      product_id: '{CF2BEA3C-26EA-32F8-AA9B-331F7E34BA97}'
      arguments: '/q'
    ignore_errors: true
      
  - name: Stop MySQL Service If Already Installed
    win_service:
      name: MySQL
      start_mode: auto
      state: stopped
    ignore_errors: true

  - name: Clean Old MySQL Deployment
    win_file:
      path: C:\Server\MySQL
      state: absent
    become_method: runas
    become_user: Administrator

  - name: Create MySQL Server Directory
    win_file:
      path: C:\Server\MySQL
      state: directory

  - name: Create Temp Directory
    win_file:
      path: C:\Temp
      state: directory

  - name: Download MySQL Server Zip
    win_get_url:
      url: https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.22-winx64.zip
      dest: C:\Temp\mysql-8.0.22-winx64.zip

  - name: Extract MySQL Server Zip
    win_unzip:
      src: C:\Temp\mysql-8.0.22-winx64.zip
      dest: C:\Temp

  - name: Install MySQL Server
    win_copy:
      remote_src: yes
      src: C:\temp\mysql-8.0.22-winx64\
      dest: C:\Server\MySQL

  - name: Copy MySQL Configuration File
    win_copy:
      src: my.ini
      dest: C:\Server\MySQL\my.ini

  
  - name: Install MySQL Service
    win_command: C:\Server\MySQL\bin\mysqld.exe --install MySQL
    ignore_errors: true

  - name: Start MySQL Service
    win_service:
      name: MySQL
      start_mode: auto
  - name: Add sql env path
    win_path:
      name: PATH
      elements: C:\Server\MySQL\bin
      scope: user
      state: present

  - name: check sql-server version
    win_command: mysql --version
    register: out

  - debug: 
      var: out.stdout_lines
      
  - name: Allow MySQL Port 3306
    win_firewall_rule:
      name: MySQL_Port
      enable: yes
      state: present
      localport: 3306
      action: Allow
      direction: In
      protocol: Tcp
      force: true

```
### Step-4
Run ansible-playbook master.yaml
```
  hosts: windows3
  name: "Intalling iis server and mysql"
  gather_facts: no
  roles: 
    - iis
    - mysql

```
In master cli
```
 ansible-playbook -i inventory.ini master.yaml
```

output:

(observe the sql version debug output)
```
root@cf1fd5908a1c:~/ansible-windows$ ansible-playbook -i inventory.ini master.yaml

PLAY [Intalling iis server and mysql] *****************************************************************************************************************

TASK [iis : Install IIS] ******************************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [iis : Open site's port on firewall] *************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Adding required dll files] **************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Adding required dll files] **************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Adding required dll files] **************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Install Visual C +++] *******************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Stop MySQL Service If Already Installed] ************************************************************************************************
fatal: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]: FAILED! => {"changed": false, "exists": false, "msg": "Service 'MySQL' is not installed, need to set 'path' to create a new service"}
...ignoring

TASK [mysql : Clean Old MySQL Deployment] *************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Create MySQL Server Directory] **********************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Create Temp Directory] ******************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Download MySQL Server Zip] **************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Extract MySQL Server Zip] ***************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Install MySQL Server] *******************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Copy MySQL Configuration File] **********************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Install MySQL Service] ******************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Start MySQL Service] ********************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : Add sql env path] ***********************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

TASK [mysql : check sql-server version] ***************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]



TASK [mysql : debug] **********************************************************************************************************************************
ok: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com] => {
    "out.stdout_lines": [
        "mysql  Ver 8.0.22 for Win64 on x86_64 (MySQL Community Server - GPL)"
    ]
}


TASK [mysql : Allow MySQL Port 3306] ******************************************************************************************************************
changed: [ec2-18-191-200-89.us-east-2.compute.amazonaws.com]

PLAY RECAP ********************************************************************************************************************************************
ec2-18-191-200-89.us-east-2.compute.amazonaws.com : ok=20   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
```

### Refrences
[Windows featues in ansible](https://docs.ansible.com/ansible/latest/collections/ansible/windows/win_feature_module.html)


[Setting firewall rules](https://docs.ansible.com/ansible/latest/collections/community/windows/win_firewall_rule_module.html)

[ansible windwos modules](https://docs.ansible.com/ansible/latest/collections/ansible/windows/index.html)

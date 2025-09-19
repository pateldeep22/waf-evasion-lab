# **Project Report: WAF Rule Development and Evasion Testing Lab**

Author: Deep Patel  
Date: September 19, 2025

### **Table of Contents**

1. Introduction & Objectives  
2. Lab Environment Setup  
3. Deploying the Target Application (DVWA)  
4. Baseline Vulnerability Assessment  
5. WAF Installation and Configuration  
6. Verifying WAF Protection  
7. Week 1 Conclusion

---

### **1\. Introduction & Objectives**

This report documents the detailed process of establishing a secure lab environment for the development and testing of Web Application Firewall (WAF) rules. The project's goal is to create a hardened WAF ruleset by first building a vulnerable environment, defending it, and then simulating attacks to test its resilience.

The primary objectives for Week 1 were to execute the complete setup from a bare operating system to a fully functional, WAF-protected web application, documenting every command and configuration step along the way.

---

### **2\. Lab Environment Setup**

This section details the foundational setup of the virtual lab environment.

#### **2.1 Tools and Technologies**

* **Operating System:** Ubuntu Server  
* **Web Server:** Apache  
* **Database:** MariaDB  
* **WAF:** ModSecurity with OWASP Core Rule Set (CRS)  
* **Vulnerable Application:** Damn Vulnerable Web App (DVWA)  
* **Virtualization:** VMware Workstation Player

#### **2.2 Virtual Machine Creation and OS Installation**

A new virtual machine was created in VMware with 2GB of RAM and a 25GB virtual disk. Ubuntu Server was installed from the official ISO image. During the installation, the apache2 server package was pre-selected to streamline the web server setup.

#### **2.3 Initial Server Configuration**

Upon first boot, the system's package list was updated, and all software was upgraded to the latest versions to ensure a stable and secure base.

* **Command:**  
```bash
  sudo apt update && sudo apt upgrade \-y
```

* **Explanation:**  
  * sudo: Executes the command with superuser (administrator) privileges.  
  * apt update: Refreshes the local cache of available software packages.  
  * &&: A logical operator that runs the second command only if the first one succeeds.  
  * apt upgrade \-y: Upgrades all installed packages to their newest versions, automatically confirming any prompts with the \-y flag.

---

### **3\. Deploying the Target Application (DVWA)**

This section outlines the precise, multi-stage process of installing and configuring the DVWA test application.

#### **3.1 Installing DVWA and Dependencies**

The required dependencies (PHP, MariaDB, Git) were installed, and the DVWA source code was cloned from its GitHub repository into the web server's root directory (/var/www/html).

* **Commands:** 
```bash 
  sudo apt install git php php-mysqli php-gd mariadb-server \-y  
  cd /var/www/html  
  sudo rm index.html  
  sudo git clone \[https://github.com/digininja/DVWA.git\](https://github.com/digininja/DVWA.git) .
```

* **Explanation:**  
  * apt install ...: Installs all specified packages in a single command.  
  * cd /var/www/html: Changes the current working directory to the webroot.  
  * rm index.html: Removes the default Apache placeholder page.  
  * git clone ... .: Clones the DVWA repository into the current directory (.).

#### **3.2 Configuring File Permissions**

Initial ownership and permissions of the web application files were set. These are crucial steps for both security and proper operation.

* **Commands:** 
```bash 
  sudo chown \-R www-data:www-data /var/www/html/  
  sudo chmod \-R 755 /var/www/html/
```

* **Explanation:**  
  * chown: Command to **ch**ange the **own**er of files and directories. The \-R (recursive) flag applies the ownership change to all files and subdirectories. www-data:www-data sets both the user and group owner to www-data, which is the user the Apache server runs as.  
  * chmod: Command to **ch**ange the file **mod**e (permissions). 755 is a common, secure permission setting for web directories and files. It breaks down as follows:  
    * The **Owner** (www-data) gets permission 7 (Read, Write, Execute).  
    * The **Group** (www-data) gets permission 5 (Read, Execute).  
    * **Others** (all other users) get permission 5 (Read, Execute).

#### **3.3 Configuring DVWA**

The application's configuration file was prepared by copying the provided template.

* **Command:**  
```bash
  sudo cp /var/www/html/config/config.inc.php.dist /var/www/html/config/config.inc.php
```

* **Action:** This file was then edited using sudo nano /var/www/html/config/config.inc.php to input the database credentials that would be created in the next step.

#### **3.4 Configuring the Database**

A MariaDB database was configured to store DVWA's data.

1. **Secure the installation:** The mysql\_secure\_installation script was executed to set a root password and remove insecure defaults.  
2. **Create the database and user:** A dedicated database (dvwa) and user (dvwa\_user) were created for the application to prevent using the high-privilege root account.  
   * **Commands (executed inside the MariaDB prompt):**  
```bash
     CREATE DATABASE dvwa;  
     CREATE USER 'dvwa\_user'@'localhost' IDENTIFIED BY '\[Your Password\]';  
     GRANT ALL PRIVILEGES ON dvwa.\* TO 'dvwa\_user'@'localhost';  
     FLUSH PRIVILEGES;  
     EXIT;
```

#### **3.5 Browser Check and Initial Troubleshooting**

Upon navigating to the server's IP address in the host browser, the DVWA setup page was displayed. However, the "Setup Check" indicated critical errors: two key directories were not writable by the web server.

* **Error Identified:**  
  * Writable folder /var/www/html/hackable/uploads/: No  
  * Writable folder /var/www/html/config/: No  
* **Corrective Action:** To resolve this, the permissions on these specific directories were changed to allow the web server to write to them.  
  * **Commands:**  
```bash
    sudo chmod \-R 777 /var/www/html/hackable/uploads/  
    sudo chmod \-R 777 /var/www/html/config/
```

  * **Explanation:** chmod 777 grants read, write, and execute permissions to all users. While insecure for a production server, it is an acceptable and necessary step for this isolated lab environment to allow the application to function correctly.  
* **Finalization:** After running the commands and refreshing the browser, the errors turned green. The "Create / Reset Database" button was clicked, which successfully initialized the application and redirected to the login page.

---

### **4\. Baseline Vulnerability Assessment**

Before installing the WAF, baseline attacks were performed to confirm the application was vulnerable. The DVWA security level was set to **Low** for these tests.

#### **4.1 SQL Injection (SQLi) Test**

* **Procedure:** Navigated to the "SQL Injection" page.  
* **Payload Used:** 1' OR '1'='1  
* **Result:** The attack was **successful**. The application returned data for all users.  
* **Evidence:**  
  * InsertascreenshotofthesuccessfulSQLInjectionattackhere

#### **4.2 Stored Cross-Site Scripting (XSS) Test**

* **Procedure:** Navigated to the "Stored XSS" page.  
* **Payload Used:** \<script\>alert('XSS Test Successful')\</script\> (Name: "Test").  
* **Result:** The attack was **successful**. An alert box appeared on the screen.  
* **Evidence:**  
  * InsertascreenshotoftheXSSalertboxhere

---

### **5\. WAF Installation and Configuration**

With the baseline confirmed, the ModSecurity WAF was installed and configured.

#### **5.1 Installing the ModSecurity Module**

* **Command:**
```bash  
  sudo apt install libapache2-mod-security2 \-y
```

#### **5.2 ModSecurity Configuration**

1. **Activating the Main Configuration File:** The recommended configuration file was renamed to make it active.  
   * **Command:**
```bash
    sudo mv /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf  
```

2. **Changing the Rule Engine's Mode:** The configuration was edited to switch the firewall from a passive logging mode to an active blocking mode.  
   * **Command:**
```bash   
    sudo nano /etc/modsecurity/modsecurity.conf  
```
   * **Action:** Inside the file, SecRuleEngine DetectionOnly was changed to SecRuleEngine On.

#### **5.3 Enabling the OWASP Core Rule Set (CRS)**

The industry-standard OWASP CRS was enabled to provide a strong default set of rules.

1. **Enable Required Apache Module:**  
   * **Command:** 
```bash
   sudo a2enmod headers  
```
2. **Add Lines to Load Rules:** The Apache-ModSecurity link file was edited to include the OWASP rules.  
   * **Command:** 
```bash
   sudo nano /etc/apache2/mods-available/security2.conf  
```
   * **Action:** The following lines were added to the file:  
```bash
     IncludeOptional /usr/share/modsecurity-crs/crs-setup.conf.example  
     IncludeOptional /usr/share/modsecurity-crs/rules/\*.conf
```

---

### **6\. Verifying WAF Protection**

The final step was to test that the newly configured WAF could successfully block the previous attacks.

#### **6.1 Apache Configuration Check and Restart**

* **Commands:**  
```bash
  \# Test Apache configuration for syntax errors  
  sudo apache2ctl configtest

  \# Reload systemd configuration and restart Apache  
  sudo systemctl daemon-reload  
  sudo systemctl restart apache2
```

* **Explanation:** The configtest ensures there are no syntax errors before restarting. The daemon-reload and restart commands apply all the new configuration changes.

#### **6.2 Final Test and Result Analysis**

The SQL Injection test was performed again. This time, the attack was **unsuccessful**. The browser displayed a **"403 Forbidden"** error page. This result confirms that ModSecurity, using the OWASP CRS, correctly identified the malicious payload and blocked the request.

* **Evidence:**  
  *!["403Forbidden"](image.png)

---

### **7\. Week 1 Conclusion**

At the end of Week 1, a fully functional and secured lab environment has been established. The process from a base OS install to a WAF-protected application has been completed and documented. The WAF is operational and proven to block basic attacks using a default, industry-standard ruleset. The project is now prepared for Week 2, which will focus on analyzing logs and writing custom rules.

# MySQL Enumeration & Connection Reference

## 🔍 Scanning MySQL (Nmap)
* **Purpose**: Scan for MySQL services, version info, and run default MySQL enumeration scripts.
* **Execution Command**: 
  `sudo nmap $TARGETIP$ -sV -sC -p3306 --script mysql*`

---

## 🔌 Connecting to MySQL
* **Without a Password**: 
  `mysql -u root -h <target_ip>`
* **With a Password**: 
  `mysql -u root -p<password> -h <target_ip>`
* **Bypassing Self-Signed Certificate / TLS Errors (Older Clients)**: 
  `mysql -u <username> -p<password> -h <target_ip> --ssl=0`


| **Command**                                          | **Description**                                                                                       |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `mysql -u <user> -p<password> -h <IP address>`       | Connect to the MySQL server. There should **not** be a space between the '-p' flag, and the password. |
| `show databases;`                                    | Show all databases.                                                                                   |
| `use <database>;`                                    | Select one of the existing databases.                                                                 |
| `show tables;`                                       | Show all available tables in the selected database.                                                   |
| `show columns from <table>;`                         | Show all columns in the selected table.                                                               |
| `select * from <table>;`                             | Show everything in the desired table.                                                                 |
| `select * from <table> where <column> = "<string>";` | Search for needed `string` in the desired table.                                                      |
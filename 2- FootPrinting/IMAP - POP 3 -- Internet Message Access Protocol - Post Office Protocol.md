
## 🛠️ Key Commands & Enumeration

### 1. Network Scanning (Nmap)
* Scan standard POP3 and IMAP ports (plain and SSL) to detect service versions and capabilities:
  `sudo nmap <target_ip> -sV -p110,143,993,995 -sC`[cite: 1]

### 2. Quick Interaction (cURL)
* **List IMAP mailboxes over SSL:** 
  `curl -k 'imaps://<target_ip>' --user username:password`[cite: 1]
* **Verbose mode (for TLS version, certificate details, and banners):** 
  `curl -k 'imaps://<target_ip>' --user username:password -v`[cite: 1]

### 3. Manual Encrypted Connections (OpenSSL)
* **Connect to POP3 over SSL/TLS:** 
  `openssl s_client -connect <target_ip>:pop3s`[cite: 1]
* **Connect to IMAP over SSL/TLS:** 
  `openssl s_client -connect <target_ip>:imaps`[cite: 1]

---

## 📥 IMAP Protocol Commands
*Entered interactively after establishing a connection*[cite: 1]

* `LOGIN username password` — Authenticates the user[cite: 1].
* `LIST` — Lists all directories and mailboxes[cite: 1].
* `SELECT INBOX` — Selects/opens a mailbox to read messages[cite: 1].
* `LOGOUT` — Closes the connection with the IMAP server[cite: 1].

---

## 📤 POP3 Protocol Commands
*Entered interactively after establishing a connection*[cite: 1]

* `USER username` — Identifies the user[cite: 1].
* `PASS password` — Authenticates the user with their password[cite: 1].
* `STAT` — Requests the number of saved emails and total size[cite: 1].
* `RETR id` — Requests and retrieves a specific email by its ID[cite: 1].

## 📥 IMAP Protocol Commands

| **Command**                     | **Description**                                                                                               |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `1 LOGIN username password`     | User's login.                                                                                                 |
| `1 LIST "" *`                   | Lists all directories.                                                                                        |
| `1 CREATE "INBOX"`              | Creates a mailbox with a specified name.                                                                      |
| `1 DELETE "INBOX"`              | Deletes a mailbox.                                                                                            |
| `1 RENAME "ToRead" "Important"` | Renames a mailbox.                                                                                            |
| `1 LSUB "" *`                   | Returns a subset of names from the set of names that the User has declared as being `active` or `subscribed`. |
| `1 SELECT INBOX`                | Selects a mailbox so that messages in the mailbox can be accessed.                                            |
| `1 UNSELECT INBOX`              | Exits the selected mailbox.                                                                                   |
| `1 FETCH <ID> all`              | Retrieves data associated with a message in the mailbox.                                                      |
| `1 CLOSE`                       | Removes all messages with the `Deleted` flag set.                                                             |
| `1 LOGOUT`                      | Closes the connection with the IMAP server.                                                                   |

## 📤 POP3 Protocol Commands

| **Command**     | **Description**                                             |
| --------------- | ----------------------------------------------------------- |
| `USER username` | Identifies the user.                                        |
| `PASS password` | Authentication of the user using its password.              |
| `STAT`          | Requests the number of saved emails from the server.        |
| `LIST`          | Requests from the server the number and size of all emails. |
| `RETR id`       | Requests the server to deliver the requested email by ID.   |
| `DELE id`       | Requests the server to delete the requested email by ID.    |
| `CAPA`          | Requests the server to display the server capabilities.     |
| `RSET`          | Requests the server to reset the transmitted information.   |
| `QUIT`          | Closes the connection with the POP3 server.                 |

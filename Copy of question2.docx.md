Scenario Context: You work in a library, and in the library there are two public computers with access to a Library Management System to find which books are available. One day you decide to view the packet capture logs to find anything strange that happened in the system.

1\. What is the IP and Port of the web server? **(8 poin)**  
**IP \= 192.168.223.129**  
**Port \= 4167**

2\. To help with viewing the server packets going in and out, what's a good filter to use and why? **(8 poin)**

**HTTP filter because its an interaction between client and server side, packets are easily readable with the un encrypted http packets**

3\. Which user logged in on Sep 19, 2025 23:17:44 (GMT+7)? **(12 poin)**

"Username":"alice\_brown"

4\. What time did one of the public computers got access to admin user? **(10 poin) FLAG**

**Date: Fri, 19 Sep 2025 16:20:52 GMT**

5\. Which IP accessed the admin user? **(10 poin)**

192.168.223.1:49524

6\. How did the attacker gain access to the admin user? **(15 poin)**

SQL injection by getting the account id of 1 which is the admin and ignoring the other requirements for logging in

7\. What book did the attacker delete? **(8 poin)**

**1984**  
**By George Orwell**

8\. The attacker added a new book to the database, what was it called? **(8 poin)**

**He put back animal farm**

{"id":22,"title":"please fix your server","author":"love, W.","isbn":"41","year":6767,"description":"come on"}\]

9\. How did the attacker successfully leak all the usernames and passwords? **(15 poin)**

By using sqli 3 times, first he sees if he can do the sqli, and the he checks all the tables, and the he leaks all the users and passwords

10\. What was the password hash of admin account? **(6 poin)**  
{"id":"admin","title":"0192023a7bbd73250516f069df18b500"  

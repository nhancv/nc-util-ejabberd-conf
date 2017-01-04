# nc-util-ejabberd-conf
Download: https://www.ejabberd.im/

Protocols: https://www.process-one.net/en/ejabberd/protocols/

Installation: https://docs.ejabberd.im/admin/installation/

```
Note: username for login is <name>@<hostname>
```
Open ejabberd server to public:
https://www.ejabberd.im/node/17509

Ejabberd config file in 
```
<ejabberd installation dir>/conf/ejabberd.yml
```


Bash alias:
```bash
export PATH=${PATH}:/home/nhancao/Softs/ejabberd-16.12.beta1/bin
alias xmpp.start='/home/nhancao/Softs/ejabberd-16.12.beta1/bin/start'
alias xmpp.stop='/home/nhancao/Softs/ejabberd-16.12.beta1/bin/stop'
```
---------------------------------------------------------------
##Setup mysql database:
https://docs.ejabberd.im/admin/guide/databases/mysql/

```
echo "GRANT ALL ON ejabberd.* TO 'ejabberd'@'localhost' IDENTIFIED BY 'password';" | mysql -h localhost -u root
echo "CREATE DATABASE ejabberd;" | mysql -h localhost -u ejabberd -p
wget https://raw.githubusercontent.com/processone/ejabberd/master/sql/mysql.sql
mysql -h localhost -D ejabberd -u ejabberd -p < mysql.sql
echo "SHOW TABLES;" | mysql -h localhost -D ejabberd -u ejabberd -p --table
```

Edit ejabberd.yml
```
sql_type: mysql
sql_server: "localhost"
sql_database: "ejabberd"
sql_username: "ejabberd"
sql_password: "password"
## If you want to specify the port:
sql_port: 3306


auth_method: sql


```

Create user on sql with cli
```
ejabberdctl register "testuser" "localhost" "passw0rd"
```


##Setup rest api
https://docs.ejabberd.im/developer/ejabberd-api/

*Note with v16.12.beta1*
```
Database / Back-ends for OAuth tokens
Currently, OAuth tokens are stored in Mnesia database. In a future release, we plan to support multiple token backends.
```

###Use api with basic auth
```rest
POST http://localhost:7011/api/get_roster
Authorization: Basic YWRtaW5AbG9jYWwuYmVlc2lnaHRzb2Z0LmNvbToxMjMzMjE=
Content-Type: application/json
{}
```
where: 
```
Authorization: Basic base64(<userJid>:<password>)
```

if you want use more command, you must add it to "add_commands" section in config file


####Extra function
- Last message: 
+ Create table

```sql
--
-- Table structure for table `last_message`
--

CREATE TABLE `last_message` (
  `username` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `peer` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `txt` text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

ALTER TABLE `last_message`
  ADD PRIMARY KEY (`username`,`peer`);
```

+ Create new trigger in archive table by run sql command with content below:

```sql
-- before insert trigger

DELIMITER $$
CREATE TRIGGER `update_last_message` BEFORE INSERT ON `archive`
FOR EACH ROW 
BEGIN
	SET @NEW_PEER = SUBSTRING_INDEX(NEW.peer, '@', 1);
	SET @COUNT = (SELECT COUNT(*) FROM last_message WHERE username = NEW.username AND peer = @NEW_PEER);
    IF @COUNT = 0 THEN
        INSERT INTO last_message (username, peer, txt) VALUES (NEW.username, @NEW_PEER, NEW.txt); 
    ELSE
    	UPDATE last_message SET txt = NEW.txt WHERE (username = NEW.username AND peer = @NEW_PEER);
    END IF;
END
$$
DELIMITER ;

-- after delete trigger

DELIMITER $$
CREATE TRIGGER `delete_last_message` AFTER DELETE ON `archive`
FOR EACH ROW 
BEGIN
	SET @NEW_PEER = SUBSTRING_INDEX(OLD.peer, '@', 1);
	SET @COUNT = (SELECT COUNT(*) FROM last_message WHERE username = OLD.username AND peer = @NEW_PEER);
    IF @COUNT > 0 THEN
        DELETE FROM `last_message` WHERE (username = OLD.username AND peer = @NEW_PEER);
    END IF;
END
$$
DELIMITER ;
```

---

##Option trigger for `last_spool`

```sql
--
-- Table structure for table `last_spool`
--

CREATE TABLE `last_spool` (
  `username` varchar(191) COLLATE utf8mb4_unicode_ci NOT NULL,
  `txt` blob NOT NULL,
  `seq` bigint(20) UNSIGNED NOT NULL,
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

ALTER TABLE `last_spool`
  ADD PRIMARY KEY (`username`);


-- before insert trigger

DELIMITER $$
CREATE TRIGGER `update_last_spool` BEFORE INSERT ON `spool`
FOR EACH ROW 
BEGIN
    SET @COUNT = (SELECT COUNT(*) FROM last_spool WHERE username = NEW.username);
    IF @COUNT = 0 THEN
        INSERT INTO last_spool (username, txt, seq, created_at) VALUES (NEW.username, NEW.xml, NEW.seq, NEW.created_at); 
    ELSE
        UPDATE last_spool SET txt = NEW.xml, seq = NEW.seq, created_at = NEW.created_at WHERE (username = NEW.username);
    END IF;
END
$$
DELIMITER ;


-- after delete trigger

DELIMITER $$
CREATE TRIGGER `delete_last_spool` AFTER DELETE ON `spool`
FOR EACH ROW 
BEGIN
    SET @COUNT = (SELECT COUNT(*) FROM last_spool WHERE username = OLD.username);
    IF @COUNT > 0 THEN
        DELETE FROM `last_spool` WHERE (username = OLD.username);
    END IF;
END
$$
DELIMITER ;
```

---
##Option last message with unique peer

```sql
--
-- Table structure for table `last_message`
--

CREATE TABLE `last_message` (
  `peer1` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `peer2` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `txt` text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

ALTER TABLE `last_message`
  ADD PRIMARY KEY (`peer1`,`peer2`);


-- before insert trigger

DELIMITER $$
CREATE TRIGGER `update_last_message` BEFORE INSERT ON `archive`
FOR EACH ROW 
BEGIN
  SET @NEW_PEER = SUBSTRING_INDEX(NEW.peer, '@', 1);
  SET @COUNT = (SELECT COUNT(*) FROM last_message WHERE (peer1 = NEW.username AND peer2 = @NEW_PEER) OR (peer2 = NEW.username AND peer1 = @NEW_PEER));
    IF @COUNT = 0 THEN
        INSERT INTO last_message (peer1, peer2, txt) VALUES (NEW.username, @NEW_PEER, NEW.xml); 
    ELSE
      UPDATE last_message SET txt = NEW.xml WHERE (peer1 = NEW.username AND peer2 = @NEW_PEER) OR (peer2 = NEW.username AND peer1 = @NEW_PEER);
    END IF;
END
$$
DELIMITER ;

-- after delete trigger

DELIMITER $$
CREATE TRIGGER `delete_last_message` AFTER DELETE ON `archive`
FOR EACH ROW 
BEGIN
  SET @OLD_PEER = SUBSTRING_INDEX(OLD.peer, '@', 1);
  SET @COUNT = (SELECT COUNT(*) FROM last_message WHERE (peer1 = OLD.username AND peer2 = @OLD_PEER) OR (peer2 = OLD.username AND peer1 = @OLD_PEER));
    IF @COUNT > 0 THEN
        DELETE FROM `last_message` WHERE (peer1 = OLD.username AND peer2 = @OLD_PEER) OR (peer2 = OLD.username AND peer1 = @OLD_PEER);
    END IF;
END
$$
DELIMITER ;


```



Error:
{error,access_rules_unauthorized}
when run ejabberdctl

-> fix:
remove api_permissions block or re-edit permission







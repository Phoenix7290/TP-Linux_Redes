❯ ss -tulpen | grep LISTEN
tcp   LISTEN 0      70          127.0.0.1:33060      0.0.0.0:*    uid:105 ino:14484 sk:5 cgroup:/system.slice/mysql.service <->
                                                        tcp   LISTEN 0      1000   10.255.255.254:53         0.0.0.0:*    ino:12298 sk:1 cgroup:/ <->                                                                         
 tcp   LISTEN 0      4096       127.0.0.54:53         0.0.0.0:*    uid:991 ino:20020 sk:2001 cgroup:/system.slice/systemd-resolved.service <->
                                                         tcp   LISTEN 0      4096        127.0.0.1:27017      0.0.0.0:*    uid:108 ino:5911 sk:6 cgroup:/system.slice/mongod.service <->                                      
  tcp   LISTEN 0      151         127.0.0.1:3306       0.0.0.0:*    uid:105 ino:14486 sk:8 cgroup:/system.slice/mysql.service <->
                                                          tcp   LISTEN 0      511         127.0.0.1:37177      0.0.0.0:*    users:(("node",pid=5073,fd=23)) uid:1000 ino:21877 sk:15006 cgroup:/init.scope <->                
   tcp   LISTEN 0      200         127.0.0.1:5432       0.0.0.0:*    uid:106 ino:25694 sk:200b cgroup:/system.slice/system-postgresql.slice/postgresql@16-main.service <->
                                                           tcp   LISTEN 0      511         127.0.0.1:6379       0.0.0.0:*    uid:107 ino:4608 sk:200c cgroup:/system.slice/redis-server.service <->                           
    tcp   LISTEN 0      4096    127.0.0.53%lo:53         0.0.0.0:*    uid:991 ino:20018 sk:2002 cgroup:/system.slice/systemd-resolved.service <->
                                                            tcp   LISTEN 0      511             [::1]:6379          [::]:*    uid:107 ino:4609 sk:200e cgroup:/system.slice/redis-server.service v6only:1 <->                 
     tcp   LISTEN 0      4096                *:4000             *:*    ino:27176 sk:2012 cgroup:/ v6only:0 <-> 
                                                             tcp   LISTEN 0      4096                *:11434            *:*    ino:10609 sk:2013 cgroup:/ v6only:0 <->                                                        
      tcp   LISTEN 0      4096                *:3000             *:*    ino:22845 sk:2014 cgroup:/ v6only:0 <->
                                                              tcp   LISTEN 0      4096                *:3002             *:*    ino:2942 sk:2016 cgroup:/ v6only:0 <->                                                        
       % 
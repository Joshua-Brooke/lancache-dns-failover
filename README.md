# Lancache DNS Failover 

To function properly Lancache must be the only DNS server available to the client, this means that if the lancache server is unavailable the clients will have no DNS.  Using Nginx compiled with lua and the following configuration file this will allow the client to use lancache when it is available and failover to a backup DNS when it is not available.

This works by testing port 80 on the lancache monolithic instance which is used to serve the files.  If your lancache DNS and fileserver are on different IP addresses you will need to edit the config to reflect this.

## Usage
Install Nginx compiled with lua, the easiest way is to install [Openresty](https://openresty.org/en/installation.html).

Append the the following to your nginx.conf
```
stream {
    upstream dns {
     server 10.10.10.2:53;
       balancer_by_lua_block {
            local balancer = require "ngx.balancer"
            local socket = require "socket"

            local host = "10.10.10.2" -- Your lancache server
            local port = 53
            local timeout = 0.1

            local test_connection = socket.tcp()
            test_connection:settimeout(timeout)
            local connect_ok, connect_err = test_connection:connect(host, 80)
            test_connection:close()

            if not connect_ok then
                host = "10.10.10.3" -- Your failover server
            end

            local ok, err = balancer.set_current_peer(host, port)


       }
   }

    server {
        listen     53 udp;
        proxy_pass dns;
        }
}
```


Restart the openresty service and point your clients DNS to the openresty server.

You may need to increase the number of files or connections that Nginx can open by adjusting the worker_rlimit_nofile or worker_connections parameters.



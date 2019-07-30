# A/B Testing with Key Value Store demo

A demo showing spliting traffic and dynamicly reconfiguring the traffic distribution using NGINX Plus key-value store API, for A/B testing, Blue-Green and Canary deployments. 

## File Structure

```
etc/
└── nginx/
    ├── conf.d/
    │   ├── default.conf ...........Default Virtual Server configuration
    │   └── dummy_servers.conf......Dummy loopback web servers responds with plain/text
    │   └── upstreams.conf..........Upstream configurations
    │   └── stub_status.conf........NGINX Open Source basic status information available http://localhost/nginx_status only
    │   └── status_api.conf.........NGINX Plus Live Activity Monitoring available on port 8080
    ├── state_files/................State files are store here which store the entries so they persist across NGINX Plus restarts
    └── nginx.conf .................Main NGINX configuration file with global settings
└── ssl/
    └── nginx/
        ├── nginx-repo.crt...........NGINX Plus repository certificate file
        └── nginx-repo.key...........NGINX Plus repository key file

```

```
                                                          Client
                                                          Port 80
                                                            +
                                                            |
                                                            |
                                                 +----------v-----------+
                                                 |                      |
                                                 |                      |       API
                                                 |                      |       & Live Monitoring Dashboard
                                                 |       NGINX Plus     +<-----+Port 8080
                                                 |                      |
                                                 |                      |
                                                 |                      |
                                                 |                      |
                                                 +----------+-----------+
                                                            |
                                     +----------------------+-------------------+
                                     |                                          |
      /                              |                                          |
      /sticky (Session Persistence)  |                                          |    /stickyroute (session persistence)
                                     |                                          |
                          +----------+----------+                               |
                          |                     |                               |
                          |                     |                               |
                          |                     |                               |
                          |                     |                               |
                          |                     |                               |
                          |                     |                               |
                  +-------+---------+  +--------+--------+    +-----------------+----------------+
                  |                 |  |                 |    |                                  |
                  |    version_a    |  |    version_b    |    |   Stickyroute                    |
                  |                 |  |                 |    |                                  |
                  | 127.0.0.1:8096  |  |  127.0.0.1:8097 |    |   127.0.0.1:8096   version_a     |
                  |                 |  |                 |    |   127.0.0.1:8097   version_b     |
                  +-----------------+  +-----------------+    |                                  |
                                                              +----------------------------------+
```

## Just add Nginx Plus License file

In order to build the Docker Container and run Nginx Plus you need to copy your Nginx Plus license or Evaluation Trial License (`nginx-repo.key` and `nginx-repo.crt`) into the `/etc/ssl/nginx/` directory, where Dockerfile will look

## Build Docker container

 1. Copy and paste your `nginx-repo.crt` and `nginx-repo.key` into `etx/ssl/nginx` directory first

 2. Build an image from your Dockerfile:
    ```bash
    # Run command from the folder containing the `Dockerfile`
    $ docker build -t nginx-plus-ab-kv-demo .
    ```
 3. Start the Nginx Plus container, e.g.:
    ```bash
    # Start a new container and publish container ports 80, 443 and 8080 to the host
    $ docker run -d -p 80:80 -p 443:443 -p 8080:8080 nginx-plus-ab-kv-demo
    ```
 4. To run commands in the docker container you first need to start a bash session inside the nginx container
    ```bash
    sudo docker exec -i -t [CONTAINER ID] /bin/sh
    ```
 5. See demo instructions below


## Non-Docker Setup

 1. Copy all the contents of `./etc/nginx` to the Nginx host, into the Nginx directory, `/etc/nginx`
 2. Create the `state_files` folder by running the command `sudo mkdir -p /etc/nginx/state_files`
 3. Make sure ports `80`, `443` and `8080` are open and any necessary firewall rules are made so that these ports are accessible on the Nginx host
 4. See demo instructions below

## Demo Instructions:

  1. While testing, you can view the live monitoring API on `http://localhost:8080`

  2. This demo uses the cookie file, `cookie.txt`, and writes results into `results1.txt`, `results2.txt`, `results3.txt` and `results4.txt`. Delete these files when restarting the demo steps:

  ```bash
  rm cookie.txt results1.txt results2.txt results3.txt results4.txt
  ```

  3. See the comments in `default.conf` for a step-by-step explanation of the nginx mechanics 

### Set the Split Clients percentage via API

Our configuration allows us tocontrol how the traffic is split between the `version_a` and `version_b` upstream groups by sending an API request to NGINX Plus and setting the `$split_level` value for a hostname. For example, the following two requests can be sent to NGINX Plus so that `5%` of the traffic for `www.example.com` is sent to the `version_b` upstream group and `25%` of the traffic for `www2.example.com` is sent to the `version_b` upstream group:

The examples uses [jq](https://stedolan.github.io/jq/download/) tool (optional) for JSON formatting

 1. The following commands will set the A/B traffic distribution by setting the variable, `$split_level`, via the API.
    In this example, we will set the A/B traffic distribution for `www.example.com` and `www2.example.com`:

```bash
# Set (POST method) split traffic distribution for www.example.com to 95% VERSION-A / 5% VERSION-B
$ curl -iX POST -d '{"www.example.com":5}' http://localhost:8080/api/3/http/keyvals/split

HTTP/1.1 201 Created
Server: nginx/1.15.2
Date: Tue, 16 Oct 2018 17:46:33 GMT
Content-Length: 0
Location: http://localhost:8080/api/3/http/keyvals/split/
Connection: keep-alive

# Set (POST method) split traffic distribution for www2.example.com to 75% VERSION-A / 25% VERSION-B
$ curl -iX POST -d '{"www2.example.com":25}' http://localhost:8080/api/3/http/keyvals/split

HTTP/1.1 201 Created
Server: nginx/1.15.2
Date: Tue, 16 Oct 2018 17:55:32 GMT
Content-Length: 0
Location: http://localhost:8080/api/3/http/keyvals/split/
Connection: keep-alive

# View split client settings in the key-val store
$ curl -s http://localhost:8080/api/3/http/keyvals/split | jq
{
  "www.example.com": "5",
  "www2.example.com": "25"
}
```

### Test A/B traffic distribution of upstream groups

We can now run some load and test the the A/B traffic distribution for `www.example.com` and `www2.example.com`

#### Simulate requests and see A/B traffic distribution in action

```bash
# Run an infinite while loop that runs cURL requests and grep the `X-Test` header. Use CTRL-C to stop the loop prematurely

# Test A/B Distribution to www.example.com
$ while true; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www.example.com" http://localhost | grep "X-Test"; done

# Test A/B Distribution to www2.example.com
$ while true; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www2.example.com" http://localhost | grep "X-Test"; done

```

#### Simulate requests 100 times and count the A/B traffic distribution

The command below will simulate repeated requests. This command will run a loop 100 times and will record `X-Test` result, `X-Test: VERSION-A` or `X-Test: VERSION-B`, in the file, `result.txt`.

We then can use `grep` and `wc`, to count the number of occurrences of `X-Test: VERSION-A` and `X-Test: VERSION-B` to check our `split_client` configuration is working as expected

 1. Let's first test `www.example.com`:

```bash
# The while loop will do a cURL 100 times, grep the `X-Test` header and save that into results.txt. Allow the command to finish the loop. Use CTRL-C to stop the loop prematurely

$ for i in {1..100}; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www.example.com" http://localhost | grep "X-Test" >> results1.txt; done

# Count the number of responses from VERSION-A. We should see around 95%
$ cat results1.txt | grep -o 'VERSION-A' | wc -l
      95
# Count the number of responses from VERSION-B. We should see around 5%
$ cat results1.txt | grep -o 'VERSION-B' | wc -l
       5
```

 2. We can also repear the steps to test `www2.example.com`:

```bash
# The while loop will do a cURL 100 times, grep the `X-Test` header and save that into results.txt. Allow the command to finish the loop. Use CTRL-C to stop the loop prematurely

$ for i in {1..100}; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www2.example.com" http://localhost | grep "X-Test" >> results2.txt; done

# Count the number of responses from VERSION-A. We should see around 75%
$ cat results2.txt | grep -o 'VERSION-A' | wc -l
      75
# Count the number of responses from VERSION-B. We should see around 25%
$ cat results2.txt | grep -o 'VERSION-B' | wc -l
       25
```

### Update the A/B distribution via API

 1. The following commands will set new A/B traffic distribution values by updating the variable, `$split_level`, via the API.
    In this example, we will update the A/B traffic distribution percentage for `www.example.com` and `www2.example.com`:


```bash
# Update (PATCH method) the traffic for www.example.com to default 0%, i.e. 100% of traffic goes to VERSION-A
$ curl -iX PATCH -d '{"www.example.com":0}' http://localhost:8080/api/3/http/keyvals/split

HTTP/1.1 204 No Content
Server: nginx/1.15.2
Date: Tue, 16 Oct 2018 22:09:21 GMT
Connection: keep-alive


# Update (PATCH method) split traffic distribution for www2.example.com to 50% VERSION-A / 50% VERSION-B
$ curl -iX PATCH -d '{"www2.example.com":50}' http://localhost:8080/api/3/http/keyvals/split

HTTP/1.1 204 No Content
Server: nginx/1.15.2
Date: Tue, 16 Oct 2018 22:09:35 GMT
Connection: keep-alive


# View split client settings in the key-val store
$ curl -s http://localhost:8080/api/3/http/keyvals/split | jq
{
  "www.example.com": "0",
  "www2.example.com": "50"
}
```

#### Retest the A/B traffic distribution of upstream groups

 1. Let's test our updated A/B traffic distribution for `www.example.com`

```bash
# The while loop will do a cURL 100 times, grep the `X-Test` header and save that into results.txt. Allow the command to finish the loop. Use CTRL-C to stop the loop prematurely

$ for i in {1..100}; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www.example.com" http://localhost | grep "X-Test" >> results3.txt; done

# Count the number of responses from VERSION-A. We should see 100%
$ cat results3.txt | grep -o 'VERSION-A' | wc -l
      100
# Count the number of responses from VERSION-B. We should see 0%
$ cat results3.txt | grep -o 'VERSION-B' | wc -l
       0
```

 2. Test our new A/B traffic distribution for `www2.example.com`

```bash
# The while loop will do a cURL 100 times, grep the `X-Test` header and save that into results.txt. Allow the command to finish the loop. Use CTRL-C to stop the loop prematurely

$ for i in {1..100}; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www2.example.com" http://localhost | grep "X-Test" >> results4.txt; done

# Count the number of responses from VERSION-A. We should see 50%
$ cat results4.txt | grep -o 'VERSION-A' | wc -l
      50
# Count the number of responses from VERSION-B. We should see 50%
$ cat results4.txt | grep -o 'VERSION-B' | wc -l
      50
```

### Split Clients with Cookie persistency

 1. Lets first set the A/B traffic distribution for `www.example.com` to 50% VERSION-A / 50% VERSION-B

```bash
# Update (PATCH method) split traffic distribution for www2.example.com to 50% VERSION-A / 50% VERSION-B
$ curl -iX PATCH -d '{"www.example.com":50}' http://localhost:8080/api/3/http/keyvals/split

# View split client settings in the key-val store
$ curl -s http://localhost:8080/api/3/http/keyvals/split | jq
{
  "www.example.com": "50",
}
```

  2. Test the new 50% split traffic distribution:

```bash
# The while loop will do a cURL request every second and grep the `X-Test` header . CTRL-C to stop the loop
$ while sleep 1; do curl -s -I -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www.example.com" http://localhost/sticky | grep "X-Test"; done

X-Test: VERSION-B
X-Test: VERSION-A
X-Test: VERSION-A
X-Test: VERSION-B
X-Test: VERSION-A
X-Test: VERSION-B
```

 3. Now we will run tests, simulating a cookies enabled web browser. Our configuration for requests made to the url path, `\sticky`, will add a cookie, `split_test_version`, to the reponse and all subsequent requests with the cookie attached will result in our request to be routed ("stick") to the same backend.

In the example below note that the cookie, `split_test_version=version_a;Path=/;Max-Age=3600`; is consistent with the backend server, `X-Test: VERSION-A` and when we run the tests for a number of seconds we see all requests "stick" to servers in the `version_a` upstream, requests are not "bounced" or load balanced between `version_a` and `version_b`

```bash
# The while loop will do a cURL request every second and grep the `X-Test` header and `split_test_version` cookie value. CTRL-C to stop the loop
$ while sleep 1; do curl -b cookies.txt -c cookies.txt -I -s -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "Host: www.example.com" http://localhost/sticky | grep -E "X-Test|split_test_version"; done

X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
```

 4. To simulate "cleared cookied" you can simply delete the cookie jar, `cookies.txt`, run: `rm cookies.txt`

### Split Clients with Sticky Route persistency

 1. Run the following cURL command to simulate repeated requests from a single client remote IP address. The cURL parameters used are to set the `User-Agent` and to use a cookie jar to simluate a web browser behaviour.

In the example below note that the cookie, `sticky route` will use the route information contained in the cookie `split_test_version`.
In this example, the cookie and value, `split_test_version=version_a;Path=/;Max-Age=3600`; is routed to the server with the route, `route=version_a` and consistent with the backend server, `X-Test: VERSION-A`:

```bash
# The while loop will do a cURL request every second and grep the `X-Test` header and `split_test_version` cookie value. CTRL-C to stop the loop
$ while sleep 1; do curl -b cookies.txt -c cookies.txt -I -s -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X x.y; rv:42.0) Gecko/20100101 Firefox/44.0" -H "host: www.example.com" http://localhost/stickyroute | grep -E "X-Test|split_test_version"; done

X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
X-Test: VERSION-A
Set-Cookie: split_test_version=version_a;Path=/;Max-Age=3600;
```
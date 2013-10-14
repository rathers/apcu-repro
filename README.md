apcu-repro
==========

Script to help reproduce https://github.com/krakjoe/apcu/issues/19

How to reproduce
----------------

Place apc-bench.php under the root of your webserver and check it works:

```
curl -v http://localhost/apc-bench.php
```

It's best if you test from a different box rather than localhost. I'm just using localhost as an example.

Now you need to throw some load at the script. ApachBench (and other tools that use a fixed concurrency model) aren't great for reproducing this issue as the concurrent threads will just hang, rather than pushing further traffic at the webserver. Eseentially you wont be able to make the apache processes spiral out of control using AB. Instead you need a tool that supports fixed rate load injection. I have been using http_load. http://acme.com/software/http_load/

Download the source tarball for http_load, untar and compile it:

```
wget http://acme.com/software/http_load/http_load-12mar2006.tar.gz
tar -zxf http_load-12mar2006.tar.gz
cd http_load-12mar2006
make
```
Create a text file with the url to hit:


```
echo "http://localhost/apc-bench.php" > url_file.txt
```
Now you're ready to load the script:


```
./http_load -rate 1 -seconds 1000 url_file.txt
```

while http_load is running, monitor the number of apache children:

```
watch -n 1 "ps aux |grep "/usr/sbin/httpd" |wc -l
```

Low load rates should be OK, the webserver should cope. You'll see a few httpd processes in top using some CPU and the process monitor should show a constant (low) number (depending on the apache settings). Now try increasing the concurrency a bit at a time until you find the tipping point. You'll know you've reached the tipping point when the the number of apache procs starts increasing fast, until it hits the MaxClients limit. On a 4-core VMWare VM this occured at a rate of 4 using http_load. You can try inserting a few manual requests with curl during the time that http_load is running to see what the response time is like:

``` 
time curl http://localhost/apc-bench.php
```

You should see the response times remain consistent until you reach the tipping point, whereupon they will increase exponentially.


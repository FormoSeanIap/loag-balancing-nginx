# Using Nginx as Load Balancer with Node.js (Express)

- This article can also be found on [Medium](https://philosophyotaku.medium.com/using-nginx-as-load-balancer-with-node-js-express-63b39948f737).

# Introduction

One of the main features of Nginx is its load balancing functionality.    

In this article, I will show what it is, and how it can be done with a step-by-step in the following instruction.    

- Prerequisite
    - You need to already have Nginx installed on the Linux OS.
    - Better to have the basic idea of how config and log files work in Nginx.
    - The basic idea of Node.js (Express.js) is also required since this instruction will be done with it.

# What is Load Balancing

## Definition

- Sometimes we cannot handle tons of requests with only one server, so there might be two or more.
    
	<img src="./img/load-balancing-nginx-1.jpg" width="50%">
    
- With more than one servers, there will be a problem of how we should distribute the requests to a specific server.
    
	<img src="./img/load-balancing-nginx-2.jpg" width="50%">
    
- So we use a load balancer to solve this problem.
    
	<img src="./img/load-balancing-nginx-3.jpg" width="50%">
    
- In this case, we use Nginx as a load balancer.
- We can use different algorithm to decide which server a specific request will go to.

## Algorithm

Different services or servers have their own algorithm, and I will introduce only basic ones in Nginx.

### ****Round Robin Load Balancing****

- Definition: to distribute requests equally to different servers.
    
    <img src="./img/load-balancing-nginx-4.jpg" width="50%">

### ****Weighted Load Balancing****

- Definition: to distribute request based on the weights on server.

	<img src="./img/load-balancing-nginx-5.jpg" width="50%">

### Least Load(Connections) Balancing

- Definition: to distribute the requests to the least loaded(connections) server.

	<img src="./img/load-balancing-nginx-6.jpg" width="50%">

- There is a new request, and there are two servers. Server B is handling less requests than Server A.

	<img src="./img/load-balancing-nginx-7.jpg" width="50%">

Since Server B was handling the least requests, this request will be sent to Server B

### (IP) hash ****Load Balancing****

- Definition: the requests coming from the same IP will always be sent to a specific server.

![Without hash balancing](./img/load-balancing-nginx-8.jpg)

Without hash balancing

![With hash balancing](./img/load-balancing-nginx-9.jpg)

With hash balancing

### Health Checks

This is actually not an algorithm for load balancing.

Before sending requests to a server, Nginx will automatically check if the server is still working.

If the server is already down, Nginx will send requests to other servers, and we will be able to find an error log in Nginx. 

# Step1: Deploy four servers

The first thing is to deploy the four servers

1. Clone files from ‣
2. Do npm install
3. Run the three servers listening on 4000, 4500, 5000, and 6000.

---

Each server come with a unique response, so that we will be able to see, with load balancing, what response we will get. 

For example, the `app4000.js`  will simply respond `res on 4000`. If we get such response, we know our last request was sent to app4000.js by Nginx.

# Step2: Setting up Nginx

Noted again that you need basic ideas of how Nginx works, in order to follow.

## The figure

As illustrated in the figure, we use `proxy_pass` to send requests into `upstream` , and decide what algorithm we are going to use in `upstream.` 

![Untitled](./img/load-balancing-nginx-10.jpg)

The exact setting in Nginx comes in like this

```
upstream test {
        #ip_hash;
        #least_conn;
        server 127.0.0.1:4000;
        server 127.0.0.1:4500; #max_fails=3 fail_timeout=5s;
        server 127.0.0.1:5000; #weight=2;
        #server 127.0.0.1:6000;
 }

server {
  listen 80;
  access_log   /var/log/nginx/nginx.vhost.access.log;
  error_log    /var/log/nginx/nginx.vhost.error.log;

  location / {
          proxy_pass http://test;
  }
}
```

## ****Round Robin Load Balancing****

The default algorithm is Round Robin. 

If we set upstream like this:

```
upstream test {
        server 127.0.0.1:4000;
        server 127.0.0.1:4500; 
        server 127.0.0.1:5000;
     }
```

Since there are three servers listening on 4000, 4500, and 5000, requests will be sent to these three servers in sequence, as shown in the following video.

https://user-images.githubusercontent.com/55405280/160231407-127b5815-36b6-4f27-b076-ec8a30de7889.mp4

## Weighted Load Balancing

If we add two weights on server 5000 and send four requests:

```
upstream test {
        server 127.0.0.1:4000;
        server 127.0.0.1:4500; 
        server 127.0.0.1:5000 weight=2;
 }
```

We will see two of them coming from server 5000.

https://user-images.githubusercontent.com/55405280/160231458-c330559f-f1b4-4496-b288-b2abf3fbd923.mp4

## IP Hash Load Balancing

If we set `ip_hash`, 

```
upstream test {
        ip_hash;
        server 127.0.0.1:4000;
        server 127.0.0.1:4500; 
        server 127.0.0.1:5000;
 }
```

We will see response coming from the same server. This is because Nginx has already remembered my IP and will always send me to the same server. 

https://user-images.githubusercontent.com/55405280/160231464-95018e83-0362-401c-99ca-466954328e51.mp4

## Least Connection Load Balancing

Set `least_conn`, and add a new server listening on 6000.

```
upstream test {
        least_conn;
        server 127.0.0.1:4000;
        server 127.0.0.1:4500; 
        server 127.0.0.1:5000;
        server 127.0.0.1:6000;
}
```

The server on 6000 will respond in 5 seconds because of `setTimeout`, like this

```jsx
app.get('/', (req, res) => {
	setTimeout(() => {
		console.log("on 6000"); 
		res.send('res on 6000'); 
	}, 5000);
})
```

As shown in the video, we can see the fourth server (the 6000 one) pending because of setTimeout. 

In this five seconds, any request will not be sent to this server thanks to `least_conn`. 

https://user-images.githubusercontent.com/55405280/160231472-f70b4874-e686-4015-bd71-21d478672c86.mp4

## Health Checks

Shutdown the server listening on 4500, but don’t remove it from the upstream. 

```
upstream test {
        server 127.0.0.1:4000;
        server 127.0.0.1:4500;
        server 127.0.0.1:5000;
}
```

We will be able to see Nginx doing health check automatically, so the response will not come from server 4500, and can see error logs at the same time.

https://user-images.githubusercontent.com/55405280/160231485-e568b22d-e92e-4bf9-8fe8-9c5bce0538ca.mp4

We can add more settings on server to decide what happens if a server goes down.

For example, `max_fails=3 fail_timeout=5s` means Nginx will try at most three times on this server, and will not try in the following 5 seconds if all three tries failed. 

```
upstream test {
        server 127.0.0.1:4000;
        server 127.0.0.1:4500 max_fails=3 fail_timeout=5s;
        server 127.0.0.1:5000;
}
```

https://user-images.githubusercontent.com/55405280/160231496-5a648fea-d6dc-4cde-baa8-8e35c1a35a66.mp4

# Conclusion

In this article, I introduce the basic idea of load balancing with Nginx. 

The idea and the commands in Nginx should be simpler than expected, especially after grasping the core idea of why we need load balancing.

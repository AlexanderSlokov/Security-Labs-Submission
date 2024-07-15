# 20110357, Dinh Tan Dung
# Firewall lab submission

1. **Log in to container router:**
 ```sh
 docker exec -it router /bin/bash
 ```

2. **Enable IP forwarding:**
 ```sh
 echo 1 > /proc/sys/net/ipv4/ip_forward
 ```

- ![img](https://github.com/AlexanderSlokov/Security-Labs-Submission/blob/main/asset/firewallSubmission1.png?raw=true)

3. **Set up `iptables` rules:**

 **a. Block all access to the router except ping:**

- We first set up the rules for the router like this to block access from all computers:
 ```sh
 # Delete old rules
 iptables -F
 iptables -X

 # Allow ping
 iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

 # Block all other access
 iptables -A INPUT -j DROP
 ```

When a packet arrives at the router, `iptables` checks the rules in top-down order.

- If the packet is a ping request (ICMP echo request), it will match the first rule (`iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT`) and be accepted, Therefore, the router will respond to the ping.
- If the packet is another connection request (like `telnet` or `HTTP`), it will not match the ping rule and continue checking for further rules. It will eventually match the `DROP` rule (`iptables -A INPUT -j DROP`) and be dropped without response.

From the `outsider` and `inner`virtual computers, we will now try to connect with the router by telnet command for cross-check if we set up the router properly or not:
```sh
docker exec -it outsider-10.9.0.5 /bin/bash
telnet 10.9.0.254 80  # Thử kết nối tới cổng 80 của router (nên bị chặn)
curl http://10.9.0.254 # Thử kết nối HTTP tới router (nên bị chặn)
 ```

- As the image has shown, even the telnet or the curl commands connecting to the router, but it does not response, while we still can ping it:
- ![img](https://github.com/AlexanderSlokov/Security-Labs-Submission/blob/main/asset/firewallSubmission2.png?raw=true)

 **b. Prevent computers in subnet `10.9.0.0/24` from accessing the internal web server (`iweb`):**
 ```sh
 # Block access to iweb from subnet 10.9.0.0/24
 iptables -A FORWARD -s 10.9.0.0/24 -d 172.16.10.110 -p tcp --dport 80 -j DROP
 iptables -A FORWARD -s 10.9.0.0/24 -d 172.16.10.110 -p tcp --dport 443 -j DROP
 ```

- Let's perform the curl and see if the rules was set correctly:
```sh
 docker exec -it outsider-10.9.0.5 /bin/bash
 curl http://172.16.10.110 # Check for outsider access to iweb (should be blocked)
```

- ![img](https://github.com/AlexanderSlokov/Security-Labs-Submission/blob/main/asset/firewallSubmission3.png?raw=true)

- The computer in subnet was not responded to the curl command, so the iptables worked as we expected from it.

 **c. Prevent computers in subnet `172.16.10.0/24` from accessing `badsite`:**
 ```sh
 # Block access to badsite from subnet 172.16.10.0/24
 iptables -A FORWARD -s 172.16.10.0/24 -d 10.9.0.10 -p tcp --dport 80 -j DROP
 iptables -A FORWARD -s 172.16.10.0/24 -d 10.9.0.10 -p tcp --dport 443 -j DROP
 ```

- Check web server access:
 ```sh
 docker exec -it inner1-172.16.10.100 /bin/bash
 curl http://10.9.0.10 # Check access from inner to badsite (should be blocked)
 ```

- ![img](https://github.com/AlexanderSlokov/Security-Labs-Submission/blob/main/asset/firewallSubmission4.png?raw=true)

- The bad-site host was not responded to the curl request from the inner computer in the subnet.
### Check configuration:
After setting up the `iptables` rules, we check again to make sure everything works as expected.

**Check `iptables` rules:**
 ```sh
 docker exec -it router /bin/bash
 iptables -L -v -n # List iptables rules
 ```

![img](https://github.com/AlexanderSlokov/Security-Labs-Submission/blob/main/asset/firewallSubmission5.png?raw=true)

### Chain INPUT
The `INPUT` chain handles packets destined for the local system.

1. **Rule 1**:
 ```plaintext
 ACCEPT icmp -- 0.0.0.0/0 0.0.0.0/0 icmp type 8
 ```
 This rule accepts ICMP packets (typically used for ping) from any source to any destination.

2. **Rule 2**:
 ```plaintext
 DROP all -- 0.0.0.0/0 0.0.0.0/0
 ```
 This rule drops all other packets from any source to any destination.

### CHAIN ​​FORWARD
The `FORWARD` chain handles packets being routed through the system (not destined for the local system).

1. **Rule 1**:
 ```plaintext
 DROP tcp -- 10.9.0.0/24 172.16.10.110 tcp dpt:80
 ```
 This rule drops TCP packets from the `10.9.0.0/24` network to the `172.16.10.110` IP address on port 80.

2. **Rule 2**:
 ```plaintext
 DROP tcp -- 10.9.0.0/24 172.16.10.110 tcp dpt:443
 ```
 This rule drops TCP packets from the `10.9.0.0/24` network to the `172.16.10.110` IP address on port 443.

3. **Rule 3**:
 ```plaintext
 DROP tcp -- 172.16.10.0/24 10.9.0.10 tcp dpt:80
 ```
 This rule drops TCP packets from the `172.16.10.0/24` network to the `10.9.0.10` IP address on port 80.

4. **Rule 4**:
 ```plaintext
 DROP tcp -- 172.16.10.0/24 10.9.0.10 tcp dpt:443
 ```
 This rule drops TCP packets from the `172.16.10.0/24` network to the `10.9.0.10` IP address on port 443.

### CHAIN ​​OUTPUT
The `OUTPUT` chain handles packets sent from the local system.

There are no specific rules defined in the `OUTPUT` chain, so the default policy (`ACCEPT`) will apply, meaning all outgoing packets are allowed.

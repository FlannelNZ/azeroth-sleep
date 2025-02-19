# azeroth-sleep

## Strapline
Freezes an azerothcore installation when no players are connected and instantly thaws it when a player tries to connect. Reduces cpu consumption to negligible levels.

## Background
I installed azerothcore on an LXC in Proxmox - 6 vcores, 8GB ram. Was surprised to find that, when no-one is connected, CPU is at 37% (ie over 2 core for doing nothing). The authserver consumes 1 vcore and the worldserver 1.2 vcores (esp the authserver - what is it doing?) Seems rather unnecessary and don't really want the server to have to be manually started and stopped when anyone wants to play. All it really has to do is have a lightweight proxy listening for a connection and then start things up.

So I built a solution - everything runs as systemctl with worldserver on port 8085 and authserver on port 372**5**.

Then also under systemctl we have

- A script **azeroth-monitor** to `kill -STOP` the auth and worldservers when there has been no connection to the worldserver for 10 mins
- A script **azeroth-proxy** that uses socat to listen on port 3724 and, when it receives an auth request, calls **azeroth-thaw** which checks if the servers are stopped, does `kill -CONT` if required to wake them up, and then bridges the request to the real authserver on port 3725 using socat.
    
Thawing of the frozen processes is near-instantaneous (under 200ms)

Result is that, when no players are connected, it reduces from the previous idle of 37% to 0.09% which is better for my energy bills.

If you do have deploy this, let me know how you get on.

I have one issue to resolve - I have the playerbots mod installed and it seems that, sometimes, when the server thaws, there are no playerbots active. That's probably just a config setting but need to chase that down.
## Prerequisites
All installation is on the azerothcore server itself.

I have tested it with worldserver and authserver running under systemd, although it should work regardless - although this solution is under systemd, it assumes nothing about the two core services. If you do want to get them under systemd, I have included worldserver.service and authserver.service which are the ones I use (you would just need to adjust the paths)

```
sudo apt install net-tools socat
```
## Installation and Configuration
To get started, clone the repo - I put the scripts under /home/azeroth/azeroth-local.

You will need to modify something variables and oaths in the scripts since I haven't spent too much time with parameterising or env files (yet?). They should be self-explanatory:
- **azeroth-monitor**: the first 7 variables - how to identify everything in your installation, how often you want to check and how long you want to leave the servers idle before freezing
- **azeroth-proxy**: the first 4 variables - ports, ip address and script path
- **azeroth-thaw**:  just 1 - the path of the azerothcore 2  binaries
- **azeroth-monitor.service**: ExecStart is full path to the azeroth-monitor script
- **azeroth-proxy.service**: ExecStart is full path to the azeroth-proxy script

Now copy these two service files to /etc/systemd/services and:
```
systemctl enable azeroth-monitor
systemctl enable azeroth-proxy
systemctl start azeroth-monitor
systemctl start azeroth-proxy
```
## Testing
To test:
1. Check that you can login as a player.
2. Logout
3. Check cpu usage
4. Wait 5 mins (or whatever you have set the idle time to in the azeroth-monitor script)
5. Check that the cpu usage is massively reduces (you can also check the authserver and worldserver processes which should be in a frozen state)
6. Check that you can log back in again and the server processes are instantly thawed.

Enjoy! Let me know how you get on.

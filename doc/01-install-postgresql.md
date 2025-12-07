### ssh (Enable SSH keepalive to prevent automatic disconnection during idle sessions):
```
sudo nano /etc/ssh/sshd_config
```
```
ClientAliveInterval 60      
ClientAliveCountMax 3   
```
```
sudo service ssh restart
```

### install postgresql:
```
sudo apt update
sudo apt upgrade -y
```

```
sudo apt install postgresql postgresql-contrib -y
```

```
sudo systemctl status postgresql
```


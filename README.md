# SecureIPTables

A Shell script for securing IPTables on a Standard Web-server.

You will need to give the script Root privledges to run it.


# Extra Security

You should also install Denyhosts and Fail2Ban because these programs deal with various Authentication Failures, and ban people appropriately. You should research both of them, Because there are a lot of beneficial features.

```
sudo apt-get update
sudo apt-get install denyhosts
sudo apt-get install fail2ban
```

Leave the settings as default unless you've changed your SSH port, 
Then you need to Google how to configure these programs.

```
sudo service denyhosts restart
sudo service fail2ban restart
```

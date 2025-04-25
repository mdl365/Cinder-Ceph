Since we didn't have a GUI object available for Dashboard, I reverse SSH tunneled the traffic to my gHost:
```yaml
ssh -R 8443:<deviceruningdashboardIP:8443> itsvm@<gHost IP>
```

Needs to staying running, I didn't put it in the background. 

on gHost, go to http://deviceruningdashboardIP:8443
    a.) use the login credentials from bootstrap
    b.) will be prompted to change password after first login
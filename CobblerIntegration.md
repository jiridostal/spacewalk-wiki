# Integration Details

## Users/Authentication




Cobbler requires users to authenticate to it and we wanted to be able to re-use existing spacewalk users.  When spacewalk needs to do anything in cobbler it does the following steps:

 1.  User 'john' is logged into spacewalk and does something that requires communication with cobbler. (create a ks profile, etc..)
 2.  A random token is created for "john" and associated with the user.  (i.e. "john" -> "34n3k4343k").  This is stored within tomcat as a Map in a way that is shared between threads
 3.  Cobbler is contacted over XMLRPC and authentication is done with ("john", TOKEN) 
 4.  Cobbler checks uses a particular authentication method (by default spacewalk_auth is set when installed from spacewalk/satellite)
 5.  If spacewalk_auth is set, cobbler calls an xmlrpc call from the spacewalk server to authenticate the user. It calls  auth.checkAuthToken("john", TOKEN) on the spacewalk server. 
 6.  A lookup is done on the Token map for the "john" user.  If the token matches what is in the map, authentication is successful and 1 is returned to cobbler.
 7.  If cobbler receives a 1 back, then it has confirmed the user and token, creates and stores it's own auth token, and then returns that token back to spacewalk.
 8.  Spacewalk receives the cobbler auth token and uses it in the future to actually do things with cobbler.  

So to summarize:  spacewalk creates a temporary token and stores it, that gets passed to cobbler, which calls back to spacewalk to confirm the token.  


One exception to this is with taskomatic.  Since taskomatic doesn't run within tomcat, it cannot add to the 'token map' as tomcat does.  For this we use a static "taskomatic_user" user and as the authtoken we use the static "web.session_swap_secret_1".  On the spacewalk xmlrpc side of things, checkAuthToken actually first checks to see if the user being passed in is "taskomatic_user" and if so it simply checks the token against the "web.session_swap_secret_1" config value. 

web.session_swap_secret_1 is a random value in /etc/rhn/rhn.conf that is generated upon install.
## Cobbler Objects



We communicate with cobbler over XMLRPC.  Upon first integrating the two we were using raw xmlrpc calls all over the place.  Very quickly we created and started using cobbler objects.  You will see raw xmlrpc calls scattered here and there, but this is HIGHLY DISCOURAGED.  Xmlrpc calls should only be made in the cobbler objects themselves.  Hopefully we can get rid of these scattered calls soon.
 
     org.cobbler.SystemRecord
     org.cobbler.Profile
     org.cobbler.Distro
You should be able to create/edit/list these items from here.  DO NOT CALL COBBLER OVER XMLRPC ANYWHERE ELSE WITHOUT A GOOD REASON.
## Taskomatic Tasks



A couple of taskomatic tasks exist do ensure cobbler and spacewalk stay in sync.  

 CobblerSyncTask - creates a cobbler profile or distro within cobbler if it doesn't exist (but exists within satellite).  Will also sync details between the two if cobbler has changed, but this isn't used for much.  This is integral for Distros that are imported through Satellite-sync to show up within cobbler.  Also during upgrade from 5.2 -> 5.3, if this task did not run, none of the existing profiles or distros would be created within cobbler.

 KickstartFileSyncTask -  This task simply creates the files in /var/lib/rhn/kickstart/wizard/ if they do not already exist.  This is what is responsible for creating those files after an upgrade to 5.3
## Provisioning a System


## Directory Structure (where things are stored)




    /var/lib/rhn/kickstart/snippets
    /var/lib/rhn/kickstart/wizard
    /var/lib/rhn/kickstart/raw
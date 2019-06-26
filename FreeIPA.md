# Setting up FreeIPA

##### [Thom Johnson](github.com/thomascjohnson)

## Motivation

I recently went through setting up FreeIPA for Global Parametrics and I thought it would be useful to document why I did it, how I did it and what problems I ran into. I'm a novice when it comes to these things, so consider my thoughts accordingly.

The idea to setup FreeIPA was motivated by the frequent interchange of requests to setup users on our set of machines that we use for web applications, databases, simulations and more. Creating accounts on each of these machines is an operational mess and a potential security issue. Centralizing authentication via FreeIPA means consistent authentication across machines, and the ability to modify authorization by host groups and even enforce policies, among many other features. FreeIPA is (from the [FreeIPA site](freeipa.org)):

>Integrated security information management solution combining Linux (Fedora), 389 Directory Server, MIT Kerberos, NTP, DNS, Dogtag certificate system, SSSD and others.

I did not make use of the BIND DNS because I don't know enough about DNS to not spend hours and hours on it (which I actually initially did before realizing I should do it later). Seems like the FreeIPA folks agree that [it's hard to setup correctly](https://www.freeipa.org/page/DNS):

>* DNS is hard to manage and lot of admins who want to deploy FreeIPA would have difficulties setting up DNS properly.
>* DNS is central to have a decent Kerberos experience.
>* Single-master DNS is error prone, especially for inexperienced admins.

That said, I am very interested in getting it working in the future.

## Requirements

I followed [this guide from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-configure-a-freeipa-client-on-centos-7). I recommend doing the same, but here are some other points:

### Install it on a machine reserved only for FreeIPA

The nature of FreeIPA dictates that it be installed on its own machine. It shouldn't be installed on the same server as some other web application, for example. [Here is a Stack Overflow answer](https://serverfault.com/a/727380) from one of the maintainers of FreeIPA saying exactly that. I actually tried to install it on a machine with other infrastructure and learned what a bad idea it is.

### Have your full certificate chain and your private key ready

As long as you have your certificate chain and the corresponding RSA private key, setting up SSL is easy. Unfortunately for me, the wildcard certificate we have had the order of the certificate chain backwards. I spent _days_ trying to figure out what the problem was before I finally got the certificates in order and added them to the certificate manager. [This thread was extremely helpful](https://lists.fedorahosted.org/archives/list/freeipa-users@lists.fedorahosted.org/thread/T5AHK6FTTUWVBIDU5HSOYKRIKMWUZ3OH/), and so was this [post on using OpenSSL to understand certificates](https://medium.com/@superseb/get-your-certificate-chain-right-4b117a9c0fce).

## Working with FreeIPA

Once you have FreeIPA installed, there are some things to consider:

### Federate FreeIPA's LDAP server to Keycloak, but as read-only

[This post](https://access.redhat.com/solutions/3010401) explains how to integrate Keycloak with FreeIPA. I wanted to see what would happen if I allowed Keycloak to write to FreeIPA's LDAP server, and I learned the hard way that it was a bad idea. There is probably a setup that works, but I don't know enough about LDAP and how Keycloak and FreeIPA interact through it to be able to make sure that they would be in harmony (or if it's even possible). I ended up having to delete my FreeIPA install entirely and start from scratch because it screwed up seemingly _everything_. So if you want to do this, don't see what happens if you allow Keycloak to write to the LDAP server unless you want to go back to square one.

### Existing Users

We have existing users on our machines, and when we made the machines FreeIPA clients they continue to exist there. Not a problem. If you create a FreeIPA user with the same username, the machine will continue to use the local user. If you delete the user and use the FreeIPA-created user, you will have problems with user IDs, privileges and so forth, so beware. A way around this seems to be to use [ID Views](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/linux_domain_identity_authentication_and_policy_guide/id-views).

### Two-Factor Authentication

It is quite easy to setup two-factor authentication (2FA) with FreeIPA, but sometimes using it isn't so clear. You can use an OTP application like Google Authenticator, which is great. If you need to use `kinit` to run `ipa` commands, though, you have a problem. Kerberos `kinit ${admin_user}` won't work with 2FA, so you have to use a workaround described [here](https://www.freeipa.org/page/V4/Kerberos_PKINIT#How_to_Use). However, `kinit -n` won't work if you don't have ```krb5-pkinit``` installed (if I remember correctly), so be forewarned.

When you use 2FA, sometimes you will be asked for only a password and you will need to write your password and OTP together like this:

    ${password}${otp}

This is confusing if you're used to more modern incarnations of 2FA, but it actually more inline with the way that 2FA should be implemented, if I'm not mistaken. However, at other times FreeIPA will ask for your first factor (password) and second factor (OTP) separately.

### Issues with services that depend on authentication and authorization

We have services that allow users to login to web applications with machine accounts, with one example being RStudio Server. The PAM modules that are setup by default don't seem to take into account what FreeIPA does, so these web applications won't allow logins via FreeIPA users. This seems to be fixable, but I haven't been able to find a working solution in <1 hour of trying – so take my warning with a grain of salt. It does seem that working with PAM modules is the solution.

If you're using a database and want to federate to that as well, I believe it's possible at least through the LDAP server but I haven't tried yet.

## Conclusion

I'm really happy to have FreeIPA setup because it is already making our lives a lot easier. There are plenty of features I'm not making use of with it, and I look forward to getting to know them. If you have any questions, get in touch with me via email at globalparametrics.com with the handle tjohnson.

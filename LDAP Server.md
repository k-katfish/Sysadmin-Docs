# How to set up an enterprise grade LDAP server

> ...the complexity of this configuration drove some of those sysadmins to drink... ~ *The Unix and Linux System Administrators Handbook, volume 5[^1]*

I'd rather build and administer a Microsoft Active Directory domain than administer EL LDAP. And that's saying something.

This article is about how to configure an LDAP Server, and doesn't even touch on how to configure a client to point your LDAP server. See `LDAP Client setup.md` for that info.

## Assumptions/prerequisites

- You're using an EL server (Rocky or actual RHEL)
- You're using the EL Port389 server (not OpenLDAP)
- I'm going to omit sudo from these instructions, figure out where you need it yourself. (hint - sudo su)

## Installation

<https://www.port389.org/docs/389ds/howto/quickstart.html>

1. Install EL Port389

    ```bash
    yum install -y 389-ds-base
    ```

2. At this point it may be helpful to do some security things, like install a firewall only allowing SSH and LDAP traffic through (:

3. Create a configuration file for your new directory.

    `/root/instance.inf`

    ```inf
    [general]
    config_version = 2

    [slapd]
    instance_name = your_instance
    root_password = supersecret

    [backend-userroot]
    sample_entries = yes
    suffix = dc=example,dc=com
    ```

    Don't forget what supersecret is. This will become the password for the "Directory Manager" account which you'll use a few more times.

4. Spin up the instance from the file

    ```bash
    dscreate from-file /root/instance.inf
    ```

5. If you can't manage to make it come up you need to find a new line of work.

6. Set up a .dsrc file so you can talk to the instance

    `/root/.dsrc`

    ```dsrc
    [your_instance]
    # Note that '/' is replaced to '%%2f'.
    uri = ldapi://%%2fvar%%2frun%%2fslapd-localhost.socket
    basedn = dc=example,dc=com
    binddn = cn=Directory Manager
    ```

    See the web guide (linked above) for an example of a .dsrc file for remote administration.

7. Add a user

    ```bash
    dsidm instance_name user create --uid eve --cn Eve --displayName 'Eve User' --uidNumber 1001 --gidNumber 1001 --homeDirectory /home/eve
    ```

8. Reset a user password

    ```bash
    dsidm localhost account reset_password uid=eve,ou=people,dc=example,dc=com
    ```

9. Add a group

    ```bash
    dsidm localhost group create
    ```

    Call it whatever you want, say like, server_admin

10. Add the user to the group

    ```bash
    dsidm localhost group add_member server_admins uid=eve,ou=people,dc=example,dc=com
    ```

## Step 1: Let the pain begin

At this point the instructions online are like "Thats it! Cogratulations! You have users and groups and the world is all unicorns and rainbow farts!" But Batman doesn't believe in unicorns. He's here to keep the people of Gotham City safe. And also his parents died. It was tragic, they were in an alley behind a movie theatre where they were shot in a mugging gone wrong. It traumatized Batman and made him, that sweet, innocent boy, into the Batman.

### Add the memberof plugin

memberof adds a reverse link between a user and who they are a member of. This makes security lookups much faster.

1. Check the status of memberof

    ```bash
    dsconf your_instance plugin memberof status
    >Enter password for cn=Directory Manager on ldaps://localhost:
    >Plugin 'MemberOf Plugin' is disabled
    ```

2. Enable it (if it's disabled) & restart

    ```bash
    dsconf localhoyour_instancest plugin memberof enable
    >Enter password for cn=Directory Manager on ldaps://localhost:3636:
    >Enabled plugin 'MemberOf Plugin'

    dsctl your_instance restart
    >Instance "your_instance" has been restarted
    ```

3. Configure it to be useful

    ```bash
    dsconf your_instance plugin memberof set --scope dc=example,dc=com
    >Enter password for cn=Directory Manager on ldaps://localhost:
    >successfully added memberOfEntryScope value "dc=example,dc=com"
    ```

4. If you followed the instructions without reading them first, you're an idiot and you have more work to do. If you were smart and read through these steps first, you can skip this.

    We need to mark eve as being a valid memberOf target: This is only because it was created before memberOf was enabled. All new groups and users after this point will work as expected - so this is a once off task:

    ```bash
    dsidm your_instance user modify alice add:objectclass:nsmemberof
    ```

5. Run a “fixup” which will regenerate memberof for everyone in the directory.

    ```bash
    dsconf your_instance plugin memberof fixup dc=example,dc=com
    > Enter password for cn=Directory Manager on ldaps://localhost:
    > Attempting to add task entry... This will fail if MemberOf plug-in is not enabled.
    > Successfully added task entry cn=memberOf_fixup_2019-01-14T13:05:04.011865,cn=memberOf task,cn=tasks,cn=config
    ```

Now anytime you add a member to a group it will update the reverse memberOf pointer. This is really great as it allows a fast caching of “what groups someone is a member of” which helps make your server faster and simpler to administer.

### Enable TLS/SSL

This is the part where you just have to embrace your inner masochist and really let the self loathing drive you through this. It's the only way through cause it's a painful thing to work through the first time. If you bang your head against the wall hard enough and long enough enough times eventually it'll fall.

<https://access.redhat.com/documentation/en-us/red_hat_directory_server/12/html/securing_red_hat_directory_server/assembly_enabling-tls-encrypted-connections-to-directory-server_securing-rhds>

Go read the documentation and learn the differences between TLS/SSL, SSL, LDAPS, and STARTTLS over LDAP. Choose wisely. You can also use SASL and Kerberos. That's not covered in this article this is strictly about LDAP.

> I'm assuming you got a certbot cert.

1. Install certbot

    ```bash
    yum install -y epel-release
    yum install -y certbot
    ```

2. Get your cert

    ```bash
    certbot --certonly
    ```

3. Import the certificate

    ```bash
    dsctl your_instance tls import-server-key-cert /etc/letsencrypt/live/CERTNAME/cert.pem /etc/letsencrypt/live/CERTNAME/privkey.pem
    ```

4. Import the CA Cert

    ```bash
    dsconf -D "cn=Directory Manager" ldap://server.example.com security ca-certificate add --file /etc/letsencrypt/live/CERTNAME/chain.pem --name "Your - CA - Here"
    ```

5. Set the trust flags of the CA Cert

    ```bash
    dsconf -D "cn=Directory Manager" ldap://server.example.com security ca-certificate set-trust-flags "Example CA" --flags "CT,,"

    dsconf -D "cn=Directory Manager" ldap://server.example.com security ca-certificate set-trust-flags "Example CA1" --flags "CT,,"
    ```

    In my case, /e/l/l/C/chain.pem came with 2 CA certificates, which were imported as "Example CA" and "Example CA1". I trusted both of them. You may have only 1, or more certs to trust.

6. Enable TLS and set the LDAPS port

    ```bash
    dsconf -D "cn=Directory Manager" ldap://server.example.com config replace nsslapd-securePort=636 nsslapd-security=on
    ```

7. Remember to set your firewall rules:

    ```bash
    firewall-cmd --permanent --add-port=636/tcp
    firewall-cmd --reload
    ```

8. Enable RSA and set up NSS:

    ```bash
    dsconf -D "cn=Directory Manager" ldap://server.example.com security rsa set --tls-allow-rsa-certificates on --nss-token "internal (software)" --nss-cert-name Server-Cert
    ```

9. You can choose to disable plain ole ldap

    ```bash
    dsconf your_instance security disable_plain_port
    ```

10. Restart the instance and pray that it works.

    ```bash
    dsctl your_instance restart
    ```

    >Saint Jude, I've been struggling to get this to work for almost a month now, because I want to honor my Father in the work that I do I hope to faithfully fulfill this task which seems impossible. Saint Isidore you may not understand the inner workings of LDAP having studied in the world prior to the internet but you understand how complicated human logic can be. I ask both of you, please pray for me that I might be successful in my task. Heavenly Father you've seen me struggling with this and you know I want it done, partly because I just want this struggle over and through with but also because I hope that the knowledge I've gained in this task and the discipline I've learned I may bring glory to Your Name. I unite my small sufferings in this task to Yours at Calvalry that those who are afflicted might be comforted. If this is not successfull I pray that my sufferengs, compounded by this setback, may be united also with Yours, and that you might give me the grace and strength to continue on that You might be glorified in my persistance in this task. I ask all this through your Son, Jesus Christ, who lives and reigns with You in the unity of the Holy Spirit, one God + , forever and ever. Amen.

[^1] It should be noted that technically the Unix and Linux System Administration Handbook, Volume 5, was actually talking about joining a Linux system to an Active Directory Domain. "Sysadmins often want their Linux systems to be members of an Active Directory Domain. In the past, the complexity of this configuration drove some of these sys-admins to drink. Fortunately, the debut of realmd has made this task much simpler. realmd acts as a configuration tool for both sssd and Kerberos." I believe however, that the complexity of LDAP configuration was about do drive me straight to the bottle after working at this for over a month, so I included it. Cause I do believe it's appropriate here too. There's a reason most people don't use LDAP in their personal setups, and for that same reason most people aren't enterprise-grade Sysadmins.

&copy; Copyright 2023 Kyle Ketchell. This was origninally published in <https://github.com/k-katfish/Sysadmin-Docs> .

# HTTPS Terminator

A simple HTTPS reverse proxy for sites such as those hosted by
[Github Pages](https://pages.github.com/) which are either unprotected or have
a certificate which is not valid for the desired domain.

For example, [https://hoverbear.org](https://hoverbear.org) is hosted by
Github Pages at [https://hoverbear.github.io/hoverbear.org/](https://hoverbear.github.io/hoverbear.org/)
and terminated by one of these boxes. If you visit [https://hoverbear.org](https://hoverbear.org)
you'll see that the certificate presented is valid and issued by [Let's Encrypt](https://letsencrypt.org/).

**Features:**

* By default uses a $5/mo [Digital Ocean](http://digitalocean.com/) droplet.
* Supports a number of domains.
* Uses [Mozilla Intermediate TLS](https://wiki.mozilla.org/Security/Server_Side_TLS#Intermediate_compatibility_.28default.29) settings.
* Obtains (free) certificates from [Let's Encrypt](https://letsencrypt.org/).
* Renewals are handled automatically.

This document is intended to be **approachable** to novice users who have a Linux or
Mac machine. Usage on Windows should be similar but the author has no way to  test or verify
this, patches to document this are welcome.

This guide will walk you through all steps to a **production** deployment,
this **will cost you money**. The costs are very low however, and if you just
want to try it out for a couple hours the cost will only be a few cents.

If you've never used Digital Ocean before you can use
[this signup link](https://m.do.co/c/b6156cf29450) to give me a bit of a
kickback for my work on this repository.

# Getting Set Up for the Fun

If you're fairly green to all this make sure to take your time and enjoy playing
around with technology! Some things might not work, or function differently than
you expect. This is okay!

## Get a Domain

**This will cost you money!**

In order to make any decent use of this repository you need to get a domain
first! There are many sites to buy domains and I don't really know which is
best, but I will ask that you don't use GoDaddy.

You can try [Enom](http://www.enom.com/) if you don't know where to start.

You can pick a domain name and a 'TLD' like `.org`, `.com`, or even something
silly like `.ninja`. I pay about $12/yr for my `.org` domain.

Once you've done that you should have a page where you manage your domains. They
generally are a table with 3 columns: "Host name", "Address", and "Record Type".

For our purposes we'll be changing the rows that look like so:

```
Host Name | Address              | Record Type
@         | (IP we get later)    | A
www       | (IP we get later)    | A
```

Sorry I can't be more help about how to do all this, but all the services are
different! So this is for you to explore and enjoy playing with. Don't be
scared, you won't hurt anything.

## Install Dependencies

You'll need `openssl`, `python`, `pip`, and `virtualenv` installed. If you have
them, skip the rest of this section. If not, read on.

You probably already have `python` which generally includes either `pip` or
`easy_install`. I can almost guarantee you have `openssl` or something similar
installed.

If your system already has `pip` you don't need to install it, obviously! If
not, install it with the following:

```bash
sudo easy_install pip
```

Then you can install `virtualenv` with `pip`:

```bash
sudo pip install virtualenv
```

## Prepare a Virtual Environment

In order to properly provision and setup the machine we need to install some
`python` packages like `ansible` and `dopy`. Thankfully `pip` and `virtualenv`
make this quite convenient.

First we source the environment to 'enter' it, then we use `pip` to install the
required packages:

```bash
virtualenv env
source env/bin/activate
pip install -r requirements.txt
```

## Make a Key

When we're working with remote machines like those on Digital Ocean it's not a
great idea to use a password for authentication for a number of reasons.

So, instead we use a key! No not a hunk of brass with little nubs on it, a
cryptographic key. We'll use what are called RSA keys for this, but there are
other kinds. It doesn't really matter for our purpose, we're using RSA because
they're supported basically everywhere.

You may already have a key, and that's fine, you can use that. You'll need to
make a minor modification to the provisioning scripts. I'll let you explore
that on your own!

> If you aren't using `openssl` here then you may need to adapt your commands!

So let's cook up a key with the following command:

```bash
ssh-keygen -b 4096 -f keys/admin
```

You can set a passphrase if you want, but it doesn't really matter. At this
point we're ready to go!

# Setting up the Domain

**These steps will cost you money!**

If you haven't yet, go [sign up](https://m.do.co/c/b6156cf29450) for Digital Ocean!
Once you've done that we're going to "Reserve a Floating IP" and set this up in
your domain settings.

**What is a Floating IP?** Well, let's say we make this machine then later
destroy it, then make it again. It might have a different IP address then! This
would require us to go and change our domain settings again! Ugh, no fun! A
floating IP means we can keep the same IP through multiple destroys and creations.

Go to the **Networking** screen then the **Floating IPs** screen and click the
reserve an IP in a datacenter screen, or [click here](https://cloud.digitalocean.com/networking/floating_ips/datacenter).
By default your droplet will be provisioned in the **Frankfurt 1** datacenter,
so reserve your floating IP there. Once we allocate the IP to a droplet it won't
cost us any extra.

If you pick a different datacenter you get to play with the scripts to
change the droplets datacenter! This is a fun adventure and requires a small
change. Just ask if you need help.

Now that you've done this take the address which looks something like
`138.68.127.212` (yours is different, I promise) and copy it. Then we'll fill
in your domain table with the value:

```
Host Name  Address               Record Type
@          138.68.127.212        A
www        138.68.127.212        A
```

This change can take a **few hours** to spread all over the world. Why don't you
take a break, make a tea, and pay attention to your friends or loved ones for a
few minutes? Go ahead, it's ok!

Great, welcome back. Let's continue.

# Configure the Repository

First, copy the example into place:

```bash
cp settings.example.yml settings.yml
```

Open up `settings.yml` in your favorite editor (`vim`, or `atom` work great).

You need to set the `token` field to your Digital Ocean API token.
[Generate one here](https://cloud.digitalocean.com/settings/api/tokens/new) and
set it in the file.

You'll also want to change the email and change the sites list to your liking.

It's **very important** you follow this format otherwise strange things will
happen:

```yml
- in: example.org
  out: https://example-user.github.io/example-repo/
```

Pay particular note the `/` at the end of the `out` field.

# Provisioning & Updating

First enter the virtual environment like before if you aren't in it anymore:

```bash
source env/bin/activate
```

Then to provision or update the machine you can run `ansible` like so:

```bash
env/bin/ansible-playbook provision.yml --key-file keys/admin
```

**Important:** Once the "Create the host" task is done you need to
go to [Floating IPs](https://cloud.digitalocean.com/networking/floating_ips) and
assign the floating IP to your droplet. This needs to be done in a timely manner
otherwise getting certificates will fail. **If you're too slow it's okay, Just
run the command again.** Cool right?

**Also Important:** If something didn't work and you destroy the droplet (it's fine!)
please be aware that there is a rate limit on Let's Encrypt around certificate
issuance. You can't do it a whole bunch!

## Getting Out

Finally you can exit the virtual environment by closing the terminal or running:

```bash
deactivate
```

# Help & Questions

You might have a few questions or hiccups on your journey, here's a couple hints. Feel free to get in touch if you have a different problem!

## How Can I Log In?

From the repository folder you can issue this:

```bash
ssh admin@${FLOATING_IP} -i keys/admin
```

Replace `${FLOATING_IP}` with your floating IP from Digital Ocean.

## I Lost My Key!

It's okay! Don't panic! Just delete the droplet on Digital Ocean and run the
provisioning script again. Seriously, it's that easy. You'll have maybe a couple
minutes of downtime, but that's your fault for being silly and losing your key! :)

## I Lost My Settings!

Write them again! Yup, easy! Did you forget them? Replace `${FLOATING_IP}` with
your floating IP from Digital Ocean.

```bash
# Log in
ssh admin@${FLOATING_IP} -i keys/admin
# Once you're in
cat /etc/nginx/sites-enabled/default
```

You'll have to spend some time reading through that file. Pay attention to
`server_name` and `proxy_pass` lines.

## I Want to Back This Up!

Probably the easiest way to do this is just to fork the repository as a private
repository and then manually add the `keys/admin` and `settings.yml` file to it.

That's what I do. :)

## I'm Getting Let's Encrypt Weirdness!

Oh no! There is a rate limit! If you recreate the machine too many times in a
week you'll hit it. However it's rather unreasonable that you're going to
rebuild the machine 5 times in a week after you have all the certificates
issuing.

## It's Not Working!

So there is a bit of a gotcha which I haven't figured out yet... If you browse
to an address that doesn't end with a `/` sometimes it will actually *redirect*
you to the page instead of proxying it. **If you know how to fix this please let
me know!**

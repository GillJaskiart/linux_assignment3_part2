# ACIT 2420 Assignment 3 Part 2
Jaskirat Gill - A01349758

>[!note] Link to my load-balancer and file server
>Load balancer: http://24.144.71.179
>File server: http://24.144.71.179/documents


This repository provide the instructions and required files to setup the following:
- two new Arch Linux droplets
- a load balancer

Both droplets will have:
	- an `nginx` web server to generate static HTML file with system information, which will get updated every day at 5 am. 
	- a file server
	- a simple firewall configuration.

## Instructions
### 1. Create the droplets

Download the [latest Arch linux](https://geo.mirror.pkgbuild.com/images/) image and upload it to your DigitalOcean account (Manage > Backups & Snapshots > Custom Images). Choose the closest datacenter while uploading the image.

>[!note] It may take a few minutes for the image to get uploaded. 

Now, create two droplets using the newly uploaded Arch image. Remember to:
- choose the closest datacenter
- add a tag, like "web"

>[!note] The "web" tag will be used later while creating a load balancer. Using a tag, you can easily filter the droplets you want to use the load balancer for.

### 2. Create load balancer

After both the "web" tag droplets are available, we can  create the load balancer. On the DigitalOcean website, Click the green **Create** button at the top and select **Load Balancers**. 

Here are the setting for the load balancer:
- Type: Regional
- Datacenter: SFO3 (choose the same as your droplets)
- Network Visibility: External (Public)
- Connect Droplets: Search for the droplets using the "web" tag.
Rest of the setting can be default.

### 3. Setup the droplets with `setup-script`

`ssh` into both the droplets in separate windows of the terminal and run the following commands:

```bash
sudo pacman -Syu 
sudo pacman -S neovim git nginx
```

Clone this repository to access all the required files on the server. update the path to the cloned directory in the `setup-script` before running it. 

Then, run the `setup-scrip` with `sudo`, it will do the following:

- creates a system user `webgen` with home directory `/var/lib/webgen` and a `nologin` shell. 

>[!note] Why Set a Specific NGINX User and Group?
>
>Rather than running NGINX as the root user (which is potentially insecure), it is best practice to run NGINX with a less-privileged system user. This minimizes potential damage in case of a system compromise.
>
>Reference: https://www.slingacademy.com/article/nginx-user-and-group-explained-with-examples/

- creates the home directory for `webgen` with the following directories in it.![](Screenshot%202024-12-01%20at%208.51.47%20PM.png)

- copies the `generate_index` script in `/var/lib/webgen/bin`

- changes the ownership of `webgen`'s home directory to `webgen`

- copies the `generate-index.service` and `generate-index.timer` in `/etc/systemd/system` directory

- changes the timezone to **PST** from the default **UTC**

- reloads daemons

>[!note] Why reload `daemons`?
>this is done when a change is made to the `systemd` unit files, so that `systemd` become aware of the updated files.

- starts `generate-index.service`

>[!note] What does `generate-index.service` do?
>This service file runs the `generate_index` script in `/var/lib/webgen/bin` directory, which will create the `index.html` page with the system information in `/var/lib/webgen/HTML` directory.

- start `nginx` service file

- copy the `nginx.conf` file into `/etc/nginx` directory

- creates `sites-available` and `sites-enabled` directories in `/etc/nginx`

- copies the cloned `webgen_server` file into newly created `sites-available` directory

- creates a symbolic link from `/etc/nginx/sites-available/webgen_server` to `/etc/nginx/sites-enabled/`

- start `nginx`

- install `ufw` and configure it to allow `ssh` and `http` connections, and also limit `ssh`.

>[!note] `ufw` 
>It manages firewall rules on Linux systems. It is commonly used to allow, deny or limit access to services and ports to ensure security.
>
>`ufw limit ssh` is used to deny an incoming address if they attempt 6 initiations in 30 seconds.

>[!warning] Warning!
>Do not enable `ufw` before allowing `ssh` connections.

### 4. Finish the firewall setup

After the `setup-script` has run successfully,  check the `ufw` configuration with the following command:

```bash
sudo ufw status verbose
```

It should output something like this:
```
To                         Action      From
--                         ------      ----
80/tcp                     ALLOW IN    Anywhere
22/tcp                     ALLOW IN    Anywhere
80/tcp (v6)                ALLOW IN    Anywhere (v6)
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

If the configuration looks okay, you can now enable `ufw`.

```bash
sudo ufw enable
```

### 5. Final changes and file server setup

Create two files inside both droplets and add some text inside every file.

```bash
sudo nvim file-one
sudo nvim file-two
```

Change ownership of the files to `webgen`.

```bash
sudo chown webgen:webgen file-one file-two
```

`webgen_server` already has the configuration for this file server.

Finally, update the IP address in the `weggen_server` file to match your droplet's IP address. 

```bash
sudo nvim /etc/nginx/sites-available/webgen_server
```

After making the change,  check for any errors.

```bash
sudo nginx -t
```

Then, reload `nginx` service.

```bash
sudo systemctl reload nginx
```

your `nginx` web-servers and load-balancers should now be working. 

>[!question] How to check if the load-balancer is working?
>Go to your DigitalOcean account on the website and copy the IP address of your load-balancer. Then, go to your browser and use the IP address of your load-balancer. You will see the "System Information" page from one of your servers.
>
>Refresh the page, to get connected to the other server. If this is working fine, your load balancer is working fine.

>[!question] How to see the file server?
>The file server can be accessed by using `/documents` after the IP address in the browser.
>Example: http://24.144.71.179/documents


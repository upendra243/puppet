# Master less puppet configration

Master less puppet configration tutorial
# Steps for basic Configration

## Step 1 — Creating a Git Repository
The first step is to create a repository where all of our Puppet modules and manifests will be stored.
It'll look something like git@your_git_server_ip:username/puppet.git.

## Step 2 — Adding an SSH Key to Git Labs
In this step, we will create an SSH key on the Puppet server, then add that key to the Git Labs server.

Log in to the Puppet server as root. (Because Puppet's files will be owned by root, we need to have rights to setup the initial Git repo in the Puppet folder.)

Create an SSH key for the root user. Make sure not to enter a passphrase because this key will be used by scripts, not a user.
```
ssh-keygen -t rsa
```
Next, display your public key with the following command.
```
cat ~/.ssh/id_rsa.pub
```
Copy this key. It will look something like ssh-rsa long_alphanumeric_string root@hostname.

Now, on your Git Labs Dashboard page, click on the Profile settings icon on the top bar, second from the right. In the left menu, click SSH Keys, then click the green Add an SSH Key button. In the Title, field add a description of the key (like "Root Puppet Key"), and paste your public key into the Key field. Finally, click Add key.

## Step 3 — Installing Puppet and Git
In this step, we will install Puppet and Git.

On the Puppet server, first download the Puppet package for Ubuntu 14.04.
```
wget http://apt.puppetlabs.com/puppetlabs-release-trusty.deb
```
Install the package.
```
dpkg -i /tmp/puppetlabs-release-trusty.deb
```
Update your system's package list.
```
apt-get update
```
Finally, install Puppet and git.
```
apt-get install puppet git-core
```
At this point, you should configure your Git environment by following the instructions in this tutorial.

## Step 4 — Pushing the Initial Puppet Configuration
With Puppet and Git installed, we are ready to do our initial push to our Puppet repository.

First, move to the /etc/puppet directory, where the configuration files live.
```
cd /etc/puppet
```
Initialize a git repository here.
```
git init
```
Add everything in the current directory.
```
git add .
```
Commit these changes with a descriptive comment.
```
git commit -m "Initial commit of Puppet files"
```
Add the Git project we created earlier as origin using the SSH URL you copied in Step 1.
```
git remote add origin git@your_server_ip:username/puppet.git
```
And finally, push the changes.
```
git push -u origin master
```
## Step 5 — Cleaning Up Puppet's Configuration
Now that Puppet is installed, we can put everything together. At this point, you can log out as root and instead log in as the sudo non-root user you created  during the prerequisites. It isn't good practice to operate as the root user unless absolutely necessary.

To get the foundation in place, we need to make a couple of changes. First, we are going to clean up the /etc/puppet/puppet.conf file. Using your favorite editor (vim, nano, etc.) edit /etc/puppet/puppet.conf with the following changes.

Let's start by making a few changes to the /etc/puppet/puppet.conf file for our specific setup. Open the file using nano or your favorite text editor.
```
sudo nano /etc/puppet/puppet.conf
```
The file will look like this:

Original /etc/puppet/puppet.conf
```
[main]
logdir=/var/log/puppet
vardir=/var/lib/puppet
ssldir=/var/lib/puppet/ssl
rundir=/var/run/puppet
factpath=$vardir/lib/facter
templatedir=$confdir/templates

[master]
# These are needed when the puppetmaster is run by passenger
# and can safely be removed if webrick is used.
ssl_client_header = SSL_CLIENT_S_DN 
ssl_client_verify_header = SSL_CLIENT_VERIFY
```
First, remove everything from the [master] line down, as we aren't running a Puppet master. Also delete the last line in the [main] section which begins with templatedir, as this is deprecated. Finally, change the line which reads factpath=$vardir/lib/facter to factpath=$confdir/facter instead. $confdir is equivalent to /etc/puppet/, i.e. our Puppet repository.

Here is what your puppet.conf should look like once you're finished with the above changes.

Modified /etc/puppet/puppet.conf
```
[main]
logdir=/var/log/puppet
vardir=/var/lib/puppet
ssldir=/var/lib/puppet/ssl
rundir=/var/run/puppet
factpath=$confdir/facter
```
## Step 6 — Adding a Puppet Module
Now Puppet is set up, but it's not doing any work. The way Puppet works is by looking at files called manifests that define what it should do, so in this step, we'll create a useful module for Puppet to run.

Our first module, which we will call cron-puppet, will deploy Puppet via Git. It'll install a Git hook that will run Puppet after a successful merge (e.g. git pull), and it'll install a cron job to perform a git pull every 30 minutes.

First, move into the Puppet modules directory.
```
cd /etc/puppet/modules
```
Next, make a cron-puppet directory containing manifests and files directories.
```
sudo mkdir -p cron-puppet/manifests cron-puppet/files
```
Create and open a file called init.pp in the manifests directory.
```
sudo nano cron-puppet/manifests/init.pp
```
Copy the following code into init.pp. This is what tells Puppet to pull from Git every half hour.

### init.pp
```
class cron-puppet {
    file { 'post-hook':
        ensure  => file,
        path    => '/etc/puppet/.git/hooks/post-merge',
        source  => 'puppet:///modules/cron-puppet/post-merge',
        mode    => 0755,
        owner   => root,
        group   => root,
    }
    cron { 'puppet-apply':
        ensure  => present,
        command => "cd /etc/puppet ; /usr/bin/git pull",
        user    => root,
        minute  => '*/30',
        require => File['post-hook'],
    }
}
```
Save and close the file, then open another file called post-merge in the files directory.
```
sudo nano cron-puppet/files/post-merge
```
Copy the following bash script into post-merge. This bash script will run after a successful Git merge, and logs the result of the run.

### post-merge
```
#!/bin/bash -e
## Run Puppet locally using puppet apply
/usr/bin/puppet apply /etc/puppet/manifests/site.pp

## Log status of the Puppet run
if [ $? -eq 0 ]
then
    /usr/bin/logger -i "Puppet has run successfully" -t "puppet-run"
    exit 0
else
    /usr/bin/logger -i "Puppet has ran into an error, please run Puppet manually" -t "puppet-run"
    exit 1
fi
```
Save and close this file

Finally, we have to tell Puppet to run this module by creating a global manifest, which is canonically found at 

/etc/puppet/manifests/site.pp.
```
sudo nano /etc/puppet/manifests/site.pp
```
Paste the following into site.pp. This creates a node classification called 'default'. Whatever is included in the 'default' node will be run on every server. Here, we tell it to run our cron-puppet module.

### site.pp
```
node default {
    include cron-puppet
}
```

Save and close the file. Now, let's make sure our module works by running it.
```
sudo puppet apply /etc/puppet/manifests/site.pp
```
After a successful run you should see some output ending with a line like this.

Notice: Finished catalog run in 0.18 seconds
Finally, let's commit our changes to the Git repository. First, log in as the root user, because that is the user with SSH key access to the repository.

Next, change to the /etc/puppet directory.
```
cd /etc/puppet
```
Add everything in that directory to the commit.
```
git add .
```
Commit the changes with a descriptive message.
```
git commit -m "Added the cron-puppet module"
```
Finally, push the changes.
```
git push -u origin master
```
# Conclusion
To add more servers, simply follow step 3 above to install Puppet and Git on the new server, then clone the Git repository to /etc/puppet and apply the site.pp manifest.

You can even automate this installation by using user data when you create a Droplet. Make sure you use an SSH key when you create the Droplet, and have that SSH key added to your GitLab server. Then just tick the Enable User Data checkbox on the Droplet creation screen and enter the following bash script, replacing the variables highlighted in red with your own.
```
#!/bin/bash -e

## Install Git and Puppet
wget -O /tmp/puppetlabs.deb http://apt.puppetlabs.com/puppetlabs-release-`lsb_release -cs`.deb
dpkg -i /tmp/puppetlabs.deb
apt-get update
apt-get -y install git-core puppet

# Clone the 'puppet' repo
cd /etc
mv puppet/ puppet-bak
git clone http://your_git_server_ip/username/puppet.git /etc/puppet

# Run Puppet initially to set up the auto-deploy mechanism
puppet apply /etc/puppet/manifests/site.pp
```
That's all! You now have a masterless Puppet system, and can spin up any number of additional servers without even having to log in to them.

    ****************************************
    * This project is no longer maintained *
    ****************************************

VAGRANT-JENKINS
===============

By [Albert Albala (alberto56)](https://drupal.org/user/245583), with amendments and additional settings from (Paflenti)

*A quick, automated way to deploy a Jenkins server. Can be used with or without vagrant to provision any server (whether or not it's a VM on your local machine). The Jenkins server then created will be able to monitor your Drupal sites' code. Please see [Dcycle project](http://dcycleproject.org) for some best practices.*

This is meant to be used with Vagrant and Virtual Box to set up a Jenkins server running on CentOS 6.x. This has been tested with Windows 7 as a host machine, but it should be possible to run this on any host system which supports Vagrant, VirtualBox and Puppet.

For an initial deployment:

 * Install *the latest version* of [VirtualBox](https://www.virtualbox.org/wiki/Downloads) on your computer (if your VM fails to boot, especially after upgrading your OS, please upgrade your copy of VirtualBox).
 * Install Vagrant on your computer (https://www.vagrantup.com/)
 
Type the following command, from the root of this directory (`vagrant-jenkins`):

    vagrant up

You might have to wait for few minutes while all the relevant files are downloaded. Once the base box is already installed, it will take less time.

Once your box is running, and assuming no other applications (including other instances of the same box code) use port 8082, you will be able to access the guest's Jenkins at the address http://localhost:8082,

For an incremental deployment (if you've already deployed a previous version of this, which you want to update):

    vagrant reload
    vagrant provision

You might need to follow further instructions on-screen.

You can then log into your box:

    vagrant ssh


 
It is a good idea to change the MySQL root password. You can call this:

    mysqladmin -u root -p'CHANGEME' password 'princess'

In my tests the mysql root password is sometimes empty, in which case the following might work:

    mysqladmin -u root password 'princess'

If that does not work you might have to follow the instructions [here](http://www.cyberciti.biz/tips/recover-mysql-root-password.html), using `sudo` for commands which give you an access denied.

Note finally that by default Jenkins is not using a password; *you will want to change this if your machine is publicly accessible*, here is how:

Setting up a password on Jenkins
--------------------------------

At first Jenkins has no security, meaning any user can do anything even without being logged in.

 * Go to Manage Jenkins > Global security
 * Check "Logged in users can do anything"
 * Immediately realize you don't have an account.

To get out of this mess, do the following:

    sudo vi /var/lib/jenkins/config.xml

In that file, find `<useSecurity>true</useSecurity>` and change it to `<useSecurity>false</useSecurity>`. Then type:

    service jenkins restart

Now let's do without locking you out of Jenkins:

 * Go to configureSecurity
 * Check "Enable security"
 * Check "Jenkins' own user database"
 * Check "Allow users to sign up"
 * Check "Logged in users can do anything"
 * Save
 * Click "Create an account"
 * Username: admin, password: princess
 * Go back to configureSecurity
 * Uncheck "Allow users to sign up"
 * Save
 * At this point anonymous users will _still_ be able to _see_ jobs and artefacts, but not initiate builds or change jobs. If you want anonymous users to be shown a login screen and nothing else, follow [these instructions](http://stackoverflow.com/questions/14226681).

Setting up a Drupal project
---------------------------

To set up a Drupal project on Jenkins:

Visit http://localhost:8082

If your site is accessible publicly, please make sure you have set up security correctly (at least a password!).

Set up your first job (Freestyle software project), make sure it can connect via SSH to the git repo (If you are having trouble connecting Jenkins and Git, [read this blog post](http://dcycleproject.org/blog/51)), and make sure your MySQL root password is changed.

Now run your job, and visit the console output.

Note the workspace; it will be something like:

    /var/lib/jenkins/jobs/myjob/workspace

Now, in the command line, log in as the jenkins user (`sudo su -s /bin/bash jenkins`) and `cd` into your workspace.

Now make sure sites/default/default.settings.php exists (you might have removed it for security reasons), create your database and install Drupal:

    echo 'create database mysite'|mysql -uroot -pprincess
    drush si --db-url=mysql://root:princess@localhost/mysite

You can now exit the jenkins user:

    exit

Make a local domain in the *guest system*'s `etc/hosts`

    sudo vi /etc/hosts

Add the following line

    127.0.0.1 mysite.jenkins

Add virtual hosts:

    sudo su
    SITE=mysite
    echo '<VirtualHost *:80>' >> /var/lib/jenkins/conf.d/$SITE.conf
    echo "     DocumentRoot /var/lib/jenkins/workspace/$SITE" >> /var/lib/jenkins/conf.d/$SITE.conf
    echo "     ServerName $SITE.jenkins" >> /var/lib/jenkins/conf.d/$SITE.conf
    echo '</VirtualHost>' >> /var/lib/jenkins/conf.d/$SITE.conf

Now restart apache on the guest

    sudo apachectl restart

On your *host machine*, make sure `/etc/hosts` is modified also:

    sudo vi /etc/hosts

Add the following line

    127.0.0.1 mysite.jenkins

Log back in as the Jenkins user, go to your workspace, and in your sites/default/settings.php, make sure the base URL is set correctly to be accessible to the VM. So your line will look like:

    $base_url = 'http://mysite.jenkins';  // NO trailing slash!

Enable Simpletest and run your test suite. If you are using a [site deployment module](http://dcycleproject.org/blog/44), and you have define your simpletests there, the following should run your tests:

    drush en simpletest -y
    drush test-run mysite_deploy

Once you get that working, you can add an "Execute shell" step to your Jenkins job via the UI. However, even if your test fails, Jenkins might still mark your job as successful, as [documented here](https://github.com/drush-ops/drush/issues/212). This is why I have included the Jenkins [Log Parser plugin](https://wiki.jenkins-ci.org/display/JENKINS/Log+Parser+Plugin) in this distribution to look for output patterns in addition to exit codes in order to determine the status of a build.

You should set it up by visiting http://localhost:8082/configure, adding a Console Output Parsing rule, with the description "Main parsing file" and the File "/tmp/logparser" (these should have been created by puppet).

Now, configure your job by adding the Console Output (build log) parsing post-build action, mark build failed as error, and save.

Note that if you need more verbose results on the command line test run, for example if tests are working on your dev machine but not on jenkins, you can run, from the command line or from the jenkins job:

    php scripts/run-tests.sh --verbose mysite_deploy

Troubleshooting
---------------

 * Please see the [issue queue](https://github.com/alberto56/vagrant-jenkins/issues) if you are having troubles, and add a new issue if you don't find what you are looking for.


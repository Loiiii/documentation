---
title: Migrate to Pantheon: Manual Site Import
description: Learn how to manually import a Drupal or WordPress site into Pantheon
keywords: migrate, import, importing site, pantheon, new site, large site, distro, upstream, git history
categories: [developing]
tags: [migrate]
---

Manually migrate your site to Pantheon when any of the following apply:

* **Large Site Archives**: Site archive is greater than the automated import limits (100MB for direct file upload or 500MB for URL upload).
* **Custom Upstream**: Site should receive updates based on an upstream other than vanilla Drupal or WordPress (e.g Panopoly or your agency's customized WordPress).
* **Preserve Git History**: Site's existing Git commit history should be retained.
* **[WordPress Site Networks](/docs/wordpress-site-networks/)**

## Requirements

* [Download](http://git-scm.com/downloads) and install [Git](/docs/git/)
* [Rsync or SFTP Client](/docs/rsync-and-sftp/)
* [MySQL Client](/docs/mysql-access/)

## Create a New Pantheon Site

From your Pantheon Dashboard:

1. Choose **Migrate Existing Site**.
2. Enter your current website URL.
3. Name your new Pantheon site.
4. Select an organization for the site (optional).
5. Select **Migrate Manually**.
6. Import your code, database, and files.
7. Once the site has been imported, click **I've Successfully Migrated Manually**.

You will now be taken to your Pantheon Site Dashboard.

## Import the Codebase

**Codebase** - all executable code, including core, custom and contrib modules or plugins, themes, and libraries.

As long as you've chosen the same codebase (Drupal 7, Commerce Kickstart, etc.) as the starting point of your Pantheon site, you can use Git to import your existing code and commit history. If you don’t have a Git version controlled codebase, the following will still work.

1. Navigate to your existing site's code directory in a local terminal. If your existing code is not version controlled with Git, create a repository and add an initial commit:

 ```bash
 git init
 git add .
 git commit -m "initial commit"
 ```
2. From the Dev environment of the Site Dashboard, set the site's [connection mode](/docs/getting-started/#interact-with-your-code) to Git.
3. Copy the SSH URL for the site repository, found in the <a href="/docs/git/#step-2-copy-the-git-clone-command" data-proofer-ignore>clone command</a>. **Do not copy `git clone` or the site name.** The URL should look similar to the following:

 ```bash
 ssh://codeserver.dev.{site-id}@codeserver.dev.{site-id}.drush.in:2222/~/repository.git
 ```

4. Add Pantheon as a remote destination, replacing `<ssh_url>` with the SSH URL copied in step 3:

 ```bash
 git remote add pantheon <ssh_url>
 ```

5. **Drupal only**: To preserve the database connection credentials for a site built on a local development environment, and to exclude them from version control, move your `settings.php` file to `settings.local.php` and add it to `.gitignore` so that it will be ignored by Git and included from Pantheon's `settings.php` when working on your site locally. Make sure that you can modify it, and restore the protections after the move:

 ```bash
 chmod u+w sites/default/{.,settings.php}
 mv sites/default/{settings.php,settings.local.php}
 chmod u-w sites/default/{settings.local.php,.}
 ```
 Drupal 8 sites running on Pantheon come with a bundled `settings.php` that includes the `settings.local.php` file, so no additional steps are required. However, sites running Drupal 6 or 7 must add a `settings.php` file that includes `settings.local.php`, as this file is not bundled on Pantheon.

6. Use Git to pull in the upstream's code (which may have Pantheon-specific optimizations) to your existing site's codebase:

 ```bash
 git pull --no-rebase --squash -Xtheirs pantheon master
 ```  

 Will yield:  
 ```bash
 Squash commit -- not updating HEAD  
 Automatic merge went well; stopped before committing as requested
 ```
 Authenticate using your Pantheon Dashboard credentials when prompted for a password. We recommend enabling passwordless access to the site's codebase for Git by [loading an SSH key](/docs/ssh-keys/) into the User Dashboard.

7. Run git commit to prepare the Pantheon core merge for pushing to the repository:
 ```bash
 git commit -m "Adding Pantheon core files"
 ```
8. Align your local branch with it's remote counterpart on Pantheon:

 ```bash
 git pull pantheon master
 ```
9. Push your newly merged codebase up to your Pantheon site repository:

 ```bash
 git push pantheon master
 ```

10. Go to the Code tab of your Dev environment on the Site Dashboard. You will see your site's pre-existing code commit history and the most recent commit adding Pantheon's core files.

## Add Your Database  

**Database** - a single .sql dump that contains the content and active state of the site's configurations.

You'll need an .sql file containing the data from the site you want to import. If you haven't done so already, make sure you remove any data from the cache tables. That will make your .sql file much smaller and your import that much quicker.


1. From the Dev environment on the Site Dashboard, click **Connection Info** and copy the Database connection string. It will look similar to this:

 ```
 mysql -u pantheon -p{massive-random-pw} -h dbserver.dev.{site-id}.drush.in -P {site-port} pantheon
 ```
2. From your terminal, `cd` into the directory containing your `.sql` archive. Paste the connection string and append it with:
`< database.sql`
Your command will now look like:

 ```
 mysql -u pantheon -p{massive-random-pw} -h dbserver.dev.{site-id}.drush.in -P {site-port} pantheon < database.sql
 ```
3. After you run the command, the .sql file is imported into your Pantheon Dev database.

## Upload Your Files

**Files** - anything in `sites/default/files` for Drupal or `wp-content/uploads` for WordPress. This houses a combination of uploaded content from site users, along with generated stylesheets, aggregated scripts, image styles, etc. For information on highly populated directories, see [Platform Considerations](/docs/platform-considerations/#highly-populated-directories).

Files are stored separately from the site's code. Larger file structures can fail in the Dashboard import due to sheer volume. It's best to use a utility such as an SFTP client or rsync. The biggest issue is having the transfer stopped due to connectivity issues. To handle that scenario, try this handy bash script:  

```bash
ENV='ENV'
SITE='SITEID'

read -sp "Your Pantheon Password: " PASSWORD
if [[ -z "$PASSWORD" ]]; then
echo "Woops, need password"
exit
fi

while [ 1 ]
do
sshpass -p "$PASSWORD" rsync --partial -rlvz --size-only --ipv4 --progress -e 'ssh -p 2222' ./files/* --temp-dir=../tmp/ $ENV.$SITE@appserver.$ENV.$SITE.drush.in:files/
if [ "$?" = "0" ] ; then
echo "rsync completed normally"
exit
else
echo "Rsync failure. Backing off and retrying..."
sleep 180
fi
done
```
This script connects to your Pantheon site's Dev environment and starts uploading your files. If an error occurs during transfer, rather than stopping, it waits 180 seconds and picks up where it left off.  

If you are unfamiliar or uncomfortable with bash and rsync, an FTP client that supports SFTP, such as FileZilla, is a good option. Find your Dev environment's SFTP connection info and connect with your SFTP client. Navigate to `/code/sites/default/files/`. You can now start your file upload.  

You should now have all three of the major components of your site imported into Pantheon. Clear your caches via the Pantheon Dashboard, and you are good to go.

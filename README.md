# koha-testing-docker---command-cheat-sheet
koha related cheat sheet

NOTE: To be read in conjunction with the KTD README

# Workflow for testing

## Update repositories

Update your Koha repository:
1. Change to your Koha cone directory: `cd koha`
2. Update your Koha clone: `git pull`

Update your KTD repository:
1. Change to your KTD directory: `cd ../koha-testing-docker`
2. Update your KTD clone: `git pull`

## Update images and start KTD

Update docker images: `ktd pull`

Start up KTD: `ktd up`

Start up KTD with Elasticsearch: `ktd --es8 up` (can also use --es7, --os1, and --os2)

## Access the Koha "container"

Use a separate terminal window to access the Koha container: `ktd --shell`

## Create a branch

1. Create a branch for the bug: `git checkout -B bzxxxxx`
2. Reproduce the issue if it is not already part of test plan.

## Apply the patches

1. Apply the patch: `git bz apply <bugzilla patch number>`
2. If there are many dependencies (instead of pressing y multiple times): `yes | git bz apply bugnumber`
3. If database updates are required: perl installer/data/mysql/updatedatabase.pl or `updatedatabase`
4. If a database schema update is required: `dbic`
5. Run `yarn build` if the bug makes changes to:
   - CSS
   - JavaScript
   - modules that use Vue (ERM, Preservation)
6. If the API specification was changed: `yarn api:bundle` (`yarn build` also does this)
7. `flush_memcached` (may have to use in some cases, such as getting changes to /etc/koha/sites/kohadev/koha-conf.xml and other configuration changes recognised)
8. Restart services: `restart_all` (restarts things such as Plack and Apache)

Some bugs may require a `reset_all`. This resets the database to the default data, and is the same as starting KTD. Notes:
- Examples of when to use: default data in .yaml files changes, index mappings change, changes to the default framework.
- Any changes made in Koha are lost (such as system preference changes).

Applying patches using the interactive mode (this may occassionaly be required, for example when you need to apply a regression patch before applying the rest of the patches):
1. Select `i` instead of yes
2. Select the patches to apply by putting a `#` in front of the patches to ignore for the moment
3. Exit the editor: `:wq`
4. Apply the patch again and add or remove `#` for patches as required

## Test the bug

Work through the test plan. I normally try to reproduce the issue, if it is not included as part of the test plan.

## Sign off a patch

Single patch:
1. `git so 1`
2. `git bz attach -e xxxxx HEAD`

Multiple patches, for example 2:
1. `git so 2`
2. `git bz attach -e xxxxx HEAD~2..`

Sign off a single patch in a patch set (for example, there may be a follow up patch that needs sign off):
1. `git so 1`
2. `git bz attach xxxxx HEAD` (no -e)
3. Manually change the status on the bug

## Reset your git repository

1. `git branch -D bzxxxxx`
2. `git checkout origin/main`
3. `git reset --hard origin/main`

## Reset things so you can test another bug

Reset from within the container: `reset_all`

## Shut down KTD

Close things down:
1. Exit from the KTD shell: `exit`
1. Get back to the command line (from the terminal docker is running in): `CTRL-C`
2. Stop things running: `ktd down`

# Testing notes

## Frequently used commands

Things you can do from within the Koha container:

* Reindex - Zebra: `koha-rebuild-zebra -d -f -v kohadev`
* Reindex - Elasticsearch (requires starting up KTD with ES or OS: `ktd --es8 up`): `koha-elasticsearch --rebuild -d -b -a kohadev`
* Access the database: `koha-mysql kohadev`
* Check the codebase for phrases [1]: `git grep word`

[1] Reference: https://irian.to/blogs/learn-git-grep-to-boost-your-command-line-search/

## Occassionaly used commands

* See what is running (in a separate terminal): `docker ps`
* Restart a container: `docker container restart [container name]` (for example: `docker container restart koha-db-1`)
* Create a saved report that takes a while to run: `select sleep(10)`
* Run the Cypress tests:
   * Run all Cypress tests:`perl /kohadevbox/misc4dev/run_tests.pl --run-cypress-tests-only`
   * Run an individual Cypress test: `yarn cypress run --spec t/cypress/integration/ERM/DataProviders_spec.ts`
* Running Selenium tests - requires starting up KTD with Selenium: `ktd --selenium up`
* Run all tests: `ktd --shell --root`, cd `/kohadevbox/koha`, then `perl /kohadevbox/misc4dev/run_tests.pl --run-all-tests` (note: will drop the database)
* Run only perl tests (excluding Selenium and the installer/onboarding tests) use the flag `--run-light-test-suite`
* RabbitMQ service: `sudo -s` `service rabbitmq-server stop|start`
* Logs in a separate terminal window: `ktd logs -f1

### Testing UNIMARC bugs

1. To setup a UNIMARC environment, edit the KTD .env file and  set `MARC_FLAVOR=unimarc`.

2. To check that KTD is using UNIMARC:
   - check MARC frameworks: should just be the default, ACQ and FA
   - search the catalog for cat: should be French titles

## Rarely used commands

* Run Koha as a Mojolicious application:
  * Find the Koha Docker container IP address:
    * Open a new terminal window on your computer
    * Find the name of the Koha container: `docker ps` (normally kohadev-koha-1)
    * Find the IP address of the Koha container (look forIPAddress near the end of the output): `sudo docker container inspect kohadev-koha-1`
  * From within the KTD shell start Koha as a Mojolicious app: `bin/intranet daemon -l http://*:3000/`
  * Access Koha using the Mojolicious IP address and port: `http://ipaddress:3000`

## Working with remote branches

Sometimes the code changes for a bug are too big to be attached to Bugzilla and are in a developers own repository, for example: https://gitlab.com/koha-dev/koha-dev and a specific branch.

Testing a bug on a remote branch:

1. Add a remote, for example: `git remote add owen https://gitlab.com/koha-dev/koha-dev`
2. Fetch the remote: `git fetch owen`
3. Checkout the branch for the remote that you want: `git checkout -b bug-26949-tinymce-upgrade owen/bug-26949-tinymce-upgrade`
4. Continue as normal

Signing off a bug on a remote branch:
1. Note in the bug that you have tested and happy to sign off.
2. Author updates remote branch with sign-off.
3. Author changes bug status to signed-off.

Notes:
* In the example commands used I have named the remote `owen`, it can be anything.
* To list the remotes you have: `git remote -v`
* To remove a remote: `git remote rm name-of-remote`

## Testing string translation bugs

Sequence when testing fixes for string translations (have used de-DE language as an example):
1. Apply the patch.
2. Fetch new translations (note that these files are now in their own git repository):
   - `cd misc/translator/po`
   - `git checkout main` (if not already on main)
   - `git fetch origin`
   - `git reset --hard origin/main`
3. Update the translation .po files with new strings in the Koha codebase (for example, when testing a patch): `gulp po:update --lang de-DE`
4. To generate new templates using those new .po files (if the language isn't already installed, skip to step 5): `koha-translate --update de-DE --dev kohadev`
5. Install the updated translation: `koha-translate --install de-DE --dev kohadev`
6. Enable the `language` and `OPACLanguages` system preferences, and select the languages to use: Administration > System preferences > I18N/L10N
7. Change the interfaces to the required language.
8. Test that the updated translation works.
9. Clean up the files.
   * `misc/translator/po`:
     * `git status`
     * `git fetch origin`
     * `git reset --hard origin/main`
     * `git clean -fd`
   * `/kohadevbox/koha`
     * `git clean -fd`
     * a `git status` should show everything is tidied up so the bug can be signed off

Troubleshooting: https://gitlab.com/koha-community/koha-testing-docker#translation-files

Reference: https://wiki.koha-community.org/wiki/Translating_Koha#Updating_the_po_files_in_your_installation

To only install a language for testing (where no updates to strings are required): `koha-translate --install de-DE --dev kohadev`

## Cleaning up files

Sometimes, such as when testing translation fixes, git status will show changes not staged for commit and unstaged files.

Once you have tested and are ready for sign off, these files need to be tidied up so you can do the sign off.

To clean up single files or directories for "Changes not staged for commit": `git checkout <pathtofile>`

To clean up all "Untracked files":
1. Change to the koha directory: `cd /kohadevbox/koha/`
2. `git clean -fd`

Reference: https://koukia.ca/how-to-remove-local-untracked-files-from-the-current-git-branch-571c6ce9b6b1

## Accessing the database container

To get into the database container: `docker exec -it koha-db-1 bash`

Restart database container: `docker restart koha-db-1` (or whatever the container name is)

Notes:
* This may be needed if you need to change any database configuration settings. For most database changes and queries using `koha-mysql kohadev` should be sufficient
* MySQL root password = the password from the .env file (the default is password)

## Using a different operating system

Use a different operating system with KTD (such as Ubuntu or an older version of Debian):
1. Edit your .env 
2. Change KOHA_IMAGE, for example: main-bullseye (for Debian 11)

## Using Koha versions other than main

Sometimes you may want to check how a specific version of Koha worked, or test a bug for an older version of Koha.

To start up KTD with a specific branch or version of Koha:
1. For your clone of the Koha git repository:
   - check out a branch, such as 24.05.x: `git checkout 24.05.x`
   - check out a specific tag, such as 24.05.01: `git checkout tags/v24.05.01 -B <branchname>`
2. For KTD, change KOHA_IMAGE from =main to =version-required in your .env change, for example: `KOHA_IMAGE=24.05`
3. Update your KTD images: `ktd pull`
4. Start up and stop as normal. It may take longer to start as it will download the docker images for the required version of Koha.

Alternative for step 2 (no editing of your .env required): `KOHA_IMAGE=24.05 ktd up`

Note: Generally only supported versions of Koha will work (the LTS version and versions that have a release maintainer).

## Bugzilla - incorrect status

If you spot a bug that is signed off, but the status is not updated, update Bugzilla.

For bugs signed off, but where the status is not updated, I normally add a comment so that the person who signed it off gets the credit on the Dashboard (this step is often missed when bugs are tested using the sandboxes):

```
Hi <name>.

It looks like you've signed off this bug.

Please update the bug status to Signed Off. (This will also give you the credit on the Dashboard at https://dashboard.koha-community.org).

<your name>
```

## Logs

To enable Zebra logging:
1. Edit /etc/koha/sites/kohadev/koha-conf.xml
2. Find the `zebra_loglevels` setting.
3. Uncomment and change what is required, such as adding 'request'.

To see changes to the logs when you are testing a bug:
1. Open a new terminal
2. Access the container: `ktd --shell`
3. "tail" the logs: `tail -f /var/log/koha/kohadev/*.log`

## Starting the web installer

Sometimes you need to test the web installer, instead of using the default setup created by KTD:
1. Access the database server[1]: mysql -uroot -ppassword -hkohadev-db-1
2. Drop the koha_kohadev database: drop database koha_kohadev;
3. Create the database: create database koha_kohadev;
4. Add privileges (for a real installation this would be limited): `grant all on koha_kohadev.* to koha_kohadev;`
5. Restart everything (there may be some errors listed): `flush_memcached` and `restart_all`
6. Access the web installer: go to `127.0.0.1:8081`
7. Use the database user name and password from /etc/koha/sites/kohadev/koha-conf.xml (the defaults are koha_kohadev, password)
8. Continue through the installation process

Note:
[1] Database password is password (from the KTD .env file)

## Enabling email for testing

Use a Google account to test sending emails:
1. Set up an App password for your Google Account (see the Google help center, or search for a tutorial).
2. Configure Koha using one of these methods:
   - The really easy way - add an SMTP server in the staff interface
   - The easy way - add a valid SMTP configuration to koha-conf.xml
   - The harder way - use the Exim4 MTA
3. Update the `KohaAdminEmailAddress` system preference with your Gmail address.
4. Add your email address to a patron's account.

### The really easy way - add an SMTP server in the staff interface

1. Go to Administration > Additional parameters > SMTP servers.
2. Add a new SMTP serber: `+ New SMTP server`
3. Complete the details and `Submit`:
   - Name: whatever you want to call it
   - Host: smtp.gmail.com
   - Port: 587
   - Timeout: 5
   - User name: GOOGLEACCOUNTUSER (your Google email address)
   - Password: GOOGLEAPPPASSWORD (your Google App password, not your Google account password)
   - Debug mode: Enabled
   - Default server: select

### The easy way - add a valid SMTP configuration to koha-conf.xml

1. Edit /etc/koha/sites/kohadev/koha-conf.xml
2. Add this configuration near the end, where:
   - GOOGLEACCOUNTUSER = your Google email address
   - GOOGLEAPPPASSWORD = your Google App password, not your Google account password
```
<smtp_server>
    <host>smtp.gmail.com</host>
    <port>587</port>
    <timeout>5</timeout>
    <ssl_mode>STARTTLS</ssl_mode>
    <user_name>GOOGLEACCOUNTUSER</user_name>
    <password>GOOGLEAPPPASSWORD</password>
    <debug>1</debug>
 </smtp_server>
```

### The harder way - use the Exim4 MTA

Note: Last tested in 2022, I now use `The really easy way`.

Configure Exim4 (an MTA installed in KTD) for sending test emails:
1. Create an "App" password for your Google account
2. Enable email for your Koha instance, for example: koha-email-enable kohadev
3. Set an email address for system preferences KohaAdminEmailAddress and ReplyTo
4. Add an email address to any accounts you are going to use.
5. Emails are added to the message_queue table in the database and are sent when the cronjob is run: perl misc/cronjobs/process_message_queue.pl
6. Run through the Exim4 setup (sudo dpkg-reconfigure exim4-config) with these values (may not match exactly):
* Select: 2. mail sent by smarthost; received via SMTP or fetchmail
* System mail name: localhost
* IP-addresses to list on for incoming SMTP connections: 127.0.0.1
* Other destinations for which mail is accepted:
* Machines to relay mail for:
* Visible domain name for local users: localhost
* IP address or host name of the outgoing smarthost: smtp.gmail.com::587 (note the double-colons between the hostname and the port number)
* Hide local mail name in outgoing mail? [yes/no] : no
* Keep number of DNS-queries minimal: No
* Delivery method for local mail: 1 mbox format
* Split configuration into small files: Yes
* Root and postmaster mail recipient: 
7. Edit /etc/exim4/passwd.client and add (with appropriate values):
   smtp.gmail.com:emailaddress@gmail.com:AppPasswordHere
8. Update and restart Exim4:
   update-exim4.conf
   service exim4 restart
9. Send a test email from the command line: echo test only | mail -s 'Test subject' youremailaddress@gmail.com
10. Send a test email from within Koha: add some items to the cart, send

Other notes:
* Monitor the send log: `sudo tail -f /var/log/exim4/mainlog`

Reference:
* Setting up Exim4 with GMail and 2-factor Authentication: https://www.talk-about-it.ca/setting-up-exim4-with-gmail-and-2-factor-authentication/

## Managing Docker containers and images

* Update all images used: `ktd pull`
* List all images on device: `docker image list`
* List all 'dangling' images: `docker image list --filter dangling=true`
* List all volumes: `docker volume ls`
* Remove a specific container: docker container rm [ID]
* Remove all unused docker things (all stopped containers, dangling images, unused networks, and unused images; doesn't delete unused volumes): `docker system prune -a`
  - This frees up a lot of space on your hard drive
  - This will remove everything, so if you use docker for other projects use other commands
* Remove all unused volumes: `docker system prune --volumes`

References:
* https://linuxize.com/post/how-to-remove-docker-images-containers-volumes-and-networks/
* https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes

## One-off or very occassional testing

### Running an SFTP server

Add these commands to `docker-compose-light.yml` (before the networks section):
```
    sftp:
        image: atmoz/sftp
        command: koha:koha:::upload
        networks:
            - kohanet
```

To connect to the SFTP server from within the KTD shell: `sftp koha@sftp`

To connect to the SFTP server from outside the KTD shell:
1. Identify the IP address of the SFTP server (kohadev-sftp-1):
```
docker inspect -f '{{printf "====================\n"}}{{.Name}} {{range $net,$v := .NetworkSettings.Networks}}{{printf "\n"}}{{printf "\n"}}{{printf "%s\n" $net}}{{.IPAddress}}{{printf "\n"}}{{end}}' $(docker ps -q)
```
2. Connect to the SFT server using the IP address for kohadev-sftp-1, for example: `sftp koha@172.18.0.2` 

### Running tests automatically multiple times

Occassionaly you may need to run tests multiple times to detect an error. Here is an example of how to run a test automatically 300 times and stop on an error, instead of running manually:
```
  % for i in {1..300}; do echo "loop $i"; yarn cypress run --spec t/cypress/integration/Admin/RecordSources_spec.ts ; if [ "$?" = "1" ]; then break; fi; done
```

# Workflow for creating patches

1. Check main is up to date: `git pull`
2. Create a branch for the bug: `git checkout -B bzXXXXX`
3 Create the test plan:
   - Should walk through all the steps (pretend that the tester is not familiar with this area of Koha)
   - Should be able to reproduce the issue
   - Apply the patch(es)
   - Restart everything (if required) or refresh the page(es)
   - Demonstrate that the issue is resolved
4. Edit files to make the changes.
5. Tidy new blocks of code introduced or modified - see https://wiki.koha-community.org/wiki/Perltidy
6. Commit files: `git add` (use . or name specific files, can use the name of  a directory as well)
7. Check to make sure there are no stray files: `git status`
8. Commit changes: `git commit`
9. Add the commit message - see the guidelines: https://wiki.koha-community.org/wiki/Commit_messages
10. Run the QA scripts: `qa --more-tests`
11. Submit the patches to Bugzilla: `git bz attach XXXX HEAD` (for a single patch)
12. Check the bug on Bugzilla: correct status, bug title, and so on
12. Add the release note text.

Other useful commands:
* Amend a commit (including the commit message): `git commit --amend`
* Amend multiple commits: `git rebase -i HEAD~X` (where X = the number of patches to go back, change pick to reword)

Reference:
* [Changing a commit message (GitHub)](https://docs.github.com/en/pull-requests/committing-changes-to-your-project/creating-and-editing-commits/changing-a-commit-message)
* [Koha development workflow](https://wiki.koha-community.org/wiki/Development_workflow)
* [Koha developer handbook](https://wiki.koha-community.org/wiki/Developer_handbook)
* [Koha coding guidelines](https://wiki.koha-community.org/wiki/Coding_Guidelines)

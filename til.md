## Git
### show git config
  git config --list

### diff'ing csvs:
  https://stackoverflow.com/questions/16804074/tool-to-diff-csv-files-at-the-field-level
  `git diff --color-words x.csv y.csv`
  `git diff --no-index --color-words x.csv y.csv`

### using multiple git accounts on the same machine
  https://kev.inburke.com/kevin/multiple-github-ssh-accounts/
  Setting up personal account:
    Set email as blank in global git config: ~/.gitconfig
      [user]
        name = John Hinrichs
        email = ''

    Create new ssh key and add to Github:
      ssh-keygen -t rsa -b 4096 -C "jmhinric@gmail.com" (enter new name for the rsa key- id_rsa_github_jmhinric)
      ssh-agent
      ssh-add -K ~/.ssh/id_rsa_github_jmhinric
      pbcopy < ~/.ssh/id_rsa_github_jmhinric.pub
      paste into Github ssh keys

    Update ~/.ssh/config:
      Host github.com-jmhinric
      IdentityFile ~/.ssh/id_rsa_github_jmhinric
      HostName github.com
      User jmhinric

    Clone a repo using the new ssh config:
      git clone git@github.com-jmhinric:jmhinric/things_i_learned.git

      In the directory of that repo, set your git config email:
        git config user.email "jmhinric@gmail.com"

      => Now you can make commits and push

### explicit repository excludes:
  add to `.git/info/exclude` then do `git update-index --assume-unchanged <file>`
  to undo this:  `git update-index --no-assume-unchanged <file>`
  https://help.github.com/articles/ignoring-files/#explicit-repository-excludes
  https://stackoverflow.com/questions/4308610/how-to-ignore-certain-files-in-git

### amend author/date info of latest commit
  git commit --amend --author="Author Name <email@address.com>"
  git commit --amend --date="Thu Sep 28 11:46:59 2017 -0500"

### move changes of latest commit to unstaged
  git reset HEAD~

### show diff of latest commit
  git show HEAD

### Instead of rebasing:
  from branch:  git fetch, git merge origin/master

### if conflict in db/structure.sql and your branch has migrations:
  git checkout --theirs db/structure.sql
  rake db:migrate
  git add db/structure.sql to be committed


### error in trying to push because your branch is stale:
  git fetch <branch_name>
  git rebase origin/<branch_name>
  -step through any conflicts

### force pull
  git fetch
  git reset --hard origin/your_branch

### diff with origin:
  git diff dev -- db/structure.sql

### git rerere
  "replays" a fix in a series of conflicts in a rebase

### Delete multiple git branches
git branch -D `git branch | grep -E 'FEAT'`

### git stash
  git stash
  git stash pop (applies latest and drops it from the list)
  git stash save "message"
  git stash apply stash@{2} (applies but doesn't drop from the list)
  git stash list
  git stash drop stash@{3}

### git push --force-with-lease
  https://robots.thoughtbot.com/git-push-force-with-lease

### auto-squashing
  https://robots.thoughtbot.com/autosquashing-git-commits





####################################################################################################
## Postgres
  rm /usr/local/var/postgres/postmaster.pid

### exporting and importing subset of data
  https://gist.github.com/ddrscott/8eed79573192d982486d7e43e3f01211

  export source data to file:
    psql anon -c "copy (SELECT * FROM assets WHERE color='red') to stdout csv header" > assets.csv

  create new database instance (without password):
    docker run --name postgres-9.6 -e POSTGRES_PASSWORD='' -d -p 65436:5432 postgres:9.6

  copy table screma:
    pg_dump --schema-only --table=assets anon > schema.sql

  might need to remove foreign keys:
    $EDITOR schema.sql

  create new schema:
    psql -h localhost -p 65436 -U postgres < schema.sql

  insert into destination DB:
    psql -h localhost -p 65436 -U postgres -c 'copy assets from stdin with (format csv, header)' < assets.csv

### Run command from command line
  psql development_db -c "select * from users limit 50"

### Save SQL output to .csv file
  psql development_db -c "copy(select * from users limit 50) to stdout csv header" > foo.csv

### Get size of entire database
  SELECT pg_size_pretty(pg_database_size('cmm_integration')) As fulldbsize;

### Prod dump restore
  pg_restore -c -F c -d local_db_name -v /Users/john.hinrichs/Downloads/db_dump_name.sql.gz

### Get environment config:
  puts ActiveRecord::Base.connection.instance_variable_get :@connection_parameters

### switching dbs
  https://gist.github.com/mswieboda/92b205153f79e7d62cad6bf205c34002

### To save a db dump:
  1. pg_dump development_db > ~/Downloads/local_db_20160516.sql

### To use a saved db dump:

  1. dropdb development_db
  2. createdb development_db
  3. psql development_db < ~/Downloads/local_db_20160516.sql

  alias restore='dropdb development_db && createdb development_db && psql development_db < `find ~/Downloads -name 'cmm_*' | fzf`'

  For a .sql.gz dump:
    - pg_restore -c -F c -d development_db -v ~/Downloads/prod_db_20160122.sql.gz
      * `-c` (clean - "Clean (drop) database objects before recreating them")
      * `-F c` (format custom - "The archive is in the custom format of pg_dump")
      * `-d` (database name)
      * `-v` (verbose)

### Dump from a remote environment:
  - pg_dump -h 10.0.0.100 -U username staging_env > /tmp/staging_env_20160511.sql
  - pg_dump -Fc -d staging_env -U username > /tmp/staging_env_20160511.sql.gz
  - scp staging:/tmp/staging_env_20160511.sql ~/Downloads/staging_env_20160511.sql
  - gunzip --stdout /tmp/development_db_201605311625.sql.gz | psql -U username integration_db

  From Scott Pierce:
  - ssh user@mike.dev.mysite.net '(gzip -c /tmp/prod_db_20160628.sql)' > /tmp/prod_db_20160628.sql.gz
  - ssh user@mike.dev.mysite.net '(gzip -c /tmp/prod_db_20160628.sql)' | ssh user@john.dev.mysite.net 'cat > /tmp/prod_db_20160628.sql.gz'
  - ssh user@mike.dev.mysite.net '(gzip -c /tmp/prod_db_20160628.sql)' | ssh user@john.dev.mysite.net 'gunzip | psql -U username integration_db'

  function dbsnap {
    out=~/Downloads/$1_`date +%Y%m%d%H%M`.sql.gz
    cmd="pg_dump $1 | gzip > $out"
    print $cmd
    eval $cmd
  }

  function pg_dump_branch {
    DB="${1:-cmm_development}"
    DEST=.pg_dump-`git rev-parse --abbrev-ref HEAD`.sql.gz
    echo "dumping $DB to $DEST"
    pg_dump $DB | gzip > $DEST
  }

  function pg_load_branch {
    DB="${1:-cmm_development}"
    SRC=.pg_dump-`git rev-parse --abbrev-ref HEAD`.sql.gz
    echo "loading $DB from $SRC"
    dropdb $DB && createdb $DB && gunzip -c $SRC | psql $DB
  }

### JOIN LATERAL
  SELECT
    plans.*
  FROM plans
  JOIN LATERAL (
    SELECT
    CONCAT(
      CASE
        WHEN plans.state IN ('approved', 'inactive')
          THEN CONCAT('Ver. ', plans.version, '-')
        WHEN plans.state IN ('revision')
          THEN CONCAT('Rev. ', plans.version, '-')
        ELSE
          ''
       END,
       plans.name,
      CASE
        WHEN plans.budget IS NOT NULL
          THEN CONCAT(' (', plans.budget, ')')
        ELSE
          ''
      END
    ) AS versioned_name
  ) AS versioned_names on true;

### Notes from Safari SQL course
  -- coalesce- default value
  SELECT id, lock_version, ordered_units, coalesce(deleted_at, '2015-5-1') as deleted_at from direct_addons
  WHERE ordered_units > 3000000
  ORDER BY ordered_units;

  -- single line comment
  SELECT id, lock_version, ordered_units, coalesce(deleted_at, '2015-5-1') as deleted_at from direct_addons
  --WHERE ordered_units > 3000000;

  -- multi-line comment
  SELECT id, lock_version, ordered_units, coalesce(deleted_at, '2015-5-1') as deleted_at from direct_addons
  /*WHERE ordered_units > 3000000
  ORDER BY ordered_units*/;


  -- LIKE, wildcard, single character wildcard
  SELECT first_name, last_name, email
  FROM auth_users
  --WHERE lower(last_name) LIKE '%ab%' -- last name contains 'ab'
  --WHERE last_name LIKE 'Ab%' -- last name starts with 'Ab'
  --WHERE lower(last_name) LIKE '%an' -- last name ends with 'an'
  --WHERE lower(last_name) LIKE '%an_a%' -- last name contains anything, 'an', single character, 'a', anything
  ORDER BY last_name, first_name;


  -- SIMILAR TO
  SELECT first_name, last_name, email
  FROM auth_users
  --WHERE lower(last_name) SIMILAR TO '%(a|b)%' -- last name contains 'a' or 'b'
  --WHERE lower(last_name) SIMILAR TO '[ab]%' -- last name starts with 'a' or 'b'
  --WHERE lower(last_name) SIMILAR TO '[a-d]%' -- last name starts with 'a', 'b', 'c' or 'd'
  ORDER BY last_name, first_name;

  -- IN/AND/OR
  SELECT first_name, last_name, email
  FROM auth_users
  WHERE last_name IN ('Block', 'Bode')
  AND first_name like '%s%'
  OR first_name = 'Michael'
  ORDER BY last_name, first_name;

  -- IN/AND/OR with logic groupings
  SELECT first_name, last_name, email
  FROM auth_users
  WHERE last_name IN ('Block', 'Bode')
  OR (first_name = 'Michael' AND last_name like '%o%')
  ORDER BY last_name, first_name;

  -- Calculation as new column
  SELECT id, ordered_units, ordered_units / 2 AS "New ordered units"
  FROM direct_addons
  LIMIT 100;

  -- String calculations as new columns
  SELECT
    first_name, last_name, email,
    strpos(email, '@') AS "Email @ index",
    substring(first_name, 2, 4) AS "First name middle",  -- extract 4 characters starting at letter 2 of the first name
    substring(email, 1, strpos(email, '@') - 1) AS "Email handle",
    left(first_name, 1) AS "First name initial", -- extract 1 character starting from the left of the first name
    right(last_name, 2) AS "Last name last 2 chars" -- extract 2 characters starting from the right of the last name
  FROM auth_users
  ORDER BY last_name, first_name;

  -- Date functions
  SELECT
    id,
    EXTRACT(MONTH from start_date) AS "Start Month",
    EXTRACT(DAY from start_date) AS "Start Day",
    EXTRACT(YEAR from start_date) AS "Start Year",
    start_date,
    EXTRACT(HOUR from created_at) AS "Created Hour",
    EXTRACT(MINUTE from created_at) AS "Created Minute",
    EXTRACT(SECOND from created_at) AS "Created Second",
    DATE_PART('dow', created_at) AS "Day of week integer",
    TO_CHAR(created_at, 'dy') AS "Day of week name abbreviation",
    created_at
  FROM direct_flights
  LIMIT 100;

  -- Calculations with dates
  SELECT
    id,
    start_date,
    created_at,
    (now() - start_date) AS "Age"
  FROM direct_flights
  LIMIT 100;


  -- Concatenating fields/ Date formatting
  SELECT
    first_name, last_name, email,
    first_name || ' ' || last_name AS "Full name",
    first_name|| ' was created at ' || to_char(created_at, 'MON DD YYYY HH12:MI:SS') AS "Was created"
  FROM auth_users
  ORDER BY last_name, first_name;


  -- Inner Joins
  SELECT a.name AS "Agency Name", /*cp.first_name, cp.last_name, c.email,*/ c.*, cp.*
  FROM catalog_contact_profiles cp
  INNER JOIN catalog_contacts c
  ON c.id = cp.contact_id
  INNER JOIN auth_agencies a
  ON a.id = cp.agency_id;

  -- Outer Joins
  SELECT *
  FROM direct_flights f
  --LEFT OUTER JOIN direct_tactics t
  RIGHT OUTER JOIN direct_tactics t
  --INNER JOIN direct_tactics t
  ON t.id = f.tactic_id
  ORDER BY f.tactic_id DESC
  LIMIT 1000;

  -- Cross (Cartesian) Joins
  SELECT *
  FROM direct_flights, direct_tactics;
  -- OR
  SELECT *
  FROM direct_flights
  CROSS JOIN direct_tactics;


  -- Group by
  SELECT a.name, count(c.id) AS "Total", min(c.created_at) AS "Earliest created"
  FROM catalog_contact_profiles cp
  INNER JOIN catalog_contacts c
  ON c.id = cp.contact_id
  INNER JOIN auth_agencies a
  ON a.id = cp.agency_id
  GROUP BY a.name
  ORDER BY "Total" DESC;


  -- Aggregate cost and number of rentals for all customers
  SELECT
    c.customer_id AS "Customer ID",
    c.first_name AS "First Name",
    c.last_name AS "Last Name",
    /*'$'|| */sum(f.rental_rate) AS "Total Cost",
    sum(c.customer_id) AS "Number of rentals"
  FROM customer c
  JOIN rental r
    ON r.customer_id = c.customer_id
  JOIN inventory i
    ON r.inventory_id = i.inventory_id
  JOIN film f
    ON f.film_id = i.film_id
  GROUP BY
    c.customer_id,
    c.first_name,
    c.last_name
  ORDER BY "Total Cost" DESC;

  -- Individual rentals for one customer
  SELECT
  c.customer_id AS "Customer ID",
  c.first_name AS "First Name",
  c.last_name AS "Last Name",
  r.rental_id,
  r.rental_date,
  f.title,
  f.rental_rate
  --sum(f.rental_rate) AS "Total Cost"
  FROM customer c
  JOIN rental r
    ON r.customer_id = c.customer_id
  JOIN inventory i
    ON r.inventory_id = i.inventory_id
  JOIN film f
    ON f.film_id = i.film_id
  WHERE c.customer_id = 1
  --GROUP BY c.customer_id
  ORDER BY f.title;

### 6/4/17 Creating and downloading database backups on heroku
https://devcenter.heroku.com/articles/heroku-postgres-backups
heroku pg:backups:capture --app evergreendataservices
heroku pg:backups:download b001

### 6/4/17 Restoring heroku backup to a local database
https://devcenter.heroku.com/articles/heroku-postgres-import-export
`pg_restore --verbose --clean --no-acl --no-owner -h localhost -U myuser -d mydb latest.dump`







####################################################################################################
## Docker

### setup
install docker-machine to get the 'Docker Quickstart Terminal' and follow steps in the guide:
  -https://docs.docker.com/v1.8/installation/mac/
    These commands from the guide should be successful:
    -'docker run hello-world'
    -'docker-machine create --driver virtualbox default'
    -'docker-machine ls'
    -'docker-machine env default'
    -'eval "$(docker-machine env default)"' (allows you to run docker commands in that shell)
    -'docker run hello-world'

clone Chris T's docker repo into a separate directory from centro-media-manager
  -http://stash.dev.sitescout.ad/users/chris.tenharmsel/repos/dev_docker/browse
    From the directory of Chris T's repo run:
    -'docker-compose up -d'

From the cmm directory:
  -make sure you've done 'eval "$(docker-machine env default)"'
  -'docker ps' should report the services

In the /configs folder of cmm, add the following two files (they are .gitignore'd automatically):
  development-override.rb and test-override.rb
  -Contents of both files are the following:
    ######
    dockerMachineIp = `docker-machine ip default`.strip
    elasticsearch.host = dockerMachineIp

    logging.logstash_host = dockerMachineIp
    logging.logstash_port = 9999

    message_queue.connection_string = "amqp://guest:guest@#{dockerMachineIp}:5672"

    redis.host = dockerMachineIp
    #####

### To run kibana:
  'docker-machine ip default'
  http://that-ip:5601
  Use default index names for kibana (the web app must have been run for that to work)
  Set auto refresh for the logs, which can be paused


### If an error after computer restart, you can remove the 'default' env and recreate it:
  docker-machine rm default
  docker-machine create --driver virtualbox default
  docker-machine env default
  eval "$(docker-machine env default)"
  docker ps
  docker-compose up -d (from the docker repo)


### Docker for Mac
https://docs.docker.com/docker-for-mac/docker-toolbox/

### if docker-compose up -d says, "no space left on device"
https://forums.docker.com/t/no-space-left-on-device-error/10894/5

ls -lah ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2
rm ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2


  #### Increase docker disk space in local git:
  https://gist.github.com/stefanfoulis/5bd226b25fa0d4baedc4803fc002829e






####################################################################################################
## Vim

  cb " change to beginning of word
  cw " change to end of word
  ciw " change inner word

  dw " delete word
  df-space " delete to next space
  dg " delete all under cursor
  gg + dG " delete all in file
  line # + G " goto line number
  gg " goto top of file
  G " goto bottom of file
  } " goto next paragraph
  { " goto previous paragraph

  CTRL-n " select multiple
    c " replace word

  vs " split vertical pane
  sp " split pane

  gg=G " reindent whole file

  cursor nav:
    \\w " gives selections for spots after cursor
    \\b " gives selections for spots before cursor
    ^ " beginning of line command mode
    $ " end of line command mode
    I " beginning of line insert mode
    A " end of line insert mode

  \n " toggle nerd tree

  d " delete
  c " change
  y " yank
  p " paste

  :m line # " move current line to after line #

  . " repeat prior command

  v " visual character
  V " visual line
  CTRL-v " visual block

  / " search within a file
  :map " map command ("ex map"- colon is ex commands):w

  i " inner
  a " around
  ciw " change inner word
  caw " change around word
  ci' " change inner single quote
  ci[ " change inner bracket

  * " forward search- word under cursor
  # " back search- word under cursor
  n " goto next found selection
  N " goto previous found selection

  coh " change- on/off highlight


### 12/11/15 Steve SanPietro - Vim

  https://s3.amazonaws.com/uploads.hipchat.com/14474/1168883/ASWh5aVYFwu2mTf/sanpietro_vim_presentation_notes.txt





####################################################################################################
## Ruby/Rails

### Run rails server or rails console with DATABASE_URL specified:
  DATABASE_URL=postgresql://postgres:password@localhost:65432/cmm_anon?pool=5 bin/rails c

### In method to binding.pry for return value but still return the original value
  .tap { |x| binding.pry } at end of query

### To see paths to source code of installed gems:
bundle list --paths

### pry step into:
n

### ruby standard output
$stderr

### rspec
  --format documentation to see spec file being run

### to locally run rails console for integration environment
DATABASE_URL=postgresql://cmm:ervDFl7qpdZ1VkmmO4zgQGJUs7HYWe@localhost:15432/integration_db be rails c
(in separate tab must be ssh'ed to the envt)

### Save json to a file
file = File.open('blah.json', 'w+')
file << '['
array.map(&:to_json).each_with_index { |k, i| file << k; file << "," unless i == (array.size - 1) };
file << ']'
file.close

### Use unpublished code from a gem
In the root Gemfile, specify a `path` to a separate directory that is the repo of the gem.
You can make changes to the gem there on a local branch.
After adding to the Gemfile, run `bundle`

gem 'some-great-gem', path: '../some-great-gem'





####################################################################################################
## Airbrake

### To log errors to Airbrake locally
in config/initializers/airbrake.rb:
  -add: `config.development_environments = []`
  -add `development` to the list of environments in the top of that file

  Airbrake.configure do |config|
    config.development_environments = []
    config.api_key = [{api key here}]
    config.logger = SemanticLogger[Airbrake]
  end






####################################################################################################
## Linux

### Notes from http://www.opsschool.org/en/latest/shells_101.html
  - A shell is the command-line interface between the user and the system.
  - Bash is known as the Bourne-again shell, written by Brian Fox, and is a play on the name of the previously named Bourne shell (/bin/sh) written by Steve Bourne.
  - By default, bash operates in emacs mode.
  - $PATH environment variable defines the set of directories that the shell can search to find a command.
  - Shell profiles are used to define a user’s environment.
    - Global profile (/etc/profile)
    - User profile (~/.bash_profile)
  - To determine the process ID (PID) of the current shell, first run `ps` to find the PID, then run `echo $$` to confirm the PID.
  - When a command is executed, it always has an exit status which defines whether or not the command was successful. For success, the exit status is 0.
    - `echo $?` to see the exit status of the previous command.
  - `cat ~/.bash_history` to see all stored commands.
  - A benefit of storing command history is that the commands can be easily recalled. To execute the last command, type `!!`.
  - To execute the nth command in history, run `!n` where n is the line number of the command as found in the output of `history`




### To list all of the shell’s environment variables:
  `env` or `printenv` command

### To change to sudo access as root user:
  sudo su -

### To find if all lines of a csv are uniq:
  cat ~/dups-test.csv | wc -l
    => 730
  cat ~/dups-test.csv | sort -u | wc -l
    => should be the same number
  * make sure the original csv has an empty last line, or the sort will insert one and be off by 1

### opendiff
  shows the diff of two files
  opendiff file1.csv file2.csv

### curl example
  curl -i -G \
  -d 'fields=name,pixel' \
  -d 'access_token=blah' \
  https://graph.facebook.com/v2.10/abc-123

### sed
  sed -i '' 's/SelectAdServerAccountDialog/DeliverySourceAccountsDialog/g' `ag -l --nocolor 'SelectAdServerAccountDialog' .`

### ssh tunnel
  ssh -L 5431:10.0.0.100:5432 user@env.mysite.net
  Then can execute local postgres commands
  psql -h localhost -d database_name -U user_name -c "select 1"

### 2/15/17 Steve SanPietro - ssh tunneling
  https://gist.github.com/sanp/e77be072c7f515a8051fbbe0e4e06e8c

### ssh config (Corey Burrows)
  Add to  ~/.ssh/config
    Host johnh
    Hostname johnh.dev.mysite.net
    User user1

  Then you can do:
    ssh johnh
    ssh johnh hostname
    scp johnh:/tmp/db.sql .

### Ted Price command line tools
  https://gist.github.com/pricees/b19dab5f584ab340a7b5

### watch/free
  free -m => shows free space (in megabytes?)
  watch "free -m" => updates the command every two seconds

### screen
  To execute a process in an ssh connection, be able to close the connection, then "reattach" later:
  Commands:
  screen
  (execute command e.g. bundle exec rails runner ...)
  Ctrl-a Ctrl-d (exits)
  screen -r (reconnects)

  To kill the screen once you're done with it:
  Ctrl-a k (It will ask if you really want to kill it)

  When doing `screen -r` and given this message:
  ```
  There are several suitable screens on:
  16694.pts-0.peterw  (Detached)
  32179.pts-0.peterw  (Detached)
  Type "screen [-d] -r [pid.]tty.host" to resume one of them.
  ```
  Execute:
  screen -r 16694

### cat/sort/diff/wc
  `cat unsorted_csv.csv | sort > sorted_csv.csv`
  `diff a1.csv a2.csv`
  `diff a1.csv a2.csv | wc -l`

### Disconnected ssh session
  To kill it:  <Enter>, then <Shift> + ~

### cron jobs
  `0 12 2 * * cd /some/path/blah && RAILS_ENV=development bundle exec rake some_task:execute` - 12:00 on second day of month
  `7 * * * * cd /some/path/blah && RAILS_ENV=development bundle exec rake some_task:execute`- at :07 every hour, every day
  `*/10 * * * * cd /some/path/blah && RAILS_ENV=development bundle exec rake some_task:execute`- every 10 minutes, every hour, every day

### Redirecting input and output
  Redirect output to a file: `date > file1.txt`

  Redirect input and output: `sort < file1.txt > file2.txt`

  Redirect output to input of another program with a pipe:
  `cat file1 file2 file3 | sort > /dev/file4.txt`

### & (background job)
  `&` after a command makes it a background job- shell not wait for the program to complete before giving the prompt again:
  `cat file1 file2 file3 | sort > /dev/file4.txt &`

#### PID of a background process can be found using the `$!` variable
  - `echo $!`




####################################################################################################
## Redis/Sidekiq
### Monitor redis
redis-cli monitor

### sidekiq consumer daemons
in project root:
watch -n 30 "ps axuw | grep bin/[c]onsumer || init_scripts/consumer_daemon.sh start"

### clear the redis queue
Sidekiq.redis { |r|  r.flushall }

### delete a sidekiq job
queue = Sidekiq::Queue.new("default")
queue.each do |job|
  job.klass # => 'MyWorker'
  job.args # => [1, 2, 3]
  job.delete if job.jid == 'abcdef1234567890'
end





####################################################################################################
## CSS
### right align:
<div className="Arrange">
  <div className="Arrange-sizeFill">
    Left aligned
  </div>
  <div className="Arrange-sizeFit">
    Right aligned
  </div>
</div>





####################################################################################################
## Homebrew

### postgres
  brew tap homebrew/services
  brew services list
  brew services stop postgresql
  psql -h localhost -p 5432 db_name

### licecap screen recording
  brew install licecap





####################################################################################################
## Pow
### no password needed for pow
sudo visudo
add the line:
%_developer ALL=(All) NOPASSWD: ALL





####################################################################################################
## Kibana

search text: no asterisk, put in quotes

"search/string" AND user_id:"341f9984-65f5-4d76-ae16-8b7f58f31049"
request_id:"a6b5c6aa-24da-46b1-9fe0-dbab8b3c2f78"

Chrome extension "Kibana Fixify" to fix autocomplete bug





####################################################################################################
## Capistrano
### deploy branch
  BRANCH=branch_name cap internal deploy





####################################################################################################
## Python
### post request
import requests
url = "https://example.com/api/users"
payload = {}
payload['email'] = 'blah@test3.com'
payload['first_name'] = 'Foo3'
payload['last_name'] = 'Bar3'
payload
r = requests.post(url, params=payload)
r.status_code # should be 201

# Deploy Rails application on AWS ElasticBeanstalk

Deploying a Rails app can be a somewhat daunting task to get set up right on new applications, even for seasoned Rails developers. While Capistrano has been a main player in Rails app deployments, it seems to have fallen somewhat out of favor as load balancers and computing servers have gained popularity.

Thankfully, AWS has created a tool for deploying and scaling Rails web application in their eco-system.

## Getting started

If you do not have an AWS account yet, go sign up.

#### After your set up your user

* Install CLI tools:

    ```
    brew update
    brew install awscli
    brew install awsebcli
    ```

* Config the `AWS CLI` [[Reference]](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html):

    ```
    aws configure
    ```

#### Lets initialize ElasticBeanstalk:

* Go to `AWS ElasticBeanstalk Service` console, create new application and add evironment for that application:

    ![awseb-create-app](https://raw.githubusercontent.com/tranquangvu/dev-notes/master/deploys-rails-to-awseb/assets/awseb-create-app.png)

    ![awseb-create-environment](https://raw.githubusercontent.com/tranquangvu/dev-notes/master/deploys-rails-to-awseb/assets/awseb-create-environment.png)

* Init `AWS EB` for your project:

    ```
    $ cd /path/to/project
    $ eb init

    # Input correct application information
    # Select a default region
    # ....
    # Select an application to use
    # ...
    # Select the default environment
    # ...
    # Do you wish to continue with CodeCommit? (y/N)
    # N
    ```

* Change default environment by:

    ```
    $ eb use environment-name
    ```

* SSH to AWS EB instances by:

    ```
    $ eb ssh
    ```

* Choose a correct platform for your environment:
    * To show current AWS EB environment's platform:

        ```
        $ eb status

        # Example output
        # Environment details for: demo
        #  Application name: demo
        #  Region: ap-southeast-1
        #  Deployed Version: app-af78-190121_144555
        #  Environment ID: e-apzg5kgizi
        #  Platform: arn:aws:elasticbeanstalk:ap-southeast-1::platform/Puma with Ruby 2.3 running on 64bit Amazon Linux/2.8.7
        #  Tier: WebServer-Standard-1.0
        #  CNAME: alex-world-prod.8dpew3vwes.ap-southeast-1.elasticbeanstalk.com
        #  Updated: 2019-01-21 06:53:20.204000+00:00
        #  Status: Ready
        #  Health: Green
        ```

    * Get available platforms from AWS EB:

        ```
        $ aws elasticbeanstalk list-available-solution-stacks
        ```

    * Update platform for your environment:

        ```
        $ aws elasticbeanstalk update-environment --solution-stack-name "64bit Amazon Linux 2018.03 v2.8.7 running Ruby 2.3 (Puma)" --environment-name "demo-production" --region "ap-southeast-1"
        ```

* Set environment variables: Set application's ENV by [AWS EB configuration interface](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environments-cfg-softwaresettings.html) or [using command line](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb3-setenv.html):

    ```
    $ eb setenv KEY=value
    ```

* Setup `Posgres database` for your rails application:
    * Add `Posgres database` with `AWS RDS` for your environment:
        1. Open the `Elastic Beanstalk` console
        2. Navigate to the management page for your environment
        3. Choose `Configuration`
        4. On the `Database` configuration card, choose `Modify`
        5. Choose a DB engine, and enter a user name and password
        6. Choose `Apply`

      ![aws-rds](https://raw.githubusercontent.com/tranquangvu/dev-notes/master/deploys-rails-to-awseb/assets/awseb-rds.png)

    * Add `config/database.yml` file (please make sure your `.gitignore` don't remove `datatabase.yml` when you commit in deploy branch): `RDS_DB_NAME, RDS_USERNAME, RDS_PASSWORD, RDS_HOSTNAME, RDS_PORT` environment variables are exported when you add `AWS RDS` to environment.

        ```
        default: &default
          adapter: postgresql
          encoding: unicode
          pool: 25
        production:
          <<: *default
          database: <%= ENV['RDS_DB_NAME'] %>
          username: <%= ENV['RDS_USERNAME'] %>
          password: <%= ENV['RDS_PASSWORD'] %>
          host: <%= ENV['RDS_HOSTNAME'] %>
          port: <%= ENV['RDS_PORT'] %>
        ```

* Add basic ebextensions config:
    * Create `.ebextensions/ruby.config` file with content:

        ```
        packages:
          yum:
            git: []
            postgresql96-devel: []
        commands:
          create_home_webapp_dir:
            command: |
              sudo mkdir -p /home/webapp
              sudo chown webapp:webapp /home/webapp
              sudo chmod 700 /home/webapp
            ignoreErrors: true
        ```

* Nice, we’re almost done. To deploy your code, just type:

    ```
    $ eb deploy
    ```


## The rest of the good stuff
* **Linux Swap:** When you get a few more gems in your app, especially those that insall native extensions, you may get failed deploys from running out of memory. Since, most apps can run on the free tier, and the only time memory is an issue is during a deploy, we will enable swap. Add a file `setup_swap.config` in the .ebextensions folder with the following content

    ```
    commands:
      01_setup:
        test: test ! -e /var/swapfile
        command: |
          dd if=/dev/zero of=/var/swapfile bs=1M count=2048
          chmod 600 /var/swapfile
          mkswap /var/swapfile
          swapon /var/swapfile
    ```

* **Auto add database.yml from database.yml.example:** Add this script to `.ebextensions/sidekiq.config` file
    ```
    files:
      "/opt/elasticbeanstalk/hooks/appdeploy/pre/03_add_database_yml.sh":
        mode: "000755"
        owner: root
        group: root
        content: |
          cd /var/app/ondeck/config
          sudo cp ./database.yml.sample ./database.yml
    ```


* **For Rails application use `sidekiq` and `redis`:**
    * Adding `redis` in aws with `AWS ElastiCache`:
        1. Open the `AWS ElastiCache` console
        2. Click on `Redis` from left menu
        3. Click create, and fill out the pretty straightforward form (picking t2.micro for free tier)
        4. Click Save and wait…
        5. Once created, clicking on the instance on the list will take you to a page with an endpoint listed in the table (only need to focus to `primary enpoint`)
        6. Set your redis url to :

            ```
            REDIS_URL=redis://your-redis-primary-endpoint/12

            # Your redis url should be like this
            # REDIS_URL=redis://your-name.iz6wli.0001.use1.cache.amazonaws.com:6379/12
            ```

    * Enable `sidekiq` auto start by adding `.ebextensions/sidekiq.config` file:

        ```
        commands:
          create_post_dir:
            command: "mkdir -p /opt/elasticbeanstalk/hooks/appdeploy/post"
            ignoreErrors: true
        files:
          "/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_sidekiq.sh":
            mode: "000755"
            owner: root
            group: root
            content: |
              #!/usr/bin/env bash
              . /opt/elasticbeanstalk/support/envvars

              EB_APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_deploy_dir)
              EB_APP_PID_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_pid_dir)
              EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config container -k app_user)
              EB_SCRIPT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k script_dir)
              EB_SUPPORT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k support_dir)

              . $EB_SUPPORT_DIR/envvars
              . $EB_SCRIPT_DIR/use-app-ruby.sh

              SIDEKIQ_PID=$EB_APP_PID_DIR/sidekiq.pid
              SIDEKIQ_CONFIG=$EB_APP_DEPLOY_DIR/config/sidekiq.yml
              SIDEKIQ_LOG=$EB_APP_DEPLOY_DIR/log/sidekiq.log

              cd $EB_APP_DEPLOY_DIR

              if [ -f $SIDEKIQ_PID ]
              then
                su -s /bin/bash -c "kill -TERM `cat $SIDEKIQ_PID`" $EB_APP_USER
                su -s /bin/bash -c "rm -rf $SIDEKIQ_PID" $EB_APP_USER
              fi

              . /opt/elasticbeanstalk/support/envvars.d/sysenv

              sleep 10

              su -s /bin/bash -c "bundle exec sidekiq \
                -e $RACK_ENV \
                -P $SIDEKIQ_PID \
                -C $SIDEKIQ_CONFIG \
                -L $SIDEKIQ_LOG \
                -d" $EB_APP_USER

          "/opt/elasticbeanstalk/hooks/appdeploy/pre/03_mute_sidekiq.sh":
            mode: "000755"
            owner: root
            group: root
            content: |
              #!/usr/bin/env bash
              . /opt/elasticbeanstalk/support/envvars

              EB_APP_USER=$(/opt/elasticbeanstalk/bin/get-config container -k app_user)
              EB_SCRIPT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k script_dir)
              EB_SUPPORT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k support_dir)

              . $EB_SUPPORT_DIR/envvars
              . $EB_SCRIPT_DIR/use-app-ruby.sh

              SIDEKIQ_PID=$EB_APP_PID_DIR/sidekiq.pid
              if [ -f $SIDEKIQ_PID ]
              then
                su -s /bin/bash -c "kill -USR1 `cat $SIDEKIQ_PID`" $EB_APP_USER
              fi
        ```

* **For Rails application crontab with `whenever` gem:**
    * Enable `whenever` auto update crontab by adding these content to `.ebextensions/ruby.config` file:

        ```
        files:
          "/opt/elasticbeanstalk/hooks/appdeploy/post/01_cronjob.sh":
            mode: "000755"
            owner: root
            group: root
            content: |
              #!/usr/bin/env bash
              set -xe

              EB_SCRIPT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k script_dir)
              EB_SUPPORT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k support_dir)
              EB_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_deploy_dir)

              . $EB_SUPPORT_DIR/envvars
              . $EB_SCRIPT_DIR/use-app-ruby.sh

              cd $EB_DEPLOY_DIR
              su -c "bundle exec whenever --set environment=production --update-crontab"
              su -c "crontab -l"
        ```

* **For Rails application use front-end assets from `npm`:**
    * Install npm assets before rails assets precompile by adding these content to `.ebextensions/ruby.config` file:

        ```
        files:
          "/opt/elasticbeanstalk/hooks/appdeploy/pre/04_asset_install.sh":
            mode: "000755"
            owner: root
            group: root
            content: |
              echo "installing nodejs 10.x version..."
              curl --silent --location https://rpm.nodesource.com/setup_10.x | sudo bash -
              sudo yum install -y nodejs
              echo "installing vendor asset via npm..."
              cd /var/app/ondeck
              sudo npm run your-script
        ```

* **For Rails 5 with Action Cable:**
    * Enable `websocket` from `nginx` by adding these content to `.ebextensions/ruby.config` file:

        ```
        files:
          "/opt/elasticbeanstalk/support/conf/webapp_healthd.conf":
            owner: root
            group: root
            mode: "000644"
            content: |
              upstream my_app {
                server unix:///var/run/puma/my_app.sock;
              }
              log_format healthd '$msec"$uri"'
                              '$status"$request_time"$upstream_response_time"'
                              '$http_x_forwarded_for';
              server {
                listen 80;
                server_name _ localhost; # need to listen to localhost for worker tier
                if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
                  set $year $1;
                  set $month $2;
                  set $day $3;
                  set $hour $4;
                }
                access_log  /var/log/nginx/access.log  main;
                access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd;
                location / {
                  proxy_pass http://my_app; # match the name of upstream directive which is defined above
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection "upgrade";
                  proxy_set_header Host $host;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
                location /assets {
                  alias /var/app/current/public/assets;
                  gzip_static on;
                  gzip on;
                  expires max;
                  add_header Cache-Control public;
                }
                location /public {
                  alias /var/app/current/public;
                  gzip_static on;
                  gzip on;
                  expires max;
                  add_header Cache-Control public;
                }
                location /cable {
                  proxy_pass http://my_app;
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade "websocket";
                  proxy_set_header Connection "Upgrade";
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
              }
          "/opt/elasticbeanstalk/support/conf/webapp.conf":
            owner: root
            group: root
            mode: "000644"
            content: |
              upstream my_app {
                server unix:///var/run/puma/my_app.sock;
              }
              server {
                listen 80;
                server_name _ localhost; # need to listen to localhost for worker tier
                location / {
                  proxy_pass http://my_app; # match the name of upstream directive which is defined above
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection "upgrade";
                  proxy_set_header Host $host;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
                location /assets {
                  alias /var/app/current/public/assets;
                  gzip_static on;
                  gzip on;
                  expires max;
                  add_header Cache-Control public;
                }
                location /public {
                  alias /var/app/current/public;
                  gzip_static on;
                  gzip on;
                  expires max;
                  add_header Cache-Control public;
                }
                location /cable {
                  proxy_pass http://my_app;
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade "websocket";
                  proxy_set_header Connection "Upgrade";
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
              }
        container_commands:
          99_restart_nginx:
            command: "service nginx restart || service nginx start"
        ```

## Tips

* **Start AWS EB rails console:**

    ```
    $ eb ssh
    $ sudo su
    $ cd /var/app/current
    $ RAILS_ENV=production bundle exec rails c
    ```

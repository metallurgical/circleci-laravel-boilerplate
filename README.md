# circleci-laravel-boilerplate
CircleCI CI/CD boilerplate for Laravel or PHP application. Modify based on your application!

# Config file
`.circleci/config.yml`

```
version: 2.1

jobs:
  # 1st Job -- Build docker container.
  app-build:
    docker:
      - image: circleci/php:7.4-node-browsers
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD
    working_directory: ~/laravel # directory where steps will run
    steps:
      - checkout
      - run:
          name: "Manage PHP dependencies, composer, etc"
          command: |
            sudo apt update
            sudo apt install -y libsqlite3-dev zlib1g-dev
            sudo docker-php-ext-install zip calendar exif pcntl
      - run:
          name: "Create Environment file"
          command: |
            cp .env.example .env
      - restore_cache:
          keys:
            - v1-composer-{{checksum "composer.lock"}}
            - v1-composer-
      - run:
          name: "Install dependencies"
          command: composer install -n --prefer-dist
      - save_cache:
          key: v1-composer-{{checksum "composer.lock"}}
          paths:
            - vendor
      - run:
          name: "Generate application key"
          command: php artisan key:generate
      - run:
          name: "Run unit/feature testing"
          command: php artisan test --testsuite=Feature
      - store_test_results:
          path: /tmp/test-reports
          
  app-test: # 2nd Job
    docker:
      - image: circleci/php:7.4-node-browsers # dockerImage:version 
        auth:
          username: $DOCKERHUB_USERNAME # define inside "credentials" context
          password: $DOCKERHUB_PASSWORD # define inside "credentials" context
    steps:
      - run: touch storage/database.sqlite
      - run: ./vendor/bin/phpunit tests/Unit # run can write using shortfont 
      
      - store_test_results: 
          path: /tmp/test-reports  
      
  app-deploy #3rd Job
    docker:
      - image: circleci/php:7.4-node-browsers # dockerImage:version
        auth:
          username: $DOCKERHUB_USERNAME # define inside "credentials" context
          password: $DOCKERHUB_PASSWORD # define inside "credentials" context
    steps:
      - run: curl <envoyer-endpoint> # trigger envoyer deploy
          
workflows:
  version: 2
  my-workflow:
    jobs:
      - app-build:
          context:
            - credentials # created inside circleCI dasboard
          filters:
            branches:
              ignore: 
                - /hotfix-.*/ # do not trigger pipeline if it was "hotfix" branch
            
      - app-test:
          context:
            - credentials # created inside circleCI dasboard
          requires:
            - app-build # require "app-build" finish only able run this job
            
      - app-deploy:
          context:
            - credentials # created inside circleCI dasboard
          requires:
            - app-build
            - app-test # require "app-test" finish only able run this job
          filters:
            branches:
              only: production # only trigger this job when branch checkout is "production"
            
 ```           

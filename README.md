# circleci-laravel-boilerplate
CircleCI CI/CD boilerplate for Laravel or PHP application. Too lazy to write the same thing when creating new project.

# Config file
`.circleci/config.yml`

```
version: 2.1

jobs:
  app-build: # 1st job
    docker:
      - image: dockerImage:version
        auth:
          username: $DOCKERHUB_USERNAME # was defined inside "credentials" context
          password: $DOCKERHUB_PASSWORD # was defined inside "credentials" context
    steps:
      - checkout # get the branch's code, if we push branch "a", this command will checkout "a" branch. Checkout is more like download the code into circleCI server and checkout the branch. Must be present in build job basically.
      
      - run: # run can be wrote like this by specifying name
          name: "Preparing environment dependencies"
          command: |
            sudo apt update
            sudo docker-php-ext-install zip
            
      - run:
          name: "Create Environment file for testing"
          command: |
            mv .env.testing .env       
          
      - restore_cache: # restore the cache file
          keys:
            - v1-composer-{{checksum composer.json}} # find for this key
            - v1-composer- # fallback to this key if above didnt found
            
      - run:
          name: "Install Dependencies"
          command: composer install -n --prefer-dist
          
      - run:
          name: "Generate Application key"
          command: php artisan key:generate     
            
      - save_cache: # recache the cache file
          key: v1-composer-{{checksum composer.lock}}
          paths: # on which path to store this file
            - /tmp
            
      - save_artifacts: # save build artifact later can be used to download from pipeline dashboard
          path: /tmp/artifacts
          destination: app-build
          
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

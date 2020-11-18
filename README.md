# circleci-laravel-boilerplate
CircleCI CI/CD boilerplate for Laravel or PHP application. Modify based on your application!

# Config file
`.circleci/config.yml`

```
version: 2.1
orbs:
  slack: circleci/slack@4.1.1

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
            sudo apt install -y libsqlite3-dev zlib1g-dev libmpc-dev
            sudo docker-php-ext-install zip calendar exif pcntl gmp bcmath
      - run:
          name: "Create Environment file"
          command: |
            mv .env.example .env
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
          command: |
            vendor/bin/phpunit --stop-on-failure --stop-on-error --debug
      - slack/notify:
          event: fail
          template: basic_fail_1
      - slack/notify:
          channel: 'C01EYA6QYSX'
          custom: |
            {
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "Build Successful! :tada:",
                    "emoji": true
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Project*:\\n$CIRCLE_PROJECT_REPONAME"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*When*:\\n$(date +'%m/%d/%Y %T')"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch*:\\n$CIRCLE_BRANCH"
                    }
                  ],
                  "accessory": {
                      "type": "image",
                      "image_url": "https://assets.brandfolder.com/otz5mn-bw4j2w-6jzqo8/original/circle-logo-badge-black.png",
                      "alt_text": "CircleCI logo"
                  }
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Job"
                      },
                      "url": "${CIRCLE_BUILD_URL}"
                    }
                  ]
                }
              ]
            }
          event: pass
      - store_test_results:
          path: test-reports


  # 2nd Job -- Deploy. Trigger envoyer link. Have to use envoyer for atomic and 0 downtime deployment.
  app-deploy:
    docker:
      - image: cimg/base:2020.01
        auth:
          username: $DOCKERHUB_USERNAME
          password: $DOCKERHUB_PASSWORD
    steps:
      - run:
          name: "Deploy to server"
          command: |
            sudo apt install curl
            echo "deploy call envoyer endpoint"
      - slack/notify:
          event: fail
          template: basic_fail_1
      - slack/notify:
          event: pass
          template: success_tagged_deploy_1

workflows:
  version: 2
  buil-test-deploy:
    jobs:
      - app-build:
          filters:
            branches:
              ignore:
                - /hotfix-.*/

      - app-deploy:
          context:
            - credentials
          requires:
            - app-build
          filters:
            branches:
              only:
                - production
            
 ```           

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
            mkdir phpunit
            vendor/bin/phpunit --stop-on-failure --stop-on-error --debug --log-junit phpunit/junit.xml tests
      - slack/notify:
          event: fail
          custom: |
            {
              "text": "",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "⚠️ Your job *${CIRCLE_JOB}* has failed. Build #${CIRCLE_BUILD_NUM} ⚠️"
                  },
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Commiter*:\\n${CIRCLE_USERNAME}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*SHA*:\\n${CIRCLE_SHA1}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch*:\n$CIRCLE_BRANCH"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Build Number*:\n$CIRCLE_BUILD_NUM"
                    }
                  ]
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
      - slack/notify:
          channel: 'C01EYA6QYSX'
          custom: |
            {
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": ":tada: Build Successful! :tada:",
                    "emoji": true
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Project*:\n$CIRCLE_PROJECT_REPONAME"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*When*:\n$(date +'%m/%d/%Y %T')"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch*:\n$CIRCLE_BRANCH"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Build Number*:\n$CIRCLE_BUILD_NUM"
                    }
                  ]
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
          path: phpunit
      - store_artifacts:
          path: phpunit


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
          custom: |
            {
              "text": "",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "⚠️ Your DEPLOYMENT job *${CIRCLE_JOB}* has failed. Build #${CIRCLE_BUILD_NUM} ⚠️"
                  },
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Commiter*:\\n${CIRCLE_USERNAME}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*SHA*:\\n${CIRCLE_SHA1}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch*:\n$CIRCLE_BRANCH"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Build Number*:\n$CIRCLE_BUILD_NUM"
                    }
                  ]
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
      - slack/notify:
          event: pass
          custom: |
            {
              "text": "",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": ":tada: DEPLOYMENT Successful! :tada:",
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
                      "text": "*Branch*:\n$CIRCLE_BRANCH"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Build Number*:\n$CIRCLE_BUILD_NUM"
                    }
                  ]
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
